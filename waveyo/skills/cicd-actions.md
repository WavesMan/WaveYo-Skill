# CI 基础设施规范（以 Gitea Actions + act_runner 为例）

> 触发时机：编写 / 修改 `.gitea/workflows/*.yml`、调整 runner 容器、CI 出现网络超时或 TLS 握手失败、
> job 一直排队无人拾取时必读。
> **关键事实**：`actions/*` 的拉取由 **act_runner 进程**执行（不在 job 容器内），
> 因此代理必须配在 **runner 容器 env**；workflow 的 `env:` 管不到 action 拉取——
> 这是「本地能拉、CI 超时」的常见根因（见 §3）。
>
> 本文用占位符描述环境：`<GITEA_URL>`、`<PROXY_HOST>:<PROXY_PORT>`、`<HOST_DATA_DIR>`、`<internal-cidr>`。
> 落地时替换为项目自己的值，**不要把真实内网地址与主机路径写进文档**。

---

## 1. Runner 部署形态

一种可用的最小形态：单机容器运行时（Docker / Podman）上常驻 2～3 个 act_runner，
实例级注册，服务该实例下的所有仓库。

| 项 | 参考值 | 说明 |
|---|---|---|
| 镜像 | `docker.gitea.com/act_runner:<pinned-version>` | **pin 版本，勿用 `latest`**（升级会改行为） |
| 容器 | `gitea-runner` / `gitea-runner-2` / `gitea-runner-3` | `--restart unless-stopped` |
| 网络 | 与 `gitea` 服务同网络（如 `<project>_default`） | job 容器加入同网才能解析服务名 |
| 实例地址 | `GITEA_INSTANCE_URL=http://gitea:3000` | **容器内网名 + 容器端口**，不是宿主机的对外地址 |
| 注册级别 | 实例级（admin registration token） | 实例级才服务所有仓库；org 级只服务该 org |
| 名称 | `gitea-runner-linux-x64[-2/-3]` | 名称用于日志定位 |
| 挂载 | 容器运行时 socket + `<HOST_DATA_DIR>/runner[-2/-3]:/data` | `.runner` 态在卷里持久，重建容器不丢注册 |
| 重启策略 | `unless-stopped` | 与持久卷配合 |

**config.yaml 关键项**（每个 runner 各持一份，路径 `/data/config.yaml`）：

```yaml
container:
  network: "<gitea_network>"   # job 容器加入同网，才能解析 gitea 主机名、checkout 代码
  privileged: false
cache:
  enabled: true
  dir: "/data/actcache"        # 缓存落卷，容器重建不丢
```

> 若 `container.network` 留空，job 容器走默认 bridge，无法解析 `gitea` → checkout 失败。

**重建一个 runner**（`.runner` 态已在卷里，通常无需重新注册；仅首次或换实例时才需 token）：

```bash
docker rm -f gitea-runner-2
docker run -d --name gitea-runner-2 --restart unless-stopped --network <gitea_network> \
  -e GITEA_INSTANCE_URL=http://gitea:3000 \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=<实例级 token> \
  -e GITEA_RUNNER_NAME=gitea-runner-linux-x64-2 \
  -e CONFIG_FILE=/data/config.yaml \
  -e HTTP_PROXY=http://<PROXY_HOST>:<PROXY_PORT> -e HTTPS_PROXY=http://<PROXY_HOST>:<PROXY_PORT> \
  -e http_proxy=http://<PROXY_HOST>:<PROXY_PORT> -e https_proxy=http://<PROXY_HOST>:<PROXY_PORT> \
  -e NO_PROXY=localhost,127.0.0.1,gitea,<internal-cidr> \
  -e no_proxy=localhost,127.0.0.1,gitea,<internal-cidr> \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v <HOST_DATA_DIR>/runner-2:/data \
  docker.gitea.com/act_runner:<pinned-version>
```

**实例级 registration token 取法**（需要 admin 权限的 API token）：

```bash
curl -X POST -H "Authorization: token <admin_token>" \
  <GITEA_URL>/api/v1/admin/actions/runners/registration-token
```

**验证 runner 在线**：`docker logs gitea-runner-2 | grep -i "declare successfully"`，
或 `GET <GITEA_URL>/api/v1/admin/actions/runners` 看 `status=online`。

---

## 2. 代理配置（三层）

若 CI 需要访问外网、而访问必须经宿主侧代理，代理要配在**三层**，缺一不可：

| 层 | 位置 | 作用对象 | 解决的问题 |
|---|---|---|---|
| ① runner 容器 env | `docker run -e HTTP_PROXY=...` | act_runner 进程的 git | **action 拉取**（`actions/checkout@v4` 等，从代码托管站 clone） |
| ② 服务端配置 | `app.ini [actions] DEFAULT_ACTIONS_URL=<actions-mirror>` | action 解析源 | 把 `actions/*` 指向可访问的镜像源 |
| ③ workflow `env:` | `.gitea/workflows/*.yml` 的 job 级 env | job 容器内步骤（apt / 包管理器 / curl） | job 内访问外网 |

**NO_PROXY 必须包含内网**：`localhost,127.0.0.1,gitea,<internal-cidr>`，
否则 gitea 与内部服务的流量被劫持到代理 → 内网调用失败。

> ①与②互为冗余：只配①（runner 走代理）通常已能工作；②进一步把 action 源换到可访问的镜像，更稳。
> 两者都开最稳。

**代理地址的来源**：容器内看到的「宿主机」通常是 NAT 网关地址，重启后可能变化。
取值方式随环境不同（如 WSL2 下 `ip route show default` 的网关），
变化后需更新 runner 容器 env 与 workflow 的 `env:`。

---

## 3. TLS handshake timeout 根因与规避

**现象**：workflow 在「拉取代码」步骤之前失败，job 日志末尾：

```
☁  git clone 'https://<code-host>/actions/checkout' # ref=v4
cloning https://<code-host>/actions/checkout to /root/.cache/act/...
Get "https://<code-host>/actions/checkout/info/refs?service=git-upload-pack": net/http: TLS handshake timeout
🏁  Job failed
```

**根因**：action 仓库的 clone 由 **act_runner 进程**（runner 容器内）执行，
缓存到 runner 容器的 `/root/.cache/act/`，**不在 job 容器内**。
所以 workflow 的 `env:` 代理（只注入 job 容器）**管不到 action 拉取**。
act_runner 直连代码托管站 → 握手超时（时通时断，表现为偶发失败或长时间后成功）。

**规避（按优先级）**：

1. **配 runner 容器代理 env**（§2 ①）——act_runner 的 git 经代理访问代码托管站。
2. **设 `DEFAULT_ACTIONS_URL`**（§2 ②）——action 源切到可访问的镜像。
3. **job 容器代理**（§2 ③）——只影响 job 内步骤，不影响 action 拉取，但包管理器/apt 必需。

> 排障顺序：先看 `docker logs gitea-runner` 与 job 日志里的 `git clone` 行；
> 若超时，确认 runner 容器有代理 env：`docker exec gitea-runner env | grep -i proxy`。

---

## 4. workflow 模板

`.gitea/workflows/ci.yml` 在 job 级注入代理 env（对应 §2 ③ 层）：

```yaml
name: CI
on:
  push:
    branches: [main]     # 按项目分支模型调整
  pull_request:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      # 代理（job 容器经容器网络网关转发到宿主机侧代理）
      HTTP_PROXY: http://<PROXY_HOST>:<PROXY_PORT>
      HTTPS_PROXY: http://<PROXY_HOST>:<PROXY_PORT>
      http_proxy: http://<PROXY_HOST>:<PROXY_PORT>
      https_proxy: http://<PROXY_HOST>:<PROXY_PORT>
      # 内网不走代理
      NO_PROXY: localhost,127.0.0.1,gitea,<internal-cidr>
      no_proxy: localhost,127.0.0.1,gitea,<internal-cidr>
    steps:
      - uses: actions/checkout@v4
      - name: build & test
        run: |
          # 按项目实际语言替换；示例为 Go
          go build ./...
          go vet ./...
          go test ./... -short
```

> `runs-on: ubuntu-latest` 需与 runner 注册的 labels 对应。
> 若走包管理器代理，可与 `GOPROXY` / pip index / npm registry 镜像叠加使用。

**CI 门禁建议**：构建 + 静态检查 + 短测试作为 PR 必过项；
集成测试、端到端、界面截图对比等重型检查按项目成本决定，不要为了「齐全」把 PR 反馈时间拖到十分钟以上。

---

## 5. 维护与排障速查

| 症状 | 排查 |
|---|---|
| run 一直 `queued`、无人拾取 | runner 是否 `online`；注册级别是否覆盖该仓库 |
| job 日志 `git clone ... TLS handshake timeout` | runner 容器缺代理 env（§2 ①）；或设 `DEFAULT_ACTIONS_URL`（§2 ②） |
| job 内访问内网服务超时 | `NO_PROXY` 漏了服务名，内网流量被劫持到代理 |
| 容器重启后代理失效 | 网关地址变了，重新取值并更新 runner env 与 workflow env |
| `.runner` 态丢失导致重注册 | 重建前确认 `/data` 卷挂载正确；重注册会生成新 runner id，旧 id 需后台删除 |
| 想省资源改按需 runner | 可用 ephemeral / watchdog 方案；常驻少量 runner 的空闲开销通常可接受 |
| 本地能拉、CI 不能拉 | 先分清是 **action 拉取**（runner 进程）还是 **job 内步骤**（job 容器）失败 |

**多副本同步（可选）**：若同一份 skill / 规范需要在多个目录保留物理副本（如不同工具的 skills 目录），
复制前先删除目标目录再整目录复制——目标已存在时直接复制会嵌套出 `x/x/` 结构。
符号链接可用时优先用链接，避免多副本漂移。

---

## 6. 注意

- 本文示例只说明**机制与排障路径**，不代表任何具体实例的配置。
- 不要把真实的实例地址、内网网段、宿主机路径、admin token 写进仓库文档或日志。
- 具体托管平台（GitHub Actions / GitLab CI / Jenkins）的等价机制：
  runner 级别环境变量控制「拉 action/插件」，job 级别环境变量控制「构建步骤」——分层的道理相同。
