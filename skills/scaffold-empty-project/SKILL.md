---
name: scaffold-empty-project
description: 当用户要求搭建、创建或初始化一个空项目、新项目、项目骨架或脚手架时必须使用本 skill（如「搭建一个空项目」「搭一个空项目」「新建一个项目」「创建一个最简项目」「初始化项目骨架」等）。按照固定技术栈生成最简可运行的全栈项目骨架：Go 后端（gin + ares 框架 + urfave/cli + gorm，cmd/server 入口 + internal/ 分层：bootstrap/controllers/services/dto/model/base/consts，外加 sql/static）+ Vue3 前端（Vite + TypeScript + Element Plus + Pinia + pnpm，位于 web/ 目录）+ 部署配置（deploy/ 下 Docker Compose + nginx + Dockerfile，多环境 yaml 配置，Makefile 构建）。只要用户想从零新建项目且未明确指定其他技术栈，就应使用本 skill，不要自行设计其他目录结构。
---

# 搭建空项目（最简全栈骨架）

按照固定的技术栈和分层结构，从零生成一个**最简、可运行**的全栈项目骨架。
所有文件内容都内嵌在本 skill 的模板中，**不需要也不应该去查找任何外部参考项目**。

## 技术栈

- 后端：Go + gin + ares 框架（github.com/inkbamboo/ares）+ urfave/cli + gorm + MySQL + Redis
- 前端：Vue3 + Vite + TypeScript + Element Plus + Pinia + Vue Router + Axios + pnpm
- 部署：Docker Compose（web/server/mysql/redis）+ nginx + Makefile + pack.sh 一键打包

## 第一步：确认参数

| 占位符 | 含义 | 默认规则 |
|---|---|---|
| `{{PROJECT_NAME}}` | 项目英文名（kebab-case），用作 module 名、目录名、容器名 | 用户指定 > 当前工作目录名 > 询问用户 |
| `{{PROJECT_TITLE}}` | 项目中文标题 | 用户指定 > `{{PROJECT_NAME}}` |
| `{{SERVER_PORT}}` | 后端端口 | 31001 |
| `{{DB_NAME}}` | MySQL 库名 | `{{PROJECT_NAME}}` 中的 `-` 替换为 `_` |

若用户只说「搭建一个空项目」而未给名字，且当前目录就是目标目录，则用目录名作为 `{{PROJECT_NAME}}`，无需反问。

## 第二步：生成目录结构

在目标目录下生成如下结构（只创建有实际文件的目录，前端其余目录按需创建；`web/public/` 与 `static/files/` 用 `.gitkeep` 占位；`docs/` 为 `make doc` 生成的 Swagger 产物，已 gitignore 不入库）：

```
{{PROJECT_NAME}}/
├── cmd/
│   └── server/
│       └── main.go                    # CLI 入口（默认启动 server，test 子命令，--conf/--env 参数）
├── config/
│   ├── config.local.yaml              # 本地开发配置
│   ├── config.stage.yaml              # 测试环境配置
│   └── config.prod.yaml               # 线上环境配置
├── config.docker.yaml                 # Docker 环境配置（构建时覆盖 config.prod.yaml）
├── deploy/
│   ├── docker-compose.yaml            # web/server/mysql/redis 编排
│   ├── nginx.conf                     # 前端 nginx（/api 反代后端）
│   ├── server.Dockerfile
│   ├── web.Dockerfile
│   └── start.sh                       # 本地一键 docker 启动
├── internal/
│   ├── base/
│   │   └── controller.go              # BaseController 基类
│   ├── bootstrap/
│   │   ├── bootstrap.go               # 配置/日志初始化
│   │   └── run.go                     # RunServer/RunTest
│   ├── consts/
│   │   ├── cache_keys.go              # Redis 缓存 key
│   │   ├── constant.go                # 常量
│   │   └── ecode/
│   │       └── ecode.go               # 统一错误码与 JSON 响应
│   ├── controllers/
│   │   ├── router.go                  # 路由注册（/ping + /api/v1 分组）
│   │   └── v1/
│   │       └── demo.controller.go     # 示例 Controller
│   ├── dto/
│   │   ├── base_dto.go                # 分页查询等通用 DTO
│   │   └── demo_dto.go                # 示例 DTO
│   ├── model/
│   │   └── base_model.go              # gorm 通用 BaseModel
│   ├── services/
│   │   ├── demo_service.go            # 示例 Service
│   │   └── demo_service_test.go       # 示例单元测试
│   └── utils/
│       └── util.go                    # 通用工具函数
├── sql/
│   └── init.sql                       # 建库建表 SQL
├── static/
│   └── files/                         # 上传文件目录（.gitkeep）
├── web/                               # Vue3 前端
│   ├── public/
│   ├── src/
│   │   ├── api/
│   │   │   └── demo-api.ts
│   │   ├── layouts/
│   │   │   └── index.vue
│   │   ├── plugins/
│   │   │   └── index.ts
│   │   ├── router/
│   │   │   └── index.ts
│   │   ├── store/
│   │   │   └── index.ts
│   │   ├── styles/
│   │   │   └── index.scss
│   │   ├── utils/
│   │   │   └── request.ts
│   │   ├── views/
│   │   │   ├── dashboard/
│   │   │   │   └── index.vue
│   │   │   └── login/
│   │   │       └── index.vue
│   │   ├── App.vue
│   │   ├── main.ts
│   │   └── settings.ts
│   ├── .env.development
│   ├── .env.production
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
├── go.mod
├── Makefile
├── pack.sh
├── README.md
└── .gitignore
```

## 第三步：按模板生成文件内容

读取以下模板文件，将所有 `{{PROJECT_NAME}}`、`{{PROJECT_TITLE}}`、`{{SERVER_PORT}}`、`{{DB_NAME}}` 占位符替换为实际值后写入目标文件：

- **`references/backend.md`** — Go 后端全部文件模板（go.mod、cmd、internal、consts、middlewares、sql、static）
- **`references/frontend.md`** — Vue3 前端全部文件模板（web/ 目录）
- **`references/deploy.md`** — 配置与部署模板（config yaml、deploy/、Makefile、pack.sh、README.md）

模板中 `go.mod` 的 go 版本按本机 `go version` 实际输出调整。

## 第四步：初始化依赖并验证

1. 后端：`go mod tidy` 后执行 `go build ./...` 和 `go test ./...`，确保编译与单元测试通过（网络问题用 `go env -w GOPROXY=https://goproxy.cn,direct`）。
2. 前端：`cd web && pnpm install`（registry 用 https://registry.npmmirror.com）。
3. 不要启动数据库等外部服务，编译/安装通过即可。

## 重要原则

- **保持最简**：只生成模板中的文件，不要自行添加登录、JWT、casbin、菜单权限等业务功能——骨架里只保留 `/ping` 健康检查和 `/api/v1/demo/hello` 示例接口。
- **内容只来自模板**：模板已包含全部所需内容，不需要读取任何参考项目或外部文件。
- **可扩展性**：分层结构（controller → service → model/dto，路由集中在 `router.go` 的 `registerAPI`）就是为后续扩展设计的。用户后续要加 JWT/casbin 时，根目录新建 `middlewares/` 包（`jwt_auth.go`、`auth_role.go`），在 `static/` 加 `rbac_model.conf`，路由处 `c.Group.Use(jwtFuc, roleFunc)`，Controller 内嵌 `base.BaseController`。
- 如果用户要求的技术栈与本项目不同（如 Python、React），不要套用本模板，按用户要求来。

## 完成后输出

向用户展示生成的目录树，并说明启动方式：

- 后端：`go run ./cmd/server --env=dev`（需本地 MySQL/Redis，或 `cd deploy && docker compose up -d mysql redis`）
- 前端：`cd web && pnpm dev`
- 构建/部署：`make build`、`make pack`、`cd deploy && docker compose up -d`
