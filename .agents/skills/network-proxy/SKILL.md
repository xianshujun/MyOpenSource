---
name: network-proxy
description: 通过用户 Windows 上的 Clash 代理访问被国内网络屏蔽的站点（Google、GitHub、OpenAI/Anthropic、Hugging Face、Docker Hub 等）。当 curl 返回 000/超时/连接被拒、git clone github 失败、go get 拉不通、apt/npm 源超时，或需要访问 google/github 等墙外服务时使用。代理不可用时必须向用户询问三件事：Windows 的 IP、Clash 是否开启、是否开启「允许局域网连接」。
---

# 网络代理（Windows Clash 出口）

本机是内网里的容器，**直连 Google、GitHub、OpenAI 等站点不通**（实测 `curl` 返回 `000`）。
唯一出口是用户 Windows 上的 Clash，它在局域网里监听一个混合端口（HTTP 与 SOCKS5 都收，默认 7897）。

链路：`容器 → 宿主机 → 内网 → Windows:PORT → Clash → 互联网`

## 端点从哪里读

**不要在本文件里找 IP**——真实地址记在同目录的 `endpoint.local`（被 `.gitignore` 排除，不进公开仓库）：

```bash
cat "$(dirname "$0")/endpoint.local"   # 内容形如：WIN_IP=...  PORT=7897
```

读到的值代入下文的 `<WIN_IP>` / `<PORT>`。若该文件不存在、或按它自检失败，走「代理不可用时」的流程。

Windows 的 IP 由 DHCP 分配，**每次开机可能变化**。任何一次使用前先做连通性自检，失败就问用户，不要拿旧 IP 反复重试。

## 判断是否需要走代理

先直连测一次（几秒即可）：

```bash
curl -s -o /dev/null -m 6 -w '%{http_code}\n' https://www.google.com   # 000 = 不通，需要代理
```

- **需要代理**：`google*`、`github*`、`api.openai.com`、`anthropic.com`、`huggingface.co`、`docker.io`、`proxy.golang.org`、`go.dev` 等墙外站点。
- **保持直连**：国内站点与镜像（`goproxy.cn`、`npmmirror.com`）、公司内网服务（含内网 LLM 网关）、`localhost`。绕代理会变慢甚至失败。

## 使用方式

命令级前缀（一次性，不影响其他命令，是本环境的首选方式）：

```bash
https_proxy=http://<WIN_IP>:<PORT> http_proxy=http://<WIN_IP>:<PORT> <命令>
```

curl 可以直接用 `-x`：

```bash
curl -x http://<WIN_IP>:<PORT> https://api.github.com/...
```

已经被环境预置好、**不需要额外加前缀**的：

- **git 访问 github.com**：已配置 `http.https://github.com.proxy`，`git clone/fetch/push` 自动走代理。
- **交互式终端里手敲的 `curl`**：`~/.bashrc` 里的 `curl()` 函数在参数含 `google` 时自动加 `-x`（非交互式 shell、脚本、其他程序不经过它）。

本环境**刻意不做全局代理**（`~/.bashrc` 里有 `unset http_proxy`），这是用户明确要求：不要往 rc 文件或系统配置里写全局代理变量，也不要修改 `~/.bashrc` 里的代理相关段落。需要代理时用命令级前缀。

## 代理不可用时（必须问用户，不要自己猜）

症状：连接 `<WIN_IP>:<PORT>` 超时 / 拒绝。

**不要扫描网段、不要猜测新 IP**——这是企业内网，主动扫段不合适，扫描结果也无法确认哪台是用户的机器。

用 `AskUserQuestion` 一次性问清三件事：

1. **你 Windows 现在的局域网 IP 是多少？** 让用户在 Win 上跑 `ipconfig`，看「WLAN」或「以太网」适配器的 IPv4 地址。
2. **Win 上的 Clash 开着吗？**
3. **Clash 的「允许局域网连接 / Allow LAN」是开启状态吗？**

拿到答案后按情况处理：

- **IP 变了**：把新值写回 `endpoint.local`（格式 `WIN_IP=...`），并更新 git 配置：
  ```bash
  git config --global http.https://github.com.proxy http://<新IP>:<PORT>
  ```
  然后重新自检。同时提醒用户：IP 会变说明是 DHCP，建议在路由器里给这台 Win 绑定静态 IP。
- **Clash 没开 / 局域网连接没开**：请用户开启后重试；不要尝试其他出口。
- **三项都正常但仍不通**：让用户在 Win 本机自测 `Test-NetConnection 127.0.0.1 -Port <PORT>` 确认 Clash 真的在监听；若监听正常，大概率是 Windows 防火墙拦了入站，让用户以管理员身份放行：
  ```powershell
  New-NetFirewallRule -DisplayName "Clash7897" -Direction Inbound -Protocol TCP -LocalPort <PORT> -Action Allow
  ```

## 连通性自检

```bash
# 1) 端口是否可达（本环境没有 nc，用 /dev/tcp）
timeout 3 bash -c "echo > /dev/tcp/<WIN_IP>/<PORT>" && echo OPEN || echo UNREACHABLE

# 2) 代理是否真的能出网
curl -s -o /dev/null -m 8 -w '%{http_code}\n' -x http://<WIN_IP>:<PORT> https://www.google.com
# 期望 200
```

## 已知的坑

- **`198.18.0.1` 不是可用的代理地址**。那是 Clash TUN 模式虚拟网卡（Meta 适配器）的地址，只在 Windows 本机内部有效，外部无法路由。必须使用 WLAN/以太网适配器的局域网 IP。（曾经踩过：对着它 curl 只会得到 `Connection timed out`。）
- Windows 防火墙对新监听的端口默认拦入站；Clash 若没被放行，症状是**连接超时**而不是拒绝。
- `no_proxy` 必须排除 `localhost`、`127.0.0.1` 与内网网段，否则访问内网服务会被无谓地绕经用户的家宽/办公网络。
- 老版本 curl（如 7.29）的 `no_proxy` 不支持 CIDR 写法，只认域名后缀和精确 IP；Go 程序则支持 CIDR。
