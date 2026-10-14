# RFC: Native Claude Code Runtime Inside Codex

**Status:** Implementation design; **no runtime integration is implemented by this document**.
**Branch:** `feature/native-claude-code-runtime`
**Target:** An updated fork of `openai/codex`. Do **not** implement against the older February 2026 baseline without syncing first.

## Goal

Integrate the **real, installed Claude Code CLI** inside the Codex Rust application in two distinct ways:

1. **First-class delegated agent:** Existing Codex sessions can spawn, monitor, message, resume, interrupt, and close a Claude Code worker with the existing multi-agent UX.
2. **Runtime-aware model picker:** The Codex TUI can choose between a Codex/Responses session and a *native Claude Code session*, with explicit labels identifying which harness is active.

A Claude model routed through a Responses proxy is **not** equivalent to this feature. The implementation must invoke Claude Code's actual programmatic runtime and preserve its independent sessions and permissions.

## Preconditions

- Sync this fork with recent upstream `openai/codex` on a separate update branch. The current fork HEAD (`f0a85ded1867072d5b4963b8be3be1ef4f216549`) was dated 2026-02-26, and its source layout differs from current upstream.
- Verify the installed `claude` CLI, exact version, authentication method, and permitted deployment conditions.
- Use a developer test repository with **no customer documents or credentials**.
- Add an opt-in feature flag; leave this experimental backend disabled by default.
- Keep the stock Codex binary available. Build and test the fork separately rather than replacing it on day one.

## Proposed architecture

```text
Codex CLI / TUI / app-server
  |
  +-- Runtime-aware session router ---------------------------+
  |                                                          |
  +-- CodexRuntime (existing Responses engine)               |
  |                                                          |
  +-- ClaudeCodeRuntime (new subprocess/agent adapter) -------+
       | bounded, sandboxed, permission-aware native worker
       +-- installed Claude Code CLI
             +-- native conversation/session ID
             +-- native Claude Code tools/subagents
```

Shared Codex IDs identify the user-facing session/child. Separate metadata retains `runtime_kind`, `native_session_id`, selected model, workspace/worktree, permissions, start time, status, and associated process. Never represent a Claude session as a Codex Responses session.

### Core runtime interface (conceptual)

```rust
enum AgentBackendKind {
    Codex,
    ClaudeCode,
}

struct RuntimeSelection {
    backend: AgentBackendKind,
    model: String,
}

trait NativeAgentRuntime {
    // Async API; real signatures to follow the up-to-date Codex conventions.
    // start, send, poll/stream, interrupt, resume, close, list_models
}
```

This is an architectural contract, **not compilable Rust implementation**.

### Claude Code process transport

- Use `tokio::process::Command`, passing arguments separately (never shell-concatenate prompts).
- Start via documented Claude Code headless/print interface, using structured `stream-json` output.
- Parse each structured stdout message with size bounds, type validation, and deterministic event mapping.
- Support native session continuation by recording the returned Claude Code session ID and using documented resume semantics.
- Map native events into existing Codex progress/result/error UI surfaces. Retain a bounded transcript for diagnostics.
- Keep stdout/stderr and exit status distinct. Surface auth, missing binary, invalid model, permission denial, crash, and timeout errors explicitly.
- On interrupt or timeout, terminate the process **and its child process group** where supported, then reconcile worker state.
- Do not record API keys, bearer tokens, secret-bearing environment variables, or the raw authentication store.

### Permissions and sandboxing

**Critical:** Launching `claude` directly from an unrestricted host subprocess is unacceptable if it bypasses Codex's approval/sandbox checks.

- Route worker execution through an existing **enforced** sandbox boundary or an equivalent restricted container/worktree.
- Default to read-only / plan mode.
- Preserve the applicable parent approval limits; a delegated worker cannot obtain more access than the parent.
- Do not use `--dangerously-skip-permissions` or any implicit allow-all mode.
- Allowlist the executable path, models, workspaces, argument types, and environment variables.
- Deny unrestricted filesystem access, host secrets, networking, deployment credentials, and customer data until explicitly authorized under applicable policy.
- Require explicit action approval for changes that exceed the current sandbox grant.
- Limit subprocess count, output bytes, running time, concurrency, and nesting depth.
- For parallel writers, allocate separate Git worktrees and reconcile diffs rather than sharing one mutable checkout.

## Subagent integration (deliver first)

On the **latest upstream source**, investigate these modules first:

- `codex-rs/core/src/agent/child_config.rs` — existing spawn configuration and runtime inheritance.
- `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs` — existing native agent spawn.
- `codex-rs/core/src/tools/handlers/multi_agents_spec.rs` — model-visible spawn schema.
- AgentControl and agent lifecycle modules used by the above — status, send, wait, interrupt, resume, close.
- Appropriate protocol/TUI status structs — accurate runtime/model attribution.

Propose an optional, validated `runtime` parameter to `spawn_agent`, defaulting to `codex`. A new value `claude_code` is accepted only when the feature is enabled and the workspace policy authorizes it. Model selection must be constrained to available Claude Code aliases/IDs.

**Proposed UX, not currently supported syntax:**

```json
{
  "task_name": "security_review",
  "runtime": "claude_code",
  "model": "sonnet",
  "message": "Review authentication logic read-only. Return paths and line references."
}
```

All existing lifecycle commands must operate on either runtime without ambiguity. Preserve a distinct backend identity in status and completion events. When delegating across runtimes, pass an **explicit, bounded plaintext task brief**. Do not forward a Codex model's encrypted/context-bound history blocks to Claude Code or vice versa. Cross-runtime `fork_turns=all` is not supported unless an explicitly designed safe transcript conversion is implemented.

### Two important upstream issues to test against

- `openai/codex#40858`: native child ignores an explicit `model_provider` override.
- `openai/codex#37237`: non-OpenAI child receives encrypted task content it cannot decode.

These are distinct from the Claude CLI adapter but underscore why cross-provider/session payloads must not be blindly forwarded.

## Runtime-aware picker (second deliverable)

On the **latest upstream source**, inspect:

- `codex-rs/models-manager` and `codex-rs/model-provider` for model catalog and provider metadata.
- `codex-rs/tui/src/app_event.rs` and TUI model-selection handling for the user interface.
- Codex `app-server` session/model APIs to ensure clients do not misidentify Claude sessions as Codex models.

**Proposed UI:**

```text
/model
  Codex          GPT model selected by current Codex account
  Claude Code    Sonnet (native CLI)
  Claude Code    Opus (native CLI)
```

A Claude Code choice must **switch session runtime**, not merely change `config.model` to a Claude model name and continue calling the Responses endpoint. Make the switch explicit when there is an active thread; preserve original thread state without silently migrating unconvertible context. Show a runtime indicator in status, transcript, and session picker.

Model discovery must reflect the actually installed CLI's supported selection behaviour and account entitlements. Never assume a model alias means the account can run it.

## Proposed user configuration (not valid Codex syntax until implemented)

```toml
[features]
native_claude_code = true

[agent_runtimes.claude_code]
command = "claude"
max_concurrent = 2
default_permission_mode = "plan"

[agents.claude_reviewer]
runtime = "claude_code"
model = "sonnet"
description = "Read-only native Claude Code reviewer"
```

The schema must explicitly reject unrecognized runtime fields/model IDs when enabled. Credential provisioning is external to this config and must follow the Claude CLI's supported authentication methods and licensing.

## Acceptance criteria

1. Existing Codex-only sessions work unchanged with the feature disabled.
2. `spawn_agent(runtime=claude_code,...)` starts the actual installed Claude Code binary, not an Anthropic API proxy or mock provider.
3. A Claude worker can complete a read-only review with streamed progress; Codex receives a stable agent ID, result, file references, and accurate model/runtime label.
4. `send_message`, `wait_agent`, `interrupt_agent`, and `close_agent` work for this backend; native session resume works after process exit.
5. A nonexistent CLI, unauthorized account, bad model, crash, timeout, malformed structured event, and denied tool permission all return understandable failures.
6. A delegated worker cannot bypass sandbox/approval rules, even with an adversarial prompt.
7. Multiple workers use isolated worktrees; cancellation does not leave child processes behind.
8. A Claude Code picker selection starts a native Claude session; switching back preserves the independent Codex session.
9. The TUI and app-server do not imply model compatibility, context transfer, or authentication that is not available.
10. Relevant unit, integration, Rust formatting/linting, and TUI tests pass on Linux and Windows; manual tests cover supported Claude Code versions.

## Phased development

- **Phase 0:** Update fork from upstream; inventory relevant current APIs; preserve an untouched build; obtain a local tested runtime.
- **Phase 1:** Implement feature-gated Claude CLI runtime adapter, fake-CLI fixtures, typed event parser, session tracking, cancellation and permission policy.
- **Phase 2:** Route `spawn_agent` and all lifecycle commands by runtime. Add isolated-worktree and cross-runtime payload tests.
- **Phase 3:** Add runtime-aware picker, session controls, status indicators and app-server projection.
- **Phase 4:** Test installed native Claude CLI and end-to-end workflows; security review, CI, packaging and staged rollout.

## References

- Official Codex source: https://github.com/openai/codex
- Codex source current child config: https://github.com/openai/codex/blob/main/codex-rs/core/src/agent/child_config.rs
- Codex current multi-agent spawn: https://github.com/openai/codex/blob/main/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs
- Codex model-provider abstraction: https://github.com/openai/codex/blob/main/codex-rs/model-provider/src/provider.rs
- Claude Code headless documentation: https://code.claude.com/docs/en/headless
- Claude Code CLI reference: https://code.claude.com/docs/en/cli-reference
- Cross-provider issue: https://github.com/openai/codex/issues/40858
- Encrypted payload issue: https://github.com/openai/codex/issues/37237

This branch contains the engineering brief **only**. No source code, installed Codex binary, production environment or authentication settings are changed by this RFC.
