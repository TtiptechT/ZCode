# Spec: PreCompact Hook

## 行为

新增 Hook 事件 `PreCompact`，在一次上下文压缩真正开始之前触发，对齐 Claude Code 的 PreCompact 语义并适配 ZCode 的 compact 架构。

- 触发点：`compactActiveConversationImpl`（core/src/runtime/methods/compact-active.ts）在 `hasEnoughRuntimeEntriesToCompact` 判定通过之后、`CompactStarted` timeline 事件持久化之前。覆盖全部 trigger（manual / auto / partial / reactive / session_memory），单一挂点。
- 历史不足（`outcome: "skipped"` 健康路径）不触发 PreCompact——没有将要发生的压缩就没有 Pre- 事件。
- matcher：匹配 `trigger` 值（如 `manual`、`auto`、`auto|reactive`），与 SessionStart 按 `source` 匹配同构。
- stdin（Claude 兼容层，`createCompatibleHookStdin`）：
  - 基础字段照旧（session_id、transcript_path、cwd、permission_mode 等）；
  - 新增 `trigger` 与 `custom_instructions`（snake_case 兼容别名），camelCase 主契约字段 `customInstructions` 保留。

## 输入 / 输出契约

- `PreCompactHookInput extends BaseHookInput`：`trigger: CompactTrigger`、`customInstructions?: string`。
- `HookSpecificOutput` 增加 PreCompact 变体：`{ additionalContext?: string }`。
- 输出语义：
  - `additionalContext`（含顶层 `additionalContext` / `additional_context`）：聚合后并入 compact summary prompt 的 Additional Instructions 块（`buildCompactPrompt` 输入），多个 hook 上下文按 `#n` 编号拼接；
  - `decision: "block"` 或 `continue: false`：跳过本次压缩——持久化 `CompactTimelineStatus.Skipped`（`reason: "pre_compact_hook_block"`），返回 `outcome: "skipped"`；不中断外层 turn（`shouldPreventContinuation` 对 PreCompact 返回 false），auto/reactive 链路按 skipped 处理。

## 所有权与失败语义

- Hook 执行仍由 `AgentRuntimeInternal.hookRunner` 单点拥有；compact 流程只消费 `HookRunResult`，不新建第二条执行路径。
- Hook 执行失败 / 超时：不阻断压缩（压缩是恢复手段），生命周期事件由 runner 记录，compact 侧仅记 warn 日志并继续。
- `stopReason`（block 时）进入 skipped displayText，便于用户理解为何未压缩。

## 枚举同步清单（新增事件必须同步）

| 层     | 文件                                                                                                                       |
| ------ | -------------------------------------------------------------------------------------------------------------------------- |
| 契约   | `apps/zcode-cli/packages/contracts/src/hooks/index.ts`（HookEventName、HookInput、HookSpecificOutput、schema×2）           |
| 契约   | `apps/zcode-cli/packages/contracts/src/hooks/workspace-hook-trust.ts`（canonicalize events、workspaceHookEventNameSchema） |
| 配置   | `apps/zcode-cli/packages/adapters/src/config/schema.ts`（hooksSchema.events）                                              |
| 共享   | `packages/shared/src/hooks.ts`（HookEvent union）                                                                          |
| 共享   | `packages/shared/src/workspace-hook-config.ts`（WORKSPACE_HOOK_EVENT_NAMES、schema）                                       |
| 共享   | `packages/shared/src/workspace-hook-trust-store-file.ts`                                                                   |
| 协议   | `packages/shared/src/zcode-protocol-v4/workspace-hook-review.ts`、`rows.ts`（hookInvocationRowSchema）                     |
| 服务   | `packages/services/src/hooks/hooksService.ts`（HOOK_EVENTS）                                                               |
| UI     | `packages/ui/src/settings/HookForm.tsx`（HOOK_EVENTS）                                                                     |
| 运行时 | `core/src/hooks/configured-runner-input.ts`（stdin 兼容）、`output.ts`（specific output）                                  |
| 运行时 | `core/src/runtime/methods/hooks.ts` + `internal-hook-methods.ts` + `methods/index.ts`（runPreCompactHooks 挂载）           |
| 触发   | `core/src/runtime/methods/compact-active.ts`                                                                               |

## 边界

- Microcompact（工具结果局部清理）不触发 PreCompact：它不是上下文压缩，语义独立。
- 投影层 lane：PreCompact 归入 `assistantWork`（`hookInvocationLane` 默认分支），无需新 lane。
- `CompactFailureReason.HookFailed` 暂不生产：本 spec 中 hook 失败不使 compact 失败。
