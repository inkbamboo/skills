---
name: go-guard-clause
description: Go 代码风格规范。编写或修改任何 Go 逻辑代码时必须使用本 skill：新增函数或方法、函数重构（卫语句、消除嵌套）、条件校验逻辑、消除冗余调用、缓存优先的数据获取、review Go 代码规范性。用户说「写简洁点」「优化写法」「规范一下」「能 return 的优先 return」时也必须触发。仅编写注释或接口文档、编译/部署/切 Go 版本、前端、SQL 索引、脚本类任务不适用。
---

# Go 代码风格规范

核心思想：**简洁、不冗余、职责清晰、缓存优先**。
适用于典型的 Go 后端技术栈（Web 框架 + ORM + Redis）。
写代码前先通读本规范，写完后用规范自查一遍。

## 一、函数结构：卫语句（Guard Clause）风格

所有失败/排除条件**前置为卫语句**，能 return 的优先 return。主成功路径零嵌套、顺序执行到函数尾部。禁止用 `if 正条件 { 深层嵌套 }` 包裹主逻辑。

**反例（嵌套式）：**
```go
func FindUser(req *QueryIn) (*User, error) {
	if req.ID != 0 {
		user, err := db.GetUser(req.ID)
		if err == nil {
			if user != nil {
				if user.Active {
					return user, nil
				} else {
					return nil, ErrInactive
				}
			} else {
				return nil, ErrNotFound
			}
		} else {
			return nil, err
		}
	}
	return nil, ErrParam
}
```

**正例（卫语句式）：**
```go
// FindUser 按 ID 查找状态正常的用户。
// ID 为空返回参数错误；不存在返回未找到；已停用返回不可用。
func FindUser(req *QueryIn) (*User, error) {
	if req.ID == 0 {
		return nil, ErrParam
	}
	user, err := db.GetUser(req.ID)
	if err != nil {
		return nil, err
	}
	if user == nil {
		return nil, ErrNotFound
	}
	if !user.Active {
		return nil, ErrInactive
	}
	return user, nil
}
```

要点：
- 先排除空值/非法参数，再排除中间步骤的失败，最后主逻辑平铺到函数尾
- 重复使用的布尔条件提取为语义化局部变量（`isVip := user.Level >= VipLevel`），只计算一次、多处复用
- 易错操作（解码、解析、类型断言）用独立局部变量接收错误（如 `decodeErr`），避免污染命名返回值
- 错误路径显式 `return nil, errXxx`；仅函数最后一步调用可借命名返回值裸 `return`
- 同一函数内多个独立分支各自提前返回，不要为「共享一个返回点」而深嵌套
- **二选一/优先级取值用 if/else 链，不用无 tag 的 `switch { case cond: }`**。tagless switch 只在 3 个以上互斥分支且 case 体为单行赋值/返回时使用；case 体含多行逻辑（如解密+判空）时 switch 反而割裂阅读。先写「全空/非法」卫语句，再按优先级取值

```go
// 反例：tagless switch 割裂了两分支的简单二选一
switch {
case req.ShareId != "":
	var shareInfo *ShareInfo
	if shareInfo, err = decrypt(req.ShareId); err != nil {
		return
	}
	boxIdx = shareInfo.BoxIdx
case req.BoxIdx != "":
	boxIdx = req.BoxIdx
default:
	return "", ErrParam
}

// 正例：全空判断前置为卫语句，if/else 按优先级取值
if req.ShareId == "" && req.BoxIdx == "" {
	return "", ErrParam
}
if req.ShareId != "" {
	var shareInfo *ShareInfo
	if shareInfo, err = decrypt(req.ShareId); err != nil {
		return
	}
	boxIdx = shareInfo.BoxIdx
} else {
	boxIdx = req.BoxIdx
}
```

## 二、消除冗余

- **同一值只获取一次**：方法调用、配置读取、上下文取值的结果存局部变量复用，不重复调用
- **外层已检查的，内层不重复**：入口层做了的前置检查，下游层不必再查（下游的兜底校验能保证安全性即可）
- **不写只用一次的 helper 方法**：单处使用的检查直接内联，不为「看起来整洁」抽方法
- **同一数据不重复提取**：从上下文/结构体取值一次，需要的字段一并取出

```go
// 反例：重复调用与重复提取
if s.GetConfig().Mode == ModeProd {
	s.Audit(s.GetConfig().Mode, ctx.UserID())
}

// 正例
mode := s.GetConfig()
userId := ctx.UserID()
if mode == ModeProd {
	s.Audit(mode, userId)
}
```

## 三、入口层 / 业务层职责分层

| 层 | 职责 |
|---|---|
| 入口层（handler/controller） | 参数绑定、前置检查（登录、来源等，内联不抽方法）、组装调用、返回响应 |
| 业务层（service） | 业务规则与条件校验，**不做前置检查** |

- 判断请求特征（来源、端类型等）用**路由/框架注册时确定的标识**，与登录状态无关、不依赖凭证内容里的字段
- 业务层方法传**基础类型**（`userId int64` 而非 `*UserClaims`），不传会话/凭证结构体，保持业务层与框架解耦

```go
// 入口层：前置检查内联，取值一次
func (h *Handler) GetDetail(ctx Ctx) error {
	req := &DetailIn{}
	if err := ctx.Bind(req); err != nil {
		return ErrParam
	}
	userId := ctx.UserID()
	if h.NeedLogin() && userId == 0 {
		return ErrLogin
	}
	detail, err := h.svc.GetDetail(req, userId)
	if err != nil {
		return err
	}
	return ctx.OK(detail)
}
```

### 例外：多端复用同一 handler 时，端差异规则下沉业务层

同一 handler 被注册到多个端路由（如小程序 `/v1/xxx` 与 App `/app/v1/xxx`，靠路由注册时传入的 providerType 区分）时，**各端不同的参数规则与归属校验属于业务规则**，且被多个 handler 复用——此时应下沉到业务层提供共用解析方法，入口层只负责取值并转换为标识传下去。「不做前置检查」针对的是单一入口的登录/来源检查，不要混淆。

规则：
- 入口层取值一次：登录用户、providerType（路由注册时确定，与登录状态无关），把 providerType 转换成 `isApp bool` 这类**无框架语义的标识**再传
- 业务层解析方法签名收基础类型（`req *In, isApp bool, playerId int64`），不收会话/凭证结构体、不感知路由与中间件
- 端差异规则写进函数头注释（哪个端允许哪些参数、是否校验登录/归属），校验用卫语句前置

```go
// 业务层：端差异规则集中在此，多个 handler 复用
// app 端（isApp）：share_id 与 box_idx 均可传，须已登录且为资源归属人本人；
// 小程序端：必须传 share_id，不校验登录与归属。
func (s *Service) ResolveBoxIdx(req *In, isApp bool, playerId int64) (boxIdx string, err error) {
	if !isApp && req.ShareId == "" {
		return "", ErrParam
	}
	if isApp && playerId <= 0 {
		return "", ErrLogin
	}
	// ... 解析 share_id / box_idx，卫语句排除非法值 ...
	if isApp {
		if err = s.checkOwner(boxIdx, playerId); err != nil {
			return
		}
	}
	return boxIdx, nil
}

// 入口层：仅取值与组装调用，不抽 helper
func (h *Handler) GetRoom(ctx Ctx) error {
	req := &In{}
	if err := ctx.Bind(req); err != nil {
		return ErrParam
	}
	var playerId int64
	if user := h.BaseUser(ctx); user != nil {
		playerId = user.ID
	}
	boxIdx, err := h.svc.ResolveBoxIdx(req, h.isApp(), playerId)
	if err != nil {
		return err
	}
	// ... 主逻辑 ...
}
```

## 四、性能优先级：缓存 > 数据库

- **数据里已有的字段不查库**：入参、解码结果、上游返回中已包含的字段直接使用，不要再查表
- **查库前先找现有缓存**：先看项目里同类数据怎么取（既有缓存 key 的值往往就是要查的字段），复用现有 key 和惯例，不发明新缓存
- **多级获取模式**：业务缓存 → 通用信息缓存 → 数据库兜底（兜底用只读从库）
- 拆成两个方法：**校验方法**（比对后返回错误）+ **纯获取方法**（内部处理多级降级）

```go
// checkOwner 校验资源归属，比对失败返回权限错误
func (s *Service) checkOwner(resKey string, userId int64) error {
	ownerId := s.getOwnerId(resKey)
	if ownerId == 0 {
		return ErrNotFound
	}
	if ownerId != userId {
		return ErrPermission
	}
	return nil
}

// getOwnerId 获取归属人：业务缓存 -> 信息缓存 -> 数据库兜底
func (s *Service) getOwnerId(resKey string) (ownerId int64) {
	if ownerId, _ = redisClient.Get(ctx, ttlKey(resKey)).Int64(); ownerId > 0 {
		return
	}
	if cache := getCache[ResMsg](msgKey(resKey)); cache != nil {
		return cache.OwnerId
	}
	// 兜底查库（只读从库）
	var res Resource
	if err := readDB.Where("res_key = ?", resKey).First(&res).Error; err == nil {
		return res.OwnerId
	}
	return
}
```

## 五、错误处理

| 场景 | 用法 |
|---|---|
| 参数错误 / 资源不存在 | 参数错误类型（如 `ErrParam`） |
| 条件不满足（无权限等） | 权限错误类型（如 `ErrPermission`） |
| 面向用户的提示 | 带消息的错误（如 `errParamWithMessage("提示文案")`） |

需要客户端识别并引导操作的场景（如未登录），返回统一业务 JSON（`HTTP 200 + 业务错误码`），不要在中间件直接 401 拒绝导致客户端解析不到统一格式。

## 六、注释规范

- 中文注释，解释**为什么**而非复述代码
- 函数头注释写明规则（不同入参/状态下的行为差异），如正例中 `FindUser` 的头注释
- 关键决策处加行内注释说明取舍（如「结果中已含归属人ID，直接比对无需查库」「兜底查库」）

## 七、修改流程

1. **先调研再动手**：读入参结构、路由注册方式（同一 handler 可能被多条路由复用）、相关 service
2. **改公共方法前先搜所有调用点**，确认影响面后再改签名
3. **优先复用项目现有模式**（中间件、缓存 key、错误码惯例），不发明新轮子
4. 改完编译验证 + `gofmt -l` 检查格式
