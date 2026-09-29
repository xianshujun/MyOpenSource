# AGENTS.md

本仓库 **MyOpenSource** 是 `xianshujun` 的开源练习场，同时用于存放可复用的 agent skill。
仓库是**公开**的，因此任何凭据、密钥、内网地址、个人隐私都不得提交。

## 目录约定

```
.
├── AGENTS.md                      # 本文件：agent 在本仓库工作时的规则
├── .gitignore
├── skill/<skill-name>/            # 技能统一放这里（本地目录，被 .gitignore 排除）
│   ├── SKILL.md                   # 必需：YAML frontmatter + markdown 正文
│   └── ...                        # 可带 scripts/、references/、*.local 等
└── <练习项目>/                     # 参与上游项目的克隆，如 adk-go/（同样不提交）
```

## submodule 指针（agent 主动维护）

`adk-go` 是登记在本仓库里的 submodule（`.gitmodules` 指向自己的 fork）。它的提交历史与父仓库**分开**，所以在 adk-go 里提交之后，父仓库记的那条指针会落后。

- 发现 `git status` 里 `adk-go` 显示 `new commits` / `modified content` 时，**主动**更新并提交指针，不用等用户提：
  ```bash
  git add adk-go && git commit -m "chore: 更新 adk-go 指针（说明进展）"
  ```
- 新克隆本仓库后 `adk-go/` 为空时，`git submodule update --init --recursive` 拉取。
- 完整流程、排查方法和「不要做的事」见 `skill/submodule-sync/SKILL.md`。

## 技能怎么放、怎么读（重要）

- **所有技能一律放在 `skill/<skill-name>/` 下，也一律从 `skill/` 里读取**：需要用到某个技能时，读 `skill/<skill-name>/SKILL.md`；它 frontmatter 里的 `description` 就是判断「这个技能是否与当前任务相关」的唯一依据。
- `skill/` 已被 `.gitignore` 排除，**是本地目录，不进公开仓库**。所以技能里可以写本机专属内容（内网 IP、私密端点等），但**密钥本体仍然不许写进任何文件**。
- 当前技能清单（新增技能时同步更新这一段）：
  - `skill/network-proxy/` —— 通过用户 Windows 上的 Clash 代理访问墙外站点。端点值在同目录的 `endpoint.local`。
  - `skill/visual-check/` —— 用无头浏览器看前端页面：截图、量样式、排查「闪一下/跳一下」。附带 `scripts/` 与 `INSTALL.md`。
  - `skill/submodule-sync/` —— 维护 `adk-go` 等 submodule 的指针同步（见下一节）。
- 公开仓库里另保留一份**可对外发布**的副本 `.agents/skills/network-proxy/`（与 `skill/network-proxy/` 内容一致，供他人复用）。**读取时以 `skill/` 为准**；技能内容有改动时，两份一起改。
- 新增技能时还要遵循：
  - 目录名与 frontmatter 里的 `name` 一致，小写 kebab-case。
  - `description` 写清楚「做什么」+「什么时候触发」，宁可写得积极一些。
  - 正文用祈使句，控制在 500 行以内；长内容拆到同目录的 `references/` 下按需读取。

## 网络规则（重要）

本机（服务器容器）位于内网，**直连 Google、GitHub、OpenAI 等站点不通**。
出口是用户 Windows 上的 Clash（已开启「允许局域网连接」），其地址记在
`skill/network-proxy/endpoint.local`，默认端口 7897。

- 访问被墙站点前，先按 `skill/network-proxy/SKILL.md` 里的流程取用代理。
- 代理不可用时，**不要自行猜测或扫描网段**，按该技能要求向用户询问三件事（Win 的 IP、Clash 是否开启、是否开启局域网连接）。
- 本环境**刻意不做全局代理**：不要往 `~/.bashrc` 等配置文件里写全局 `http_proxy`，用命令级前缀或工具自身配置。
- 国内可直连的资源（`goproxy.cn`、npm/apt 国内镜像、内网服务）保持直连，不要绕代理。

## 环境事实

- 工作目录 `/workspace/github`，ZCode 运行在内网的 Docker 容器中（无 `ip`/`ping`/`nc` 命令，用 `curl`、`/dev/tcp` 代替）。
- Go：`go1.22.2`，`GOTOOLCHAIN=auto`（高版本 go.mod 会自动下载工具链，走 `GOPROXY=goproxy.cn`，已验证可用）。
- `gh` CLI 已安装在 `~/bin/gh`，已登录 GitHub 账号 `xianshujun`，git 走 https 协议。
- git 全局身份用的是工作邮箱，**不要让它出现在公开仓库的提交里**。GitHub 账号是 `xianshujun`，其已验证邮箱按仓库本地配置（`git config user.email`），值不要写进本文件。
- 提交 Google 相关开源项目（如 adk-go）时，必须用 GitHub 账号已验证的邮箱，否则 CLA 机器人匹配不上。
- 已有 git 配置：`http.https://github.com.proxy` 指向 Win 上的 Clash，仅对 github.com 生效。

## LLM 与密钥

- 本项目优先用内网的 OpenAI 兼容端点，模型名需带 provider 前缀（如 `deepseek/deepseek-flash`），不再用 Google 的 key；该端点内网直连可达，不要走代理。
- 密钥与 base-url 存放在容器内 `/root/.config/adk-go/env.local`（仓库外，绝不提交、不要把值贴进任何文件）；一条命令跑起来：`/root/.config/adk-go/run.sh console`。
- **ADK 的 `model/openaimodel` 走 OpenAI Responses API（`POST /v1/responses`）**，只实现 chat completions 的网关跑不通——接新端点前先确认它支持 Responses API。
- 模型名写错（漏掉 `provider/` 前缀）会得到 `503 no_provider_error: No matching provider found`，别误判成服务故障。

## 参与开源项目的约定

- 上游仓库一律加为 `upstream` remote，自己的 fork 为 `origin`。
- 每个改动单独开分支，分支名带 issue 编号（如 `fix-1463-misspell`）。
- 提交前本地跑通目标包的测试：`go test ./<pkg>/... -count=1`。
- 开 PR 用 `gh pr create`，PR 描述里引用 issue 编号。
- 不要提交任何密钥；密钥只在运行时通过环境变量传入。
