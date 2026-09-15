# Desktop 契约与验证

本插件基于 [anywhere-labs/dsh-desktop](https://github.com/anywhere-labs/dsh-desktop) 的插件规范编写，落地文档是该仓库的 [`docs/plugin-development.md`](https://github.com/anywhere-labs/dsh-desktop/blob/master/docs/plugin-development.md) 与包级 service 合同 [`dsh-plugin-desktop/docs/plugin-services.md`](https://github.com/anywhere-labs/dsh-desktop/blob/master/dsh-plugin-desktop/docs/plugin-services.md)。

## 采用的接口

| 接口 | 用途 | 说明 |
| --- | --- | --- |
| `desktopProfiles.current` | 活动 profile 的名称与绝对目录 | 一代之内不变；插件不从 argv、`ctx.baseUrl`、settings 或 `$DSH_HOME` 推断 profile |
| `desktopProfiles.list()` | `status` 之外的可选只读信息 | 只读清单，不改 patch、依赖或 bundle 顺序 |
| `desktopPnpm.runPlugin(argv, invokingDir, signal)` | 执行打包的 `dsh plugin --profile <active>` | 参数按 argv 传递，不拼 shell 字符串；`invokingDir` 用活动 profile 目录 |
| `ctx.get('sandboxPolicy')` | 只读会话判定 | 只读会话一律拒绝变更 |
| `ctx.get('approval')` | 变更前的显式确认 | 工具调用且带 agent 时必须 `allowed-once` |

`desktopPnpm` 不提供安装事务、快照、回滚或重试；Desktop 自己的健康启动检查点负责恢复，因此本插件不假装提供事务语义：它报告实际退出码、终止信号与有界输出。

## 规范检查表对应

| 规范要求 | 实现 |
| --- | --- |
| 变更必须来自显式用户或管理员动作 | `install`/`update` 走 approval，或由用户直接输入命令；只读会话直接拒绝 |
| 不跨代保留 service 引用 | 每次动作都重新读 `ctx.desktopProfiles.current`，不缓存 profile 或句柄 |
| 优先显式 pnpm argv，必要时用插件适配器 | 只用 `runPlugin()`，因为本插件的职责正是 DSH bundle 收敛 |
| 操作完成后校验领域状态 | 操作结束后重新读取 profile，结果放在 `verified` |
| 自备超时并保留句柄以便取消 | `AbortSignal.timeout(timeoutMs)` 与 `AbortSignal.any()` 合并调用方信号；句柄保留在操作器内 |
| 持续读取输出并有界保存 | 两条流都持续读取，只保留尾部 `maxOutputChars` 字符 |
| 分别处理拒绝、非零退出与终止信号 | `done` 拒绝映射为 `DESKTOP_SUITE_SPAWN_FAILED`；`exitCode` 与 `signal` 分开上报，`ok` 要求两者分别为 0 与 `null` |
| 不并发修改 profile | 服务本身每代仅允许一个操作，插件再在本地拒绝第二个调用 |
| 卸载时取消并等待 | `ctx.effect` 的清理函数调用 `dispose()`：先取消，再等待 `done` |

## 实机验证

桌面实例上验证安装与更新路径（需要 DSH Desktop 正在运行，且 profile patch 已启用本插件）：

1. 把仓库 `link:` 进活动 profile，并在 profile 的 `dsh.profile.bundles` 中加入 `dsh-desktop-suite`，然后重启 DSH Desktop。
2. 在会话里执行 `/desktop-suite status`：应列出活动 profile 的名称与目录，以及每个成员的声明目标与已安装版本。
3. 执行 `/desktop-suite install`：只对尚未声明的成员发起一次 `dsh plugin add`，结束后 `verified` 里对应成员的 `declared` 应为 `true`。
4. 在 profile 目录里核对 `package.json` 的依赖行与 `dsh.profile.bundles` 由打包 CLI 写入，而不是本插件写入。
5. 触发一次失败路径（例如指向不存在版本的目标），确认结果是 `ok: false` 且带真实 `exitCode`，而不是抛出不透明错误。

单元测试不覆盖真实 pnpm 与 Desktop 服务：`npm test` 用假 ctx 与假句柄覆盖计划、门控、结果映射与清理路径。
