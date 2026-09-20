# 配置与部署模板

占位符 `{{PROJECT_NAME}}`、`{{PROJECT_TITLE}}`、`{{SERVER_PORT}}`、`{{DB_NAME}}` 替换后写入对应路径。

## config/config.local.yaml

```yaml
domain: "0.0.0.0:{{SERVER_PORT}}"
autoMigrate: false
debug: true
maxUploadSize: 52428800
jwt:
  secret: "CHANGE_ME_TO_A_RANDOM_STRING"
databases:
  - "alias": "master"
    "dialect": "mysql"
    "host": "127.0.0.1"
    "port": 3306
    "dbName": "{{DB_NAME}}"
    "username": "root"
    "password": "123456"
    "maxIdleConns": 25
    "maxOpenConns": 25
caches:
  - alias: "default"
    section: ""
    adapter: "redis"
    host: "127.0.0.1"
    port: 6379
    password: "123456"
    db: 0
```

## config/config.stage.yaml

内容同 `config.local.yaml`，`debug: false`，host 指向测试环境地址。

## config/config.prod.yaml

内容同 `config.local.yaml`，`debug: false`，host 指向线上环境地址。

## config.docker.yaml（项目根目录）

Docker 环境专用，构建镜像时覆盖为 `config/config.prod.yaml`。host 使用 compose 服务名：

```yaml
domain: "0.0.0.0:{{SERVER_PORT}}"
autoMigrate: false
debug: false
maxUploadSize: 52428800
jwt:
  secret: "CHANGE_ME_TO_A_RANDOM_STRING"
databases:
  - "alias": "master"
    "dialect": "mysql"
    "host": "mysql"
    "port": 3306
    "dbName": "{{DB_NAME}}"
    "username": "root"
    "password": "123456"
    "maxIdleConns": 25
    "maxOpenConns": 25
caches:
  - alias: "default"
    section: ""
    adapter: "redis"
    host: "redis"
    port: 6379
    password: "123456"
    db: 0
```

## deploy/docker-compose.yaml

```yaml
# ============================================================
# Docker Compose 发版编排（预构建，上传即用）
# 服务器无需 Node/Go/pnpm，只需 Docker
# ============================================================

networks:
  {{PROJECT_NAME}}-net:
    driver: bridge

services:
  web:
    build:
      context: .
      dockerfile: web.Dockerfile
    image: {{PROJECT_NAME}}-web:latest
    container_name: {{PROJECT_NAME}}-web
    restart: always
    mem_limit: 256m
    ports:
      - '80:80'
    depends_on:
      - server
    networks:
      - {{PROJECT_NAME}}-net

  server:
    build:
      context: .
      dockerfile: server.Dockerfile
    image: {{PROJECT_NAME}}-server:latest
    container_name: {{PROJECT_NAME}}-server
    restart: always
    mem_limit: 512m
    ports:
      - '{{SERVER_PORT}}:{{SERVER_PORT}}'
    volumes:
      - ../serverFiles:/opt/{{PROJECT_NAME}}/static/files
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - {{PROJECT_NAME}}-net

  mysql:
    image: docker.1ms.run/library/mysql:8.0.20
    container_name: {{PROJECT_NAME}}-mysql
    restart: always
    mem_limit: 1024m
    privileged: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_DATABASE: "{{DB_NAME}}"
    command:
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_general_ci
      - --explicit_defaults_for_timestamp=true
      - --lower_case_table_names=1
      - --sql_mode=ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-u", "root", "-p123456"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - ../dbdata/mysql:/var/lib/mysql
    networks:
      - {{PROJECT_NAME}}-net

  redis:
    image: docker.1ms.run/library/redis:7-alpine
    container_name: {{PROJECT_NAME}}-redis
    restart: always
    mem_limit: 256m
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass 123456
    healthcheck:
      test: ["CMD-SHELL", "redis-cli --no-auth-warning -a 123456 ping | grep -q PONG"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - ../dbdata/redis:/data
    networks:
      - {{PROJECT_NAME}}-net
```

## deploy/nginx.conf

```nginx
server {
    listen       80;
    server_name  localhost;

    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 1;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png application/vnd.ms-fontobject font/ttf font/opentype font/x-woff image/svg+xml;
    gzip_vary on;
    gzip_disable "MSIE [1-6]\.";

    location ^~ /api/ {
        proxy_pass http://server:{{SERVER_PORT}};
        proxy_connect_timeout 30;
        proxy_send_timeout 30;
        proxy_read_timeout 30;
        client_max_body_size 1000m;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    root /usr/share/nginx/html;
    location / {
        try_files $uri $uri/ @router;
        client_max_body_size 1000m;
        index index.html;
    }
    location @router {
        rewrite ^.*$ /index.html last;
    }

    error_page 404 /index.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}
```

## deploy/server.Dockerfile

```dockerfile
# ============================================================
# 后端 Dockerfile（预构建，无需编译环境）
# ============================================================
FROM docker.1ms.run/library/alpine:latest

ENV TZ=Asia/Shanghai
RUN apk add --no-cache tzdata \
    && ln -sf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /opt/{{PROJECT_NAME}}

COPY server ./
COPY config/ ./config/
COPY static/ ./static/
COPY config.docker.yaml ./config/config.prod.yaml

EXPOSE {{SERVER_PORT}}

ENTRYPOINT ["./server"]
CMD ["--env=prod", "s"]
```

## deploy/web.Dockerfile

```dockerfile
# ============================================================
# 前端 Dockerfile（预构建，无需编译环境）
# ============================================================
FROM docker.1ms.run/library/nginx:alpine

COPY web/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## deploy/start.sh

```sh
#!/bin/sh
# ============================================================
# 本地开发/测试一键启动（从源码构建，需要 Node/Go 环境）
# 服务器发版请用 pack.sh 打包后部署
# ============================================================
set -e

cd "$(dirname "$0")"

case "${1:-start}" in
    start)
        echo "本地启动（从源码构建）..."
        docker compose up -d --build
        echo "启动完成 - http://localhost:80"
        ;;
    stop)
        docker compose down
        ;;
    restart)
        docker compose restart
        ;;
    logs)
        docker compose logs -f
        ;;
    status)
        docker compose ps
        ;;
    *)
        echo "用法: sh start.sh {start|stop|restart|logs|status}"
        ;;
esac
```

## Makefile（项目根目录）

```makefile
# ============================================================
# 构建/部署 Makefile
# ============================================================

# 默认目标：本地构建前后端
build: build-web build-server

build-web:
	@echo "====== 构建前端 ======"
	@cd web && pnpm config set registry https://registry.npmmirror.com && pnpm install && pnpm run build-only
	@rm -rf build/web && mkdir -p build/web
	@cp -rf web/dist/* build/web/
	@rm -rf web/dist
	@echo "====== 前端构建完成 → build/web/ ======"

build-server:
	@echo "====== 构建后端 ======"
	@go env -w GO111MODULE=on && go env -w GOPROXY=https://goproxy.cn,direct
	@go env -w CGO_ENABLED=0 && go mod tidy
	@mkdir -p build/server
	@cd cmd/server && CGO_ENABLED=0 go build -ldflags "-s -w" -o ../../build/server/server .
	@cp -rf config static build/server/
	@echo "====== 后端构建完成 → build/server/ ======"

# ============================================================
# 部署
# ============================================================

deploy-start:
	@cd deploy && docker compose up -d

deploy-stop:
	@cd deploy && docker compose down

deploy-restart:
	@cd deploy && docker compose restart

deploy-status:
	@cd deploy && docker compose ps

deploy-logs:
	@cd deploy && docker compose logs -f

# 一键发版打包
pack:
	@bash pack.sh

# ============================================================
# 开发辅助
# ============================================================

doc:
	@swag init --parseDependency --parseDepth 1 -d ./cmd/,./ -o ./docs

clean:
	@rm -rf build/ release/ docs/ web/dist/ {{PROJECT_NAME}}-*.zip

install:
	@cd web && pnpm install
	@go mod tidy

# ============================================================
# 帮助
# ============================================================

help:
	@echo "============================================"
	@echo "  {{PROJECT_NAME}} 构建/部署"
	@echo "============================================"
	@echo ""
	@echo "构建:"
	@echo "  make build            构建前后端 → build/"
	@echo "  make build-web        仅构建前端"
	@echo "  make build-server     构建后端"
	@echo ""
	@echo "发版:"
	@echo "  make pack             构建 + 打包 zip"
	@echo ""
	@echo "Docker 部署:"
	@echo "  make deploy-start     docker compose up -d"
	@echo "  make deploy-stop      停止服务"
	@echo "  make deploy-restart   重启服务"
	@echo "  make deploy-logs      查看日志"
	@echo "  make deploy-status    查看状态"
	@echo ""
	@echo "其他:"
	@echo "  make clean            清理构建产物"
	@echo "  make install          安装依赖"
	@echo "  make doc              Swagger 文档"
	@echo ""
```

## pack.sh（项目根目录）

```bash
#!/bin/bash
# ============================================================
# 一键发版打包脚本
# 构建前后端 → 打包为 zip → 服务器解压后 docker compose up -d
# ============================================================
set -e

ROOT="$(cd "$(dirname "$0")" && pwd)"
RELEASE_NAME="{{PROJECT_NAME}}-$(date +%Y%m%d-%H%M%S)"

echo "============================================"
echo "  {{PROJECT_NAME}} 发版打包"
echo "============================================"

echo "[1/3] 构建前后端 ..."
make build

echo "[2/3] 收集产物 ..."
mkdir -p "${ROOT}/release/${RELEASE_NAME}"
cp -rf "${ROOT}/build/web" "${ROOT}/release/${RELEASE_NAME}/web"
cp -rf "${ROOT}/build/server" "${ROOT}/release/${RELEASE_NAME}/server"
cp -rf "${ROOT}/deploy" "${ROOT}/release/${RELEASE_NAME}/deploy"
cp -f "${ROOT}/config.docker.yaml" "${ROOT}/release/${RELEASE_NAME}/" 2>/dev/null || true

echo "[3/3] 打包 zip ..."
cd "${ROOT}/release"
zip -rq "${RELEASE_NAME}.zip" "${RELEASE_NAME}"
rm -rf "${RELEASE_NAME}"
echo "打包完成: release/${RELEASE_NAME}.zip"
```

## .gitignore（项目根目录）

```gitignore
# 构建产物
build/
release/
web/dist/
docs/
{{PROJECT_NAME}}-*.zip

# 依赖
web/node_modules/

# 运行时数据
serverFiles/
dbdata/

# IDE / 系统
.idea/
.vscode/
.DS_Store
```

（`docs/` 为 `make doc` 生成的 Swagger 产物，不入库）

## README.md（项目根目录）

````markdown
# {{PROJECT_TITLE}}

{{PROJECT_TITLE}}（{{PROJECT_NAME}}），Go + Vue3 前后端分离全栈项目。

## 技术栈

- **后端**：Go + gin + ares 框架 + urfave/cli + gorm + MySQL + Redis
- **前端**：Vue3 + Vite + TypeScript + Element Plus + Pinia + Vue Router + Axios
- **部署**：Docker Compose（web/server/mysql/redis）+ nginx

## 目录结构

```
├── cmd/            # CLI 入口
├── config/         # 多环境配置（local/stage/prod）
├── consts/         # 常量、错误码
├── deploy/         # Docker Compose、nginx、Dockerfile
├── docs/           # Swagger 文档（make doc 生成）
├── internal/       # 业务代码（controllers/dto/model/scripts/services/utils）
├── middlewares/    # 中间件
├── sql/            # 建库建表 SQL
├── static/         # 静态资源与上传文件
└── web/            # Vue3 前端
```

## 快速开始

### 后端

```bash
# 1. 启动依赖（或使用本地 MySQL/Redis）
cd deploy && docker compose up -d mysql redis && cd ..

# 2. 初始化数据库
mysql -h127.0.0.1 -uroot -p123456 < sql/init.sql

# 3. 启动（--env: dev/test/prod 对应 local/stage/prod 配置）
go run ./cmd/server --env=dev
```

健康检查：`curl http://127.0.0.1:{{SERVER_PORT}}/ping`

### 前端

```bash
cd web
pnpm install
pnpm dev   # http://localhost:3000
```

## 构建与部署

```bash
make build          # 构建前后端 → build/
make pack           # 构建 + 打包 zip
cd deploy && docker compose up -d   # Docker 部署（http://localhost:80）
```

## 常用命令

| 命令 | 说明 |
|---|---|
| `go run ./cmd/server --env=dev` | 启动后端服务 |
| `go run ./cmd/server test --env=dev` | 运行临时调试代码 |
| `go test ./...` | 运行单元测试 |
| `cd web && pnpm dev` | 前端开发模式 |
| `make doc` | 生成 Swagger 文档 |
```
