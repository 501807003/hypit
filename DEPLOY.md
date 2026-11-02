# Hypit 部署记录

本文件记录本 fork（`501807003/hypit`）的部署方式，供后续复现与排障使用。

## 1. 仓库

| 项 | 值 |
|---|---|
| Fork | https://github.com/501807003/hypit |
| 上游 | https://github.com/hypit-ai/hypit |
| 默认分支 | `main` |
| 版本 | 0.3.1 |
| License | Hypit Open Source License |

`origin` 指向本人的 fork，`upstream` 指向上游，便于同步更新：

```bash
git fetch upstream
git merge upstream/main
```

## 2. 环境要求

| 组件 | 版本 | 本机实际 |
|---|---|---|
| Node.js | ≥ 22.15 | v24.21.0 |
| pnpm | 10.33.x | 10.33.0 |
| ffmpeg / ffprobe | 较新稳定版 | 已装（`E:\tools\ffmpeg\bin`） |
| Python | 3.10–3.13 | 3.12.14（仅本地 WhisperX / OpenCV 需要） |
| Chrome/Chromium | 由 `hypit runtime up` 下载 | 由运行时管理 |

本机没有全局 `node` 和 `npm`：使用 DSH 打包的 Node
（`C:\Users\郭宸昊\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe`），
pnpm 直接从 npm registry 取 tarball 解压到 `_tools/pnpm`，不依赖 corepack。

## 3. 本机安装与验证

```powershell
$env:PATH="C:\Users\郭宸昊\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin;$env:PATH"
$env:HTTPS_PROXY="http://127.0.0.1:7890"

cd hypit
node _tools/pnpm/package/bin/pnpm.cjs install --frozen-lockfile --config.dangerouslyAllowAllBuilds=true
node bin/hypit.mjs --version          # 0.3.1
node bin/hypit.mjs check examples/ranking-football/reference.svml   # ✓ Source is valid, Outputs 98
```

`--config.dangerouslyAllowAllBuilds=true` 是必需的：pnpm 默认拦截 `esbuild`、`koffi` 的
安装脚本，而 `koffi` 是 FFI 绑定，渲染链路会直接用到。

运行时绑定与计划：

```powershell
cd examples/ranking-football
node ..\..\bin\hypit.mjs runtime use hypit.runtime.json
node ..\..\bin\hypit.mjs plan reference.svrun     # 29 requests / 14 Provider / 15 Local
node ..\..\bin\hypit.mjs runtime up               # 准备浏览器与本地程序，起 Worker
```

`plan` 报告 `Preflight needs attention` 属预期：`hypihub.default` 端点需要 HypiHub 凭据。
纯代码渲染、字幕、本地媒体系处理不需要该凭据。

### Chrome Headless Shell 手动安装（本机必做）

`hypit runtime up` 首次会下载 Chrome Headless Shell，本机直连 Google 存储超时失败
（265s 后 `Failed`，日志在
`C:\Users\郭宸昊\.cache\hypit\html-rasterizer\chrome\.hypit-html-browser-program\install.log`）。
Worker 不继承 shell 的代理变量，因此必须手动放入：

```powershell
$env:HTTPS_PROXY="http://127.0.0.1:7890"
$d="C:\Users\郭宸昊\.cache\hypit\html-rasterizer\chrome\chrome-headless-shell\win64-152.0.7928.2"
curl.exe -sSL --proxy $env:HTTPS_PROXY --retry 5 --retry-all-errors --max-time 900 `
  -o _tools\chs.zip `
  "https://storage.googleapis.com/chrome-for-testing-public/152.0.7928.2/win64/chrome-headless-shell-win64.zip"
Expand-Archive _tools\chs.zip -DestinationPath $d -Force
Remove-Item "C:\Users\郭宸昊\.cache\hypit\html-rasterizer\chrome\.hypit-html-browser-program\lifecycle.lock" -Force
node bin/hypit.mjs runtime up
```

两个坑：
1. `curl` 会报 `(56) schannel: server closed abruptly`，这是代理侧的 close_notify 缺失，
   报文本身完整——判断依据是解压条目数（290 条）与 `chrome-headless-shell --version`，
   不要只看 curl 退出码。截断的 80 MB 包会报「找不到中央目录结尾记录」。
2. 残留 `lifecycle.lock` 会让运行时认为安装仍在进行，必须删除后再 `runtime up`。

修复后 `hypit runtime status` 输出 `✓ Local Runtime ready / Worker running / Programs 2/2 ready`，
`hypit doctor` 只剩 `RUNTIME_CREDENTIAL_MISSING`（HypiHub 未登录，属预期）。

## 3.5 端到端验证（纯本地，零 Provider 费用）

`examples/semantic-composition` 不调用任何生成模型，是验证整条渲染链的最佳用例：

```powershell
node _tools/pnpm/package/bin/pnpm.cjs build:public-types
node _tools/pnpm/package/bin/pnpm.cjs --filter @example/chat-scene build
node bin/hypit.mjs check  examples/semantic-composition/chat.svml --project examples/semantic-composition
node bin/hypit.mjs build  examples/semantic-composition/chat.svrun --project examples/semantic-composition `
  --runtime examples/semantic-composition/hypit.runtime.json --follow
node bin/hypit.mjs get <build-id> --output final.video --project examples/semantic-composition --to out-chat.mp4
```

实测结果（2026-11-02）：

| 项 | 值 |
|---|---|
| Build | `bld_20261102T060842477Z_7932189202`，`Outcome complete` |
| 请求数 | 3（全部本地） |
| 渲染 | 240 帧，本地 Chromium 光栅化，23 秒 |
| 成片 | `out-chat.mp4`，82 075 字节 |
| 规格 | 540×960 / 30fps / 240 帧 / 8.0s / H.264 + AAC |
| 画面 | `proof-frame.png`，三条聊天气泡按编排时间入场，字体与配色正常 |

## 4. GitHub Actions

fork 上的工作流默认处于「仅手动」状态，首次 push 不会自动运行。已处理：

- `PUT /repos/501807003/hypit/actions/permissions` → `enabled: true`，允许全部 action。
- `PUT /repos/501807003/hypit/actions/permissions/workflow` → 默认 workflow 权限 `write`。
- push 一次提交，触发 GitHub 注册 fork 的 workflow 文件（此前 `actions/workflows` 返回 0 条）。

已注册并激活 6 条：

| 工作流 | 文件 | 触发 |
|---|---|---|
| CI | `ci.yml` | push / PR / 手动 |
| npm package execution | `npm-package.yml` | 手动 / 被 CI 调用 |
| Publish npm | `publish-npm.yml` | tag |
| Repository analysis | `repository-analysis.yml` | 手动 |
| Check repository analysis | `automation-check.yml` | push / 手动 |
| Notify HypiHub of documentation and asset updates | `sync-hypihub-docs.yml` | push |

手动触发一次完整 CI：

```bash
gh workflow run ci.yml            # 或用 REST：
curl -X POST -H "Authorization: token $TOKEN" \
  https://api.github.com/repos/501807003/hypit/actions/workflows/ci.yml/dispatches \
  -d '{"ref":"main"}'
```

CI 矩阵：`ubuntu-latest` + `windows-latest` 双平台，步骤为
`pnpm install --frozen-lockfile` → 装 ffmpeg → `pnpm check`（tsc）→ `pnpm test`；
另有 `package` job 跑 `npm run pack:release-candidate` 并上传 tarball 制品。

## 5. 关于「部署」的说明

Hypit 不是 Web 服务，没有前端站点或常驻 HTTP 端口。它的形态是：

1. **CLI**（`bin/hypit.mjs`，通过 npm 包 `@hypit/hypit` 分发）；
2. **Skill**（`skills/hypit`，由 `npx skills add hypit-ai/hypit -g` 安装给 Claude Code / Codex 等 Agent）；
3. **本地 Runtime**（Worker + 无头 Chromium + ffmpeg，`hypit runtime up` 拉起）。

因此「部署到 GitHub」的实质是：代码托管于本人 fork、CI 在 GitHub 上完成类型检查/测试/打包，
产物以 Actions artifact 形式留存；真正跑视频渲染仍在本机或你指定的机器上执行。

## 6. 凭据

Hypit 走 HypiHub 托管服务或自带 API Key。凭据只写入本机 Credential Store
（`hypit auth login <endpoint>`），不进入 Author Source、Run Source 或 Runtime Profile。
`.hypit/`、`output/` 已在 `.gitignore` 中，不应提交生成媒体与运行状态。
