# Project Delivery Pattern — WaveYo 风格

> 触发条件：需要将后端服务打包为可部署交付物时加载。
> 来源证据：YoOSF-API、Koishi-Mirror、YoBFF、ip-api 的部署规范。

---

## 部署交付物清单

每个可交付的后端项目必须包含：

```
project/
├── Dockerfile              # 多阶段构建，alpine 基础镜像
├── docker-compose.yml      # 编排应用 + Redis + DB
├── systemctl-files/        # systemd unit 文件
│   └── myapp.service
├── .env.example            # 配置模板
├── release-build.sh        # Linux 构建/发布脚本
├── release-build.ps1       # Windows 构建脚本（可选）
└── README.md               # 含部署步骤
```

---

## Dockerfile 模板（Go 多阶段构建）

```dockerfile
# Stage 1: 编译
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

# Stage 2: 运行
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/server .
COPY --from=builder /app/config.yaml ./
EXPOSE 8080
ENTRYPOINT ["./server"]
```

---

## docker-compose.yml 模板

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "${ADDR:-8080}:8080"
    env_file:
      - .env
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER:-app}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-changeme}
      POSTGRES_DB: ${DB_NAME:-myapp}
    volumes:
      - db_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-app}"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  redis_data:
  db_data:
```

---

## systemd unit 模板

```ini
# systemctl-files/myapp.service
[Unit]
Description=MyApp API Service
After=network.target redis.service postgresql.service
Wants=redis.service postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/server
Restart=always
RestartSec=5
EnvironmentFile=/opt/myapp/.env

# 日志
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# 安全加固
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/opt/myapp/data

[Install]
WantedBy=multi-user.target
```

---

## 构建/发布脚本

```bash
#!/bin/bash
# release-build.sh
set -e

APP_NAME="myapp"
VERSION="${1:-$(git describe --tags --always)}"
BUILD_DIR="build/${APP_NAME}-${VERSION}"

echo "Building ${APP_NAME} ${VERSION}..."

# 编译
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w -X main.version=${VERSION}" \
    -o "${BUILD_DIR}/${APP_NAME}" .

# 复制部署文件
cp -r config.yaml "${BUILD_DIR}/"
cp -r systemctl-files "${BUILD_DIR}/"
cp .env.example "${BUILD_DIR}/.env.example"
cp README.md "${BUILD_DIR}/"

# 打包
tar -czf "${BUILD_DIR}.tar.gz" -C build "${APP_NAME}-${VERSION}"

echo "Release built: ${BUILD_DIR}.tar.gz"
echo "Deploy: tar -xzf ${BUILD_DIR}.tar.gz && sudo cp systemctl-files/*.service /etc/systemd/system/"
```

---

## README 部署步骤模板

```markdown
## 部署

### 使用 systemd

\`\`\`bash
# 1. 解压
tar -xzf myapp-v1.0.0.tar.gz -C /opt/myapp

# 2. 配置环境变量
cp .env.example .env
vim .env  # 修改数据库、Redis、对象存储地址

# 3. 安装 systemd unit
sudo cp systemctl-files/myapp.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp

# 4. 检查运行状态
sudo systemctl status myapp
curl http://localhost:8080/health
\`\`\`

### 使用 Docker Compose

\`\`\`bash
cp .env.example .env
docker compose up -d
curl http://localhost:8080/health
\`\`\`
```

---

## 健康检查端点（必须）

```go
// GET /health
// 200 OK: {"status":"ok","deps":{"db":"ok","redis":"ok","s3":"ok"}}
// 503: {"status":"degraded","deps":{"db":"ok","redis":"error: ..."}}
```

---

## 注意事项

- systemd unit 使用 `Type=simple` + `Restart=always` + `RestartSec=5`
- Docker 镜像使用多阶段构建，最终镜像基于 alpine 减小体积
- .env.example 在首次提交时必须存在，包含所有可配置项
- docker-compose 中每个服务配备 healthcheck 定义
- systemd unit 使用最少权限原则（`NoNewPrivileges`、`ProtectSystem`）
- README 中的部署步骤必须可直接复制执行
- 构建脚本输出包含版本号的压缩包

> 来源：WaveYo / YoOSF-API、Koishi-Mirror、YoBFF、ip-api 的部署模式。
