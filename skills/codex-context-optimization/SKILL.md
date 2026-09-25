---
name: codex-context-optimization
description: Optimize Codex CLI context-window usage, long-session context management, auto-compaction, and lazy App/MCP tool loading. Use when initial token usage is unexpectedly large, Apps/MCP tools consume too much context, tool_search/deferred loading needs enabling or verification, or experimental context rollover/token-budget behavior needs configuring or troubleshooting.
metadata:
  short-description: Optimize Codex context and lazy tools
---

# Codex Context Optimization

Use this skill when Codex spends a large fraction of the model context window before the first real task, or when tuning long-running sessions that use compaction and context rollover.

## Workflow

1. Measure the current first-turn input-token baseline with the intended model.
2. Inspect the effective model catalog with `codex debug models`; do not assume a capability is enabled because the backend model supports it.
3. Prefer deferred App/MCP tool loading over disabling useful integrations.
4. Treat experimental context management separately from tool deferral; either feature can be enabled and validated independently.
5. Make a backup before editing `config.toml` or a custom model catalog.
6. Re-run the same minimal prompt after each change and compare input tokens.
7. For tool deferral, perform one real App/MCP call and verify a `tool_search_call` precedes the tool call.
8. For experimental context management, verify the rollout contains `token_budget.context_window`; do not claim rollover is validated until a window actually changes.

Read [references/tutorial.md](references/tutorial.md) for the configuration, capability overrides, verification commands, measured example, rollback steps, and version caveats.

## Important boundaries

- `supports_search_tool` and `supports_experimental_context` are model-catalog capabilities, not ordinary `config.toml` feature flags.
- Custom catalog overrides are version-sensitive. Re-check them after Codex upgrades or model-catalog refreshes.
- Do not reduce functionality by disabling all Apps merely to save tokens when deferred tool loading works.
- Do not enable a capability only because its field exists. Validate one real request with the target model/provider.
- Experimental context management can change upstream; keep a rollback path.

