# Codex context optimization tutorial

This guide covers two independent optimizations:

1. **Experimental long-session context management** — token-budget metadata, context-window identity, rollover support, and the newer auto-compaction accounting mode.
2. **Deferred App/MCP tool loading** — keep integrations available without injecting hundreds of tool schemas into the first model request.

The examples were validated against Codex CLI `0.158.0-alpha.8` on 2026-09-25. Both areas are evolving. Always verify the effective behavior of the installed Codex version.

## 1. Establish a baseline

Use the same model and a minimal prompt before and after changes:

```bash
codex exec --model gpt-6-luna --json 'Reply exactly: ok' </dev/null
```

Read the final `turn.completed.usage.input_tokens`. Use a fresh session so a resumed conversation does not distort the comparison.

Also record:

```bash
codex --version
codex features list
codex debug models
```

The important distinction is:

- `config.toml` controls feature activation and policy.
- the model catalog controls per-model capabilities such as `supports_search_tool` and `supports_experimental_context`.

## 2. Enable the new long-session context mechanism

### 2.1 Use body-after-prefix auto-compaction accounting

Add this top-level setting:

```toml
model_auto_compact_token_limit_scope = "body_after_prefix"
```

The older `total` accounting includes the retained prefix from the previous compacted state when deciding when to compact again. `body_after_prefix` measures growth after the current window prefix, reducing repeated early compaction in long sessions.

This does **not** bypass the model's hard context limit.

### 2.2 Enable experimental context management and token budget

```toml
[features.context_management]
experimental_mode = true

[features.token_budget]
enabled = true
```

Verify:

```bash
codex features list | grep -E '^(context_management|token_budget)'
```

Expected state:

```text
context_management  ...  true
token_budget        ...  true
```

These features may still be marked `under development`.

### 2.3 Enable the model capability

If the effective catalog reports:

```text
supports_experimental_context = false
```

the feature may remain gated even though the feature flags are enabled.

For a custom `model_catalog.json`, set the target model entry to:

```json
{
  "supports_experimental_context": true
}
```

Do this only for models you will actually validate. If several local aliases point to the same backend capability, update each intended alias consistently.

### 2.4 Verify runtime activation

Run a real short session, locate its rollout under the Codex sessions directory, and inspect the records for:

```text
token_budget.context_window
```

A working session should contain a developer item similar to:

```xml
<context_window>
Agent name: ...
First context window id: ...
Current context window id: ...
</context_window>
```

This proves the context-window/token-budget metadata path is active.

It does **not** prove rollover. To validate rollover, a sufficiently long session must eventually show `Current context window id` changing and should preserve the intended state across the transition.

### 2.5 Context-window size is a separate choice

The catalog's `context_window` remains independent from the mechanism above. For example:

```json
{
  "context_window": 383000,
  "effective_context_window_percent": 95
}
```

gives an effective model window of:

```text
383000 * 0.95 = 363850 tokens
```

Do not copy `383000` blindly; choose a value appropriate for the backend model and latency/cost goals.

## 3. Enable deferred App/MCP tool loading

### Why this matters

When Codex Apps exposes every connected tool directly, the first request can contain hundreds of function schemas even if the user only wants one tool.

In one measured environment, the Apps catalog contained:

```text
21 connectors
431 tools
```

including large namespaces such as GitHub, Google Drive, Notion, Sites, Vercel, Supabase, and AgentDock.

With all those schemas exposed directly, a minimal first turn used about:

```text
120,364 input tokens
```

After enabling the search/deferred path on GPT-6 Luna, the same prompt used:

```text
18,419 input tokens
```

That is about an 84.7% reduction in first-turn input tokens in that environment.

GPT-5.6 Luna showed the same behavior, with a minimal first turn around 19k tokens after the capability was enabled.

These numbers are an example, not a guarantee. The exact savings depend on connected Apps, skill descriptions, developer instructions, memory, and the selected model.

### 3.1 Keep Apps enabled

Do not start by disabling Apps globally. The goal is to keep them callable while deferring their schemas.

Useful feature checks:

```bash
codex features list | grep -E '^(apps|plugins|tool_suggest)'
```

`tool_suggest` should remain available. Apps and plugins can remain enabled as needed.

### 3.2 Enable search-tool capability in the model catalog

Check the target model:

```bash
codex debug models
```

If its effective entry reports:

```text
supports_search_tool = false
```

and the backend/model supports the tool-search path, set the custom catalog entry to:

```json
{
  "supports_search_tool": true
}
```

The crucial behavior is not the boolean itself; it is the runtime result. After enabling it, the model should use tool search and receive selected namespaces/functions with `defer_loading: true` rather than receiving every App tool schema in the initial request.

### 3.3 Verify token savings

Repeat the same minimal prompt:

```bash
codex exec --model gpt-6-luna --json 'Reply exactly: ok' </dev/null
```

Compare `turn.completed.usage.input_tokens` against the baseline.

If the count does not fall materially, inspect:

- whether the intended model alias actually has `supports_search_tool = true`;
- whether the session is fresh rather than resumed;
- whether a different tool surface is still being injected directly;
- whether the provider accepts the search/deferred tool protocol used by the model.

### 3.4 Verify a real deferred tool call

Token reduction alone is insufficient. Ask for a real App operation, for example a read-only GitHub lookup:

```text
Use the GitHub app/tool only to read a repository and return one metadata field.
```

In the rollout, verify the sequence:

```text
tool_search_call
  -> tool_search_output
     -> selected namespace/function with defer_loading=true
        -> function_call / MCP tool call
```

A validated GitHub example followed:

```text
tool_search_call
  -> mcp__codex_apps__github
     -> github.get_repo / github.fetch
```

This proves the App remains usable while its full tool schema is omitted from the first request.

## 4. Optional additional trimming

### Disable genuinely unused plugins

If a plugin is not useful in the current environment, disable it explicitly:

```toml
[plugins."example@marketplace"]
enabled = false
```

Do not expect this to produce the same savings as deferred Apps when the main cost comes from the global Apps tool surface.

### Remove duplicate skill catalog entries

Codex does not load every `SKILL.md` body at startup. It normally exposes skill name, description, and path, then loads the body when selected.

Duplicate skill registrations still waste discovery context. If two roots contain byte-identical copies, prefer path-based disablement of the duplicate instead of deleting files that another application resynchronizes:

```toml
[[skills.config]]
path = "/absolute/path/to/duplicate/SKILL.md"
enabled = false
```

Keep one canonical copy visible.

## 5. Recommended combined configuration

The ordinary configuration portion can look like:

```toml
model_auto_compact_token_limit_scope = "body_after_prefix"

[features.context_management]
experimental_mode = true

[features.token_budget]
enabled = true
```

Then, in the custom model catalog for each validated target model:

```json
{
  "supports_experimental_context": true,
  "supports_search_tool": true
}
```

The catalog fields are deliberately shown separately because they are model metadata, not normal `config.toml` keys.

## 6. Rollback

Always keep backups before editing:

```bash
cp -a ~/.codex/config.toml ~/.codex/config.toml.bak
cp -a ~/.codex/model_catalog.json ~/.codex/model_catalog.json.bak
```

To roll back the experimental context mechanism:

- remove or disable `features.context_management`;
- remove or disable `features.token_budget`;
- restore the previous `model_auto_compact_token_limit_scope` if desired;
- restore `supports_experimental_context` in the catalog.

To roll back deferred tool loading:

- restore the original `supports_search_tool` value for the affected model aliases.

After rollback, start a fresh session and re-run the baseline.

## 7. Upgrade checklist

After any Codex CLI or model-catalog update:

1. Run `codex --version`.
2. Run `codex features list`.
3. Run `codex debug models`.
4. Confirm the intended model still reports the two capability flags.
5. Run the minimal first-turn A/B check.
6. Make one real deferred App/MCP call.
7. For context management, verify `token_budget.context_window` still appears.
8. Remove local overrides that upstream now supplies correctly.

## 8. Upstream implementation references

Useful source locations in the OpenAI Codex repository:

- `codex-rs/core/src/session/context_window.rs` — context-window accounting.
- `codex-rs/features/src/feature_configs.rs` — feature configuration.
- `codex-rs/core/src/tools/handlers/tool_search.rs` — tool-search handling.
- `codex-rs/core/src/mcp_tool_exposure_test.rs` — direct versus deferred MCP/App exposure behavior.
- `codex-rs/core/src/connectors.rs` — connector discovery and tool-suggest integration.
- `codex-rs/core/config.schema.json` — current configuration schema.

Prefer the installed version's behavior over assumptions from a newer `main` branch.

