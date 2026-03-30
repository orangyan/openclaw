# 项目指南

- 仓库：https://github.com/openclaw/openclaw
- GitHub issues/PR 评论：使用字面量多行字符串或 `-F - <<'EOF'`（或 `$'...'`）表示真实换行；永远不要嵌入 `"\\n"`。
- GitHub 评论坑：永远不要在 body 包含反引号或 shell 字符时使用 `gh issue/pr comment -b "..."`。始终使用单引号 heredoc（`-F - <<'EOF'`）以避免命令替换/转义损坏。
- GitHub 链接坑：不要在想要自动链接时将 issue/PR 引用如 `#24643` 包在反引号中。使用纯 `#24643`（或可选地添加完整 URL）。
- 安全公告分析：在进行分类/严重性决策之前，请阅读 `SECURITY.md` 以了解 OpenClaw 的信任模型和设计边界。

## 项目结构与模块组织

- 源代码：`src/`（CLI 接线在 `src/cli`，命令在 `src/commands`，web provider 在 `src/provider-web.ts`，基础设施在 `src/infra`，媒体管道在 `src/media`）。
- 测试：就近放置 `*.test.ts`。
- 文档：`docs/`（图片、队列、Pi 配置）。构建输出在 `dist/`。
- 插件/扩展：位于 `extensions/*`（工作区包）。将插件专属依赖放在扩展的 `package.json` 中；除非核心使用，否则不要添加到根 `package.json`。
- 插件：install 在插件目录运行 `npm install --omit=dev`；运行时依赖必须在 `dependencies` 中。避免在 `dependencies` 中使用 `workspace:*`（npm install 会失败）；将 `openclaw` 放在 `devDependencies` 或 `peerDependencies` 中（运行时通过 jiti 别名解析 `openclaw/plugin-sdk`）。
- 安装程序：从 `https://openclaw.ai/*` 提供：位于兄弟仓库 `../openclaw.ai`（`public/install.sh`、`public/install-cli.sh`、`public/install.ps1`）。
- 消息通道：重构共享逻辑（路由、允许列表、配对、命令门控、入职、文档）时，始终考虑所有内置 + 扩展通道。
  - 核心通道文档：`docs/channels/`
  - 核心通道代码：`src/telegram`、`src/discord`、`src/slack`、`src/signal`、`src/imessage`、`src/web`（WhatsApp web）、`src/channels`、`src/routing`
  - 扩展（通道插件）：`extensions/*`（如 `extensions/msteams`、`extensions/matrix`、`extensions/zalo`、`extensions/zalouser`、`extensions/voice-call`）
- 添加通道/扩展/应用/文档时，更新 `.github/labeler.yml` 并创建匹配的 GitHub 标签（使用现有通道/扩展标签颜色）。

## 文档链接（Mintlify）

- 文档托管在 Mintlify（docs.openclaw.ai）。
- `docs/**/*.md` 中的内部文档链接：根相对，无 `.md`/`.mdx`（示例：`[Config](/configuration)`）。
- 处理文档时，请阅读 mintlify 技能。
- 章节交叉引用：使用根相对路径上的锚点（示例：`[Hooks](/configuration#hooks)`）。
- 文档标题和锚点：避免在标题中使用破折号和撇号，因为它们会破坏 Mintlify 锚点链接。
- 当 Peter 要求链接时，回复完整的 `https://docs.openclaw.ai/...` URL（不是根相对）。
- 当你修改文档时，在回复末尾附上你引用的 `https://docs.openclaw.ai/...` URL。
- README（GitHub）：保留绝对文档 URL（`https://docs.openclaw.ai/...`）以便在 GitHub 上链接有效。
- 文档内容必须通用：不要包含个人设备名称/主机名/路径；使用占位符如 `user@gateway-host` 和 "gateway host"。

## 文档国际化（zh-CN）

- `docs/zh-CN/**` 是生成的；除非用户明确要求，否则不要编辑。
- 流程：更新英文文档 → 调整词汇表（`docs/.i18n/glossary.zh-CN.json`）→ 运行 `scripts/docs-i18n` → 仅在收到指示时进行针对性修复。
- 翻译记忆：`docs/.i18n/zh-CN.tm.jsonl`（生成）。
- 参见 `docs/.i18n/README.md`。
- 管道可能很慢/低效；如果卡住了，在 Discord 上 ping @jospalmbier 而不是绕过它。

## exe.dev VM 运维（通用）

- 访问：稳定路径是 `ssh exe.dev` 然后 `ssh vm-name`（假设 SSH 密钥已设置）。
- SSH 不稳定：使用 exe.dev web 终端或 Shelley（web agent）；长时间操作保持 tmux 会话。
- 更新：`sudo npm i -g openclaw@latest`（全局安装需要在 `/usr/lib/node_modules` 上有 root 权限）。
- 配置：使用 `openclaw config set ...`；确保 `gateway.mode=local` 已设置。
- Discord：仅存储原始 token（不要 `DISCORD_BOT_TOKEN=` 前缀）。
- 重启：停止旧 gateway 并运行：
  `pkill -9 -f openclaw-gateway || true; nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &`
- 验证：`openclaw channels status --probe`、`ss -ltnp | rg 18789`、`tail -n 120 /tmp/openclaw-gateway.log`。

## 构建、测试和开发命令

- 运行时基准：Node **22+**（保持 Node + Bun 路径工作）。
- 安装依赖：`pnpm install`
- 如果依赖缺失（例如 `node_modules` 缺失、`vitest not found` 或 `command not found`），运行仓库的包管理器安装命令（优先使用 lockfile/PM 定义），然后立即重新运行请求的命令。将此应用于 test/build/lint/typecheck/dev 命令；如果重试仍然失败，报告命令和第一个可操作的错误。
- 预提交钩子：`prek install`（运行与 CI 相同的检查）。
- 也支持：`bun install`（在接触依赖/补丁时保持 `pnpm-lock.yaml` + Bun 补丁同步）。
- 优先使用 Bun 执行 TypeScript（脚本、开发、测试）：`bun <file.ts>` / `bunx <tool>`。
- 开发模式运行 CLI：`pnpm openclaw ...`（bun）或 `pnpm dev`。
- Node 仍然支持运行构建输出（`dist/*`）和生产安装。
- Mac 打包（开发）：`scripts/package-mac-app.sh` 默认为当前架构。发布清单：`docs/platforms/mac/release.md`。
- 类型检查/构建：`pnpm build`
- TypeScript 检查：`pnpm tsgo`
- Lint/格式化：`pnpm check`
- 格式化检查：`pnpm format`（oxfmt --check）
- 格式化修复：`pnpm format:fix`（oxfmt --write）
- 测试：`pnpm test`（vitest）；覆盖率：`pnpm test:coverage`

## 代码风格与命名约定

- 语言：TypeScript（ESM）。优先严格类型；避免 `any`。
- 格式化/linting 通过 Oxlint 和 Oxfmt；提交前运行 `pnpm check`。
- 永远不要添加 `@ts-nocheck` 且不要禁用 `no-explicit-any`；修复根本原因，仅在需要时更新 Oxlint/Oxfmt 配置。
- 永远不要通过原型突变共享类行为（`applyPrototypeMixins`、`Object.defineProperty` 在 `.prototype` 上，或导出 `Class.prototype` 用于合并）。使用显式继承/组合（`A extends B extends C`）或辅助组合，以便 TypeScript 可以类型检查。
- 如果需要此模式，请在发货前停止并获得明确批准；默认行为是拆分/重构为显式类层次结构并保持成员强类型。
- 在测试中，优先使用每个实例的 stub 而不是原型突变（`SomeClass.prototype.method = ...`），除非测试明确记录为什么需要原型级修补。
- 为棘手或非显而易见的逻辑添加简要代码注释。
- 保持文件简洁；提取辅助函数而不是"V2"副本。对 CLI 选项使用现有模式，通过 `createDefaultDeps` 进行依赖注入。
- 目标是将文件保持在约 700 行以下；仅作为指导（非硬性护栏）。在提高清晰度或可测试性时进行拆分/重构。
- 命名：产品/应用/文档标题使用 **OpenClaw**；CLI 命令、包/二进制、路径和配置键使用 `openclaw`。

## 发布渠道（命名）

- stable：仅限标记发布（如 `vYYYY.M.D`），npm dist-tag `latest`。
- beta：预发布标签 `vYYYY.M.D-beta.N`，npm dist-tag `beta`（可能不包含 macOS 应用）。
- beta 命名：优先使用 `-beta.N`；不要创建新的 `-1/-2` beta。旧的 `vYYYY.M.D-<patch>` 和 `vYYYY.M.D.beta.N` 仍然被识别。
- dev：`main` 的移动头部（无标签；git checkout main）。

## 测试指南

- 框架：Vitest，V8 覆盖率阈值（70% 行/分支/函数/语句）。
- 命名：与源名称匹配 `*.test.ts`；e2e 在 `*.e2e.test.ts`。
- 接触逻辑后推送前运行 `pnpm test`（或 `pnpm test:coverage`）。
- 不要将测试 workers 设置高于 16；已经试过了。
- 如果本地 Vitest 运行导致内存压力（非 Mac-Studio 主机上常见），使用 `OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test` 进行 land/gate 运行。
- 实时测试（真实密钥）：`CLAWDBOT_LIVE_TEST=1 pnpm test:live`（仅 OpenClaw）或 `LIVE=1 pnpm test:live`（包含 provider 实时测试）。Docker：`pnpm test:docker:live-models`、`pnpm test:docker:live-gateway`。入职 Docker E2E：`pnpm test:docker:onboard`。
- 完整工具包及覆盖内容：`docs/testing.md`。
- Changelog：仅限面向用户的更改；无内部/元注释（版本对齐、appcast 提醒、发布流程）。
- 纯测试添加/修复通常不需要 changelog 条目，除非它们改变了面向用户的行为或用户要求。
- 移动端：使用模拟器之前，检查已连接的真实设备（iOS + Android）并在可用时优先使用。

## 提交与拉取请求指南

**完整的维护者 PR 工作流程（可选）：**如果你想要仓库的端到端维护者工作流程（分类顺序、质量栏、rebase 规则、提交/changelog 约定、贡献者政策以及 `review-pr` > `prepare-pr` > `merge-pr` 管道），请参阅 `.agents/skills/PR_WORKFLOW.md`。维护者可以使用其他工作流程；当维护者指定工作流程时，遵循该工作流程。如果没有指定工作流程，默认为 PR_WORKFLOW。

- 使用 `scripts/committer "<msg>" <file...>` 创建提交；避免手动 `git add`/`git commit` 以保持暂存区范围化。
- 遵循简洁的、面向操作的提交消息（例如，`CLI: add verbose flag to send`）。
- 分组相关更改；避免捆绑无关的重构。
- PR 提交模板（规范）：`.github/pull_request_template.md`
- Issue 提交模板（规范）：`.github/ISSUE_TEMPLATE/`

## 简写命令

- `sync`：如果工作树脏了，提交所有更改（选择合理的 Conventional Commit 消息），然后 `git pull --rebase`；如果 rebase 冲突且无法解决，则停止；否则 `git push`。

## Git 注意

- 如果 `git branch -d/-D <branch>` 被策略阻止，直接删除本地引用：`git update-ref -d refs/heads/<branch>`。
- 批量关闭/重新打开 PR 安全：如果关闭操作会影响超过 5 个 PR，首先请求用户确认确切的 PR 数量和目标范围/查询。

## GitHub 搜索（`gh`）

- 在提出新工作或重复修复之前，优先使用有针对性的关键字搜索。
- 首先使用 `--repo openclaw/openclaw` + `--match title,body`；在分类跟进线程时添加 `--match comments`。
- PR：`gh search prs --repo openclaw/openclaw --match title,body --limit 50 -- "auto-update"`
- Issue：`gh search issues --repo openclaw/openclaw --match title,body --limit 50 -- "auto-update"`
- 结构化输出示例：
  `gh search issues --repo openclaw/openclaw --match title,body --limit 50 --json number,title,state,url,updatedAt -- "auto update" --jq '.[] | "\(.number) | \(.state) | \(.title) | \(.url)"'`

## 安全与配置提示

- Web provider 将凭据存储在 `~/.openclaw/credentials/`；如果登出请重新运行 `openclaw login`。
- Pi 会话默认位于 `~/.openclaw/sessions/`；基础目录不可配置。
- 环境变量：参见 `~/.profile`。
- 永远不要提交或发布真实的电话号码、视频或实时配置值。在文档、测试和示例中使用明显的假占位符。
- 发布流程：在任何发布工作之前，始终阅读 `docs/reference/RELEASING.md` 和 `docs/platforms/mac/release.md`；一旦这些文档回答了常规问题就不要询问。

## GHSA（仓库公告）补丁/发布

- 在审查安全公告之前，请阅读 `SECURITY.md`。
- 获取：`gh api /repos/openclaw/openclaw/security-advisories/<GHSA>`
- 最新 npm：`npm view openclaw version --userconfig "$(mktemp)"`
- 私有 fork PR 必须关闭：
  `fork=$(gh api /repos/openclaw/openclaw/security-advisories/<GHSA> | jq -r .private_fork.full_name)`
  `gh pr list -R "$fork" --state open`（必须为空）
- 描述换行坑：通过 heredoc 将 Markdown 写入 `/tmp/ghsa.desc.md`（不要使用 `"\\n"` 字符串）
- 通过 jq 构建补丁 JSON：`jq -n --rawfile desc /tmp/ghsa.desc.md '{summary,severity,description:$desc,vulnerabilities:[...]}' > /tmp/ghsa.patch.json`
- GHSA API 坑：无法在同一 PATCH 中设置 `severity` 和 `cvss_vector_string`；请分别调用。
- 补丁 + 发布：`gh api -X PATCH /repos/openclaw/openclaw/security-advisories/<GHSA> --input /tmp/ghsa.patch.json`（发布 = 包含 `"state":"published"`；无 `/publish` 端点）
- 如果发布失败（HTTP 422）：缺少 `severity`/`description`/`vulnerabilities[]`，或私有 fork 有开放 PR
- 验证：重新获取；确保 `state=published`，`published_at` 已设置；`jq -r .description | rg '\\\\n'` 返回空

## 故障排除

- 品牌迁移/遗留配置/服务警告：运行 `openclaw doctor`（参见 `docs/gateway/doctor.md`）。

## Agent 特定说明

- 词汇："makeup" = "mac app"。
- 永远不要编辑 `node_modules`（全局/Homebrew/npm/git 安装也会覆盖）。技能笔记放在 `tools.md` 或 `AGENTS.md` 中。
- 当在任何地方添加新的 `AGENTS.md` 时，同时添加指向它的 `CLAUDE.md` 符号链接（示例：`ln -s AGENTS.md CLAUDE.md`）。
- Signal："update fly" => `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` 然后 `fly machines restart e825232f34d058 -a flawd-bot`。
- 处理 GitHub Issue 或 PR 时，在任务末尾打印完整 URL。
- 回答问题时仅提供高置信度答案：在代码中验证；不要猜测。
- 永远不要更新 Carbon 依赖。
- 任何带有 `pnpm.patchedDependencies` 的依赖必须使用精确版本（无 `^`/`~`）。
- 修补依赖（pnpm patches、overrides 或 vendored 更改）需要明确批准；默认不要这样做。
- CLI 进度：使用 `src/cli/progress.ts`（`osc-progress` + `@clack/prompts` spinner）；不要手写 spinner/bar。
- 状态输出：保持表格 + ANSI 安全包装（`src/terminal/table.ts`）；`status --all` = 只读/可粘贴，`status --deep` = 探测。
- Gateway 目前仅作为菜单栏应用运行；没有安装单独的 LaunchAgent/helper label。通过 OpenClaw Mac 应用或 `scripts/restart-mac.sh` 重启；验证/杀死使用 `launchctl print gui/$UID | grep openclaw` 而不是假设固定 label。**在 macOS 上调试时，通过应用启动/停止 gateway，而不是临时 tmux 会话；在切换前杀死任何临时隧道。**
- macOS 日志：使用 `./scripts/clawlog.sh` 查询 OpenClaw 子系统的统一日志；它支持 follow/tail/category 筛选，需要密码less sudo 以使用 `/usr/bin/log`。
- 如果本地有共享 guardrails，请查看它们；否则遵循此仓库的指导。
- SwiftUI 状态管理（iOS/macOS）：优先使用 `Observation` 框架（`@Observable`、`@Bindable`）而不是 `ObservableObject`/`@StateObject`；除非兼容性需要，否则不要引入新的 `ObservableObject`，并在接触相关代码时迁移现有用法。
- 连接 provider：添加新连接时，更新每个 UI 表面和文档（macOS 应用、web UI、移动端（如适用）、入职/概述文档）并添加匹配的状态 + 配置表单，以使 provider 列表和设置保持同步。
- 版本位置：`package.json`（CLI）、`apps/android/app/build.gradle.kts`（versionName/versionCode）、`apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist`（CFBundleShortVersionString/CFBundleVersion）、`apps/macos/Sources/OpenClaw/Resources/Info.plist`（CFBundleShortVersionString/CFBundleVersion）、`docs/install/updating.md`（固定 npm 版本）、`docs/platforms/mac/release.md`（APP_VERSION/APP_BUILD 示例）、Peekaboo Xcode 项目/Info.plist（MARKETING_VERSION/CURRENT_PROJECT_VERSION）。
- "Bump version everywhere" 意味着上述所有版本位置**除了** `appcast.xml`（仅在切割新的 macOS Sparkle 发布时触摸 appcast）。
- **重启应用：** "restart iOS/Android apps" 意味着重新构建（重新编译/安装）并重新启动，而不仅仅是杀死/启动。
- **设备检查：** 测试前，验证已连接的真实设备（iOS/Android）后再使用模拟器/仿真器。
- iOS Team ID 查询：`security find-identity -p codesigning -v` → 使用 Apple Development (…) TEAMID。回退：`defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`。
- A2UI bundle hash：`src/canvas-host/a2ui/.bundle.hash` 是自动生成的；忽略意外更改，仅在需要时通过 `pnpm canvas:a2ui:bundle`（或 `scripts/bundle-a2ui.sh`）重新生成。将 hash 作为单独提交。
- 发布签名/notary 密钥在仓库外管理；遵循内部发布文档。
- Notary 认证环境变量（`APP_STORE_CONNECT_ISSUER_ID`、`APP_STORE_CONNECT_KEY_ID`、`APP_STORE_CONNECT_API_KEY_P8`）预期在你的环境中（根据内部发布文档）。
- **多 agent 安全：** 不要创建/应用/删除 `git stash` 条目，除非明确要求（包括 `git pull --rebase --autostash`）。假设其他 agent 可能正在工作；保持无关的 WIP 不变，避免跨切状态更改。
- **多 agent 安全：** 当用户说 "push" 时，你可以 `git pull --rebase` 整合最新更改（永不丢弃其他 agent 的工作）。当用户说 "commit" 时，仅限你的更改。当用户说 "commit all" 时，按分组块提交所有内容。
- **多 agent 安全：** 不要创建/删除/修改 `git worktree` 检查out（或编辑 `.worktrees/*`），除非明确要求。
- **多 agent 安全：** 不要切换分支/检出不同分支，除非明确要求。
- **多 agent 安全：** 多 agent 运行是可以的，只要每个 agent 有自己的会话。
- **多 agent 安全：** 当看到未识别的文件时，继续；专注于你的更改，仅提交那些。
- Lint/format 改动：
  - 如果 staged+unstaged diff 仅是格式化，自动解决，无需询问。
  - 如果已请求 commit/push，自动 stage 并在同一次提交中包含格式化-only 后续（或小后续提交如需要），无需额外确认。
  - 仅在更改是语义（逻辑/数据/行为）时才询问。
- Lobster 接缝：使用 `src/terminal/palette.ts` 中的共享 CLI 调色板（无硬编码颜色）；将调色板应用于入职/配置提示和其他 TTY UI 输出。
- **多 agent 安全：** 聚焦于你的编辑；除非真正受阻，否则避免 guard-rail 免责声明；当多个 agent 接触同一文件时，如果安全则继续；仅在相关时简要提及"存在其他文件"。
- Bug 调查：在下结论之前阅读相关 npm 依赖的源代码和所有相关本地代码；目标是高置信度的根本原因。
- 代码风格：为棘手逻辑添加简要注释；尽可能保持文件在约 500 行以下（根据需要拆分/重构）。
- 工具 schema guardrails（google-antigravity）：避免在工具输入 schema 中使用 `Type.Union`；不使用 `anyOf`/`oneOf`/`allOf`。对字符串列表使用 `stringEnum`/`optionalStringEnum`（Type.Unsafe enum），并使用 `Type.Optional(...)` 而非 `... | null`。保持顶层工具 schema 为 `type: "object"` 带 `properties`。
- 工具 schema guardrails：避免在工具 schema 中使用原始 `format` 属性名；某些验证器将 `format` 视为保留关键字并拒绝 schema。
- 当被要求打开 "session" 文件时，在 `~/.openclaw/agents/<agentId>/sessions/*.jsonl` 下打开 Pi 会话日志（使用系统提示中 Runtime 行的 `agent=<id>` 值；除非给出特定 ID，否则使用最新的），而不是默认的 `sessions.json`。如果需要从另一台机器获取日志，通过 Tailscale SSH 并在那里读取相同路径。
- 不要通过 SSH 重新构建 macOS 应用；重新构建必须直接在 Mac 上运行。
- 永远不要向外部消息表面（WhatsApp、Telegram）发送流式/部分回复；仅应交付最终回复。流式/工具事件仍可发送到内部 UI/控制通道。
- 语音唤醒转发提示：
  - 命令模板应保持 `openclaw-mac agent --message "${text}" --thinking low`；`VoiceWakeForwarder` 已经对 `${text}` 进行了 shell 转义。不要添加额外的引号。
  - launchd PATH 很小；确保应用的 launch agent PATH 包含标准系统路径加上你的 pnpm bin（通常为 `$HOME/Library/pnpm`），以便通过 `openclaw-mac` 调用时 `pnpm`/`openclaw` 二进制可以解析。
- 对于包含 `!` 的手动 `openclaw message send` 消息，使用下面注意的 heredoc 模式以避免 Bash 工具的转义。
- 发布 guardrail：未经操作员明确同意，不要更改版本号；在运行任何 npm publish/release 步骤之前始终请求许可。
- Beta 发布 guardrail：使用 beta Git 标签时（例如 `vYYYY.M.D-beta.N`），在 `--tag beta` 上发布匹配的 beta 版本后缀（例如 `YYYY.M.D-beta.N`），而不是普通版本；否则普通版本名称会被消耗/阻止。

## NPM + 1Password（发布/验证）

- 使用 1password 技能；所有 `op` 命令必须在新的 tmux 会话中运行。
- 登录：`eval "$(op signin --account my.1password.com)"`（应用已解锁 + 集成已开启）。
- OTP：`op read 'op://Private/Npmjs/one-time password?attribute=otp'`。
- 发布：`npm publish --access public --otp="<otp>"`（从包目录运行）。
- 无本地 npmrc 副作用验证：`npm view <pkg> version --userconfig "$(mktemp)"`。
- 发布后杀死 tmux 会话。

## 插件发布快速路径（无核心 `openclaw` 发布）

- 仅发布已存在于 npm 的插件。源列表在 `docs/reference/RELEASING.md` 的 "Current npm plugin list" 下。
- 在 tmux 内运行所有 CLI `op` 调用和 `npm publish` 以避免挂起/中断：
  - `tmux new -d -s release-plugins-$(date +%Y%m%d-%H%M%S)`
  - `eval "$(op signin --account my.1password.com)"`
- 1Password 助手：
  - `npm login` 使用的密码：
    `op item get Npmjs --format=json | jq -r '.fields[] | select(.id=="password").value'`
  - OTP：
    `op read 'op://Private/Npmjs/one-time password?attribute=otp'`
- 快速发布循环（本地辅助脚本在 `/tmp` 中是可以的；保持仓库干净）：
  - 比较本地插件 `version` 与 `npm view <name> version`
  - 仅在版本不同时运行 `npm publish --access public --otp="<otp>"`
  - 如果包在 npm 上缺失或版本已匹配则跳过。
- 保持 `openclaw` 不变：除非明确要求，否则永远不要从仓库根目录运行发布。
- 每个发布的发布后检查：
  - per-plugin：`npm view @openclaw/<name> version --userconfig "$(mktemp)"` 应该是 `2026.2.17`
  - 核心 guard：`npm view openclaw version --userconfig "$(mktemp)"` 应保持在前一个版本，除非明确要求。

## Changelog 发布说明

- 切割带 beta GitHub 预发布的 mac 发布时：
  - 从发布提交中创建标签 `vYYYY.M.D-beta.N`（示例：`v2026.2.15-beta.1`）。
  - 创建标题为 `openclaw YYYY.M.D-beta.N` 的预发布。
  - 使用 `CHANGELOG.md` 版本部分的发布说明（`Changes` + `Fixes`，无标题重复）。
  - 至少附加 `OpenClaw-YYYY.M.D.zip` 和 `OpenClaw-YYYY.M.D.dSYM.zip`；如有 `.dmg` 也包含。

- 保持 `CHANGELOG.md` 顶部版本条目按影响排序：
  - `### Changes` 在前。
  - `### Fixes` 去重，优先面向用户的修复。
- 标记/发布前运行：
  - `node --import tsx scripts/release-check.ts`
  - `pnpm release:check`
  - `pnpm test:install:smoke` 或非 root 烟雾路径的 `OPENCLAW_INSTALL_SMOKE_SKIP_NONROOT=1 pnpm test:install:smoke`。
