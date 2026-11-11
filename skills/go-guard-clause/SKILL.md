---
name: go-guard-clause
description: Go 代码风格规范。编写或修改任何 Go 逻辑代码时必须使用本 skill：新增函数或方法、函数重构（卫语句、消除嵌套）、条件校验逻辑、消除冗余调用、缓存优先的数据获取、review Go 代码规范性。用户说「写简洁点」「优化写法」「规范一下」「能 return 的优先 return」时也必须触发。涉及变量遮蔽（局部遮蔽包级标识符、嵌套块内 err 重复定义）、命名规范（去阶段数字、魔数提常量）、计数循环改 range-int 时也必须触发。仅编写注释或接口文档、编译/部署/切 Go 版本、前端、SQL 索引、脚本类任务不适用。
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
				}
				return nil, ErrInactive
			}
			return nil, ErrNotFound
		}
		return nil, err
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

### 分支收尾即 return（三个递进层次）

分支是函数的完整终点时，按能到达的层次改写，不用 `else if` 串接：

1. 分支执行完就无事可做 → 分支末尾直接 `return`，下一个条件写**独立 if**。每个分支平顶展开，读者无需回溯前面所有分支的取反条件
2. 独立 if 是函数**最后一个**分支且「条件不成立时什么都不做」→ 条件**反向**为卫语句提前 `return`，主逻辑脱掉 if 外壳平铺到函数尾——卫语句始终在最顶上，主逻辑始终在零嵌套的最深处
3. 改写后若函数有命名返回值，收尾 `return` 可以裸写

```go
// 反例：else if 链式串接，读后面分支要回溯前面的否定条件
func applyDiscount(order *Order) {
	if order.VipLevel >= 3 {
		order.Price *= 0.8
	} else if order.Coupon != "" {
		order.Price *= 0.9
	}
}

// 正例：分支收尾 return + 尾段反转卫语句，主逻辑平铺到函数尾
func applyDiscount(order *Order) {
	if order.VipLevel >= 3 {
		order.Price *= 0.8
		return
	}
	if order.Coupon == "" {
		return
	}
	order.Price *= 0.9
}
```

适用边界：
- 层次 1 的前提是分支体为函数尾部唯一要做的事。若分支后还有**公共收尾逻辑**（如统一落库），保持 if/else 互斥结构、收尾写一份，不要在每个分支里重复
- 层次 2 仅当反向条件有清晰语义时使用（如「范围过小，无法映射」），并在卫语句上方写一行注释说明不成立时保持原值/跳过的原因——反向条件往往比正向难读，注释补上这一层

### 二选一/优先级取值用 if/else 链，不用 tagless switch

无 tag 的 `switch { case cond: }` 只在 3 个以上互斥分支且 case 体为单行赋值/返回时使用；case 体含多行逻辑（如解密+判空）时 switch 割裂阅读。先写「全空/非法」卫语句，再按优先级取值。

```go
// 反例：两分支二选一被 tagless switch 割裂
switch {
case req.Token != "":
	var payload *Payload
	if payload, err = decodeToken(req.Token); err != nil {
		return
	}
	filter = payload.Filter
default:
	filter = req.Filter
}

// 正例：全空判断前置为卫语句，if/else 按优先级取值
if req.Token == "" && req.Filter == "" {
	return "", ErrParam
}
if req.Token != "" {
	var payload *Payload
	if payload, err = decodeToken(req.Token); err != nil {
		return
	}
	filter = payload.Filter
} else {
	filter = req.Filter
}
```

### 已定义的变量不再用 `:=` 重复声明

函数有命名返回值 `err`（或 `err` 已在前文声明）时，`ok, err := f()` 虽因含新变量在语法上合法，但读起来像把 `err` 重新定义了一遍；更危险的是该语句一旦被挪进嵌套作用域（if/for 块内）就变成真遮蔽，defer 中的 `err` 日志会永远拿到 nil。统一写法：新变量 `var` 预声明，赋值放进 if 初始化语句；若赋值结果需供 if 块之后的代码使用，则 `var` 预声明 + 单独一行 `=` 赋值。

```go
// 反例：err 是命名返回值（签名中已定义），:= 造成「重复定义」观感
func BuildReport(...) (report *Report, err error) {
	conn, err := dialPrimary(...)
	if err != nil {
		return fromCache(...)
	}
	...
}

// 正例：新变量 var 预声明，赋值放进 if 初始化语句
func BuildReport(...) (report *Report, err error) {
	var conn *Conn
	if conn, err = dialPrimary(...); err != nil || conn == nil {
		return fromCache(...)
	}
	...
}
```

注意三点：
- if 初始化语句里必须用 `=`（不是 `:=`）——`:=` 会在 if 块内新建作用域变量、遮蔽外层/命名返回值
- `err` 是**首次出现**的新变量时 `:=` 是正确且首选的（如正例 `FindUser` 中的 `user, err := db.GetUser(...)`）；只有变量已存在（命名返回值、前文声明过）才要求预声明 + `=`
- 判断「err 是否已定义」**只看当前函数作用域**：命名返回值、前文 `var`/`:=` 声明。`if err := f()` 里的 err 属于 if 块的独立作用域，**不算**已定义，不影响后续再次 `err :=`

### 嵌套作用域内禁止 `err :=` 遮蔽外层已声明的 err

外层函数作用域已有 `err`（命名返回值或前文声明）时，**任何嵌套块**（if 初始化语句、if/for 块体、循环体内）中出现 `err :=` 都是真遮蔽——虽然语法合法，但 defer 日志、外层后续检查拿到的仍是旧值。统一改法：if 初始化语句里 `:=` 换 `=`；循环体内需要新变量时循环外 `var` 预声明。

```go
// 反例 1：if-init 内 := 遮蔽外层 err（ref, err := 已在前文声明）
if err := utils.ReadJSON(path, &cfg); err != nil {
	return fmt.Errorf("读取配置失败: %w", err)
}

// 正例 1：复用外层 err，if-init 用 =
if err = utils.ReadJSON(path, &cfg); err != nil {
	return fmt.Errorf("读取配置失败: %w", err)
}

// 反例 2：循环体内 extra, err := 每次迭代都遮蔽外层 err
for _, f := range files {
	extra, err := utils.ReadCSV(f)
	if err != nil {
		return nil, err
	}
	...
}

// 正例 2：新变量循环外 var 预声明，循环内用 = 复用外层 err
var extra *Table
for _, f := range files {
	extra, err = utils.ReadCSV(f)
	if err != nil {
		return nil, err
	}
	...
}
```

边界：
- 与上节第三点对照：`if err := f()` 的 err 是否构成遮蔽，**看外层函数作用域是否已有同名 err**。没有则 `:=` 正常；有则必须 `=`。函数内存在多处 if-init 检查时，建议函数开头一次声明 `err`，后续全部 `=`，全文只用一个 err 变量
- AST 遮蔽检测可用 go/ast 自写检查器（walk Block/if/for/range/switch 作用域栈）批量扫包级目录；`_` 空标识符的多重出现不构成遮蔽，可忽略

### 混合形态：`已有参数, 新err := f(...)`

同作用域下 `参数, err := f(参数, ...)` 在语法上会复用参数（重赋值）并声明新 err，Go 规范允许。但**统一不这么写**：左侧混有已有变量时 `:=` 读起来像把参数重新定义了一遍，观感歧义大于紧凑收益。统一改写为 `var` 预声明 + `=`：

```go
// 反例：items 混在 := 左侧，观感像重复定义；且赋值与 err 检查分离两行
items, err := filterInvalid(items, cond)
if err != nil {
    return nil
}

// 正例：:= 只留给「左侧全是新变量」；已有变量用 = 且赋值放进 if 初始化语句，
// 与错误检查合并为一行
var err error
if items, err = filterInvalid(items, cond); err != nil {
    return nil
}
```

配套规则：
- **`:=` 左侧必须全是新变量**——这是唯一允许 `:=` 的形态（`user, err := db.GetUser(...)` 两边都是新的）
- 调用后**原值已成死值**（不再被读取）→ 重赋值参数本身没问题（类似 `s = strings.TrimSpace(s)`），只是赋值形式用 `=`；若原值与新值要在**同一作用域共存** → 新值必须起独立名字
- 该形态严禁挪进 if/for 等嵌套块：嵌套块内 `:=` 会同时遮蔽参数和新建 err，外层/defer 拿到的永远是旧值

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

同一 handler 被注册到多个端路由（如小程序 `/v1/xxx` 与 App `/app/v1/xxx`，靠路由注册时传入的 providerType 区分）时，**各端不同的参数规则与归属校验属于业务规则**，且被多个 handler 复用——下沉到业务层提供共用解析方法，入口层只负责取值并转换为标识传下去。「不做前置检查」针对的是单一入口的登录/来源检查，不要混淆。

- 入口层取值一次：登录用户、providerType，把 providerType 转换成 `isApp bool` 这类**无框架语义的标识**再传
- 业务层解析方法签名收基础类型（`req *In, isApp bool, playerId int64`），不收会话/凭证结构体、不感知路由与中间件
- 端差异规则写进函数头注释（哪个端允许哪些参数、是否校验登录/归属），校验用卫语句前置

```go
// 业务层：端差异规则集中在此，多个 handler 复用
// app 端（isApp）：token 与 target_id 均可传，须已登录且为资源归属人本人；
// 小程序端：必须传 token，不校验登录与归属。
func (s *Service) ResolveTarget(req *In, isApp bool, playerId int64) (targetId string, err error) {
	if !isApp && req.Token == "" {
		return "", ErrParam
	}
	if isApp && playerId <= 0 {
		return "", ErrLogin
	}
	// ... 解析 token / target_id，卫语句排除非法值 ...
	if isApp {
		if err = s.checkOwner(targetId, playerId); err != nil {
			return
		}
	}
	return targetId, nil
}

// 入口层：仅取值与组装调用，不抽 helper
func (h *Handler) GetResource(ctx Ctx) error {
	req := &In{}
	if err := ctx.Bind(req); err != nil {
		return ErrParam
	}
	var playerId int64
	if user := h.BaseUser(ctx); user != nil {
		playerId = user.ID
	}
	targetId, err := h.svc.ResolveTarget(req, h.isApp(), playerId)
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
- 函数头注释写明规则（不同入参/状态下的行为差异），如 `FindUser` 的头注释
- 关键决策处加行内注释说明取舍（如「结果中已含归属人ID，直接比对无需查库」「兜底查库」）

## 七、循环与命名规范

### 计数循环用 range-int（Go 1.22+）

纯计数循环 `for i := 0; i < n; i++` 统一写成 `for i := range n`。遍历切片/映射用 `for i := range s` 或 `for k, v := range m`；仅索引时不需要 `_` 占位值。

```go
// 反例：经典三段式计数循环
for i := 0; i < len(rows); i++ {
	rows[i] = ...
}
for i := 0; i < n; i++ {
	vals[i] = f(i)
}

// 正例：range-int 直接写
for i := range rows {
	rows[i] = ...
}
for i := range n {
	vals[i] = f(i)
}
```

### 局部变量不得遮蔽包级标识符

包级函数/变量用了通用名（`sum`、`max`、`count` 等）后，函数体内 `sum := 0.0` 这类局部声明会静默遮蔽它——本想累加却定义了一个同名新变量，编译器不报错。两种修法按场景选：

```go
// 反例：包级有 func sum(xs []float64) float64，函数内局部 sum 把它遮蔽了
func weightedMean(xs []float64) float64 {
	sum := 0.0        // 遮蔽包级 sum 函数
	for _, x := range xs {
		sum += x
	}
	return sum / float64(len(xs))
}

// 正例 A（首选）：包级通用名太宽泛，改名让语义更窄更准
func sumOf(xs []float64) float64 { ... }  // 包级改名 sumOf

// 正例 B：局部变量换有语义的名字
func weightedMean(xs []float64) float64 {
	total := 0.0      // 局部换名，不与包级冲突
	for _, x := range xs {
		total += x
	}
	return total / float64(len(xs))
}
```

选择顺序：包级标识符是通用名（sum/max/min/count/find）→ 优先改包级名（`sumOf`/`maxOf`/`findFirst`），通用名本就不该占据包级命名空间；包级名语义明确且被多处引用 → 改局部变量名。检测方式：对包内每个 `:=`/`var` 声明的标识符，比对包级标识符表。

### 命名不带阶段/序号数字，魔数提常量

- 文件名与标识符不用 `stage1`、`output23`、`calc2` 这类阶段序号命名：数字不传达语义，读者无从知道 1 和 2 的差别。按**职责**命名——`stage1.go` → `single.go`（单灾评价）、`output23.go` 与 `output.go` 职责相近则直接**合并**为一个文件
- 函数体内的字面量魔数（如网格面积 `0.0025`）提取为有语义的包级常量 `gridAreaKm2`，同一魔数多处出现时尤其必须收敛为单一常量定义

## 八、修改流程

1. **先调研再动手**：读入参结构、路由注册方式（同一 handler 可能被多条路由复用）、相关 service
2. **改公共方法前先搜所有调用点**，确认影响面后再改签名
3. **优先复用项目现有模式**（中间件、缓存 key、错误码惯例），不发明新轮子
4. 改完编译验证 + `gofmt -l` 检查格式
