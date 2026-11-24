# pi-claude-code

## About this fork

This is a fork of [earendil-works/pi-claude-code](https://github.com/earendil-works/pi-claude-code). It carries upstream pull requests that were open and unmerged when it was made, plus one fix of its own. The changes live on the `fix/claude-code-harness-parity` branch, on top of upstream `main` at `d5b5e04`. It exists to stop the `claude-code` provider running slower and producing more output than the same model through Pi's `anthropic` provider.

The upstream pull requests were cherry-picked with `git cherry-pick -x` in this order, so each commit keeps its original author and names the commit it came from:

1. [#51](https://github.com/earendil-works/pi-claude-code/pull/51) tags replayed assistant turns with their model. Without it, Claude Code strips the earlier turns' thinking, so the model works its reasoning out again on every step ([#52](https://github.com/earendil-works/pi-claude-code/issues/52)).
2. [#57](https://github.com/earendil-works/pi-claude-code/pull/57) omits the earlier turns' effort setting when the fallback system snapshot is replayed.
3. [#65](https://github.com/earendil-works/pi-claude-code/pull/65) routes a model by its exact ID, so a model is not run through a family alias that now points at a different model.
4. [#66](https://github.com/earendil-works/pi-claude-code/pull/66) adds the beta header that the per-message effort setting requires.
5. [#69](https://github.com/earendil-works/pi-claude-code/pull/69) passes the model's output limit to Claude Code, which otherwise caps output at 32K and can spend all of it on thinking.
6. [#39](https://github.com/earendil-works/pi-claude-code/pull/39) answers tool calls left without a result by an aborted or failed turn. Without it, every later request in that session fails with `tool_use ids were found without tool_result blocks` ([#49](https://github.com/earendil-works/pi-claude-code/issues/49)).

Order matters only where two pull requests touch the same files. #51 and #57 both change `src/claude-cli.ts`, and #51 and #69 both change `src/stream.ts`. Each pull request applies cleanly to upstream `main` on its own, and the sequence above applies without conflicts. No other order has been tested.

The fork's own commit fixes a problem no upstream issue or pull request covers. Claude Code can serve a model before Pi's bundled Anthropic catalog lists it. Fable 5.2 was one: it matched no built-in model and fell back to a 200K context window and 64K output limit, so Pi compacted it at about 190K tokens when the API accepts 1M. An undated model missing from Pi's catalog now takes the context window and output limit of the newest older model in its family. Pricing is not inferred, and a model with no older model in its family keeps 200K and 64K.

The tests run on Node 24. Node 26 rejects the `--experimental-transform-types` flag that `npm test` passes.

The rest of this README is upstream's.

## Overview

This Pi extension lets you use your Claude Pro or Max subscription in Pi by
routing all Claude model requests through the official Claude Code CLI.

Pi still handles conversation history, approvals, compaction, and tool
execution. Claude Code only talks to the model. It never runs tools itself.

> [!WARNING]
> **The future of this extension is uncertain.** Right now it works by running
> Pi's requests through the official Claude Code CLI in non-interactive mode
> (`claude -p`), and that usage counts against your normal Claude subscription
> limits. In May 2026 Anthropic
> [announced](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
> that `claude -p`, Agent SDK, and third-party usage would move to a separate,
> smaller "Agent SDK credit" billed at API rates. Anthropic has since put that
> change on hold and says it will give advance notice before any future change.
>
> Until then the extension keeps working as described here, but that could
> change at any time. If Anthropic changes how `claude -p` usage is billed, this
> extension may cost more, stop drawing from your subscription, or stop working
> altogether. See Zed's
> [write-up](https://zed.dev/blog/anthropic-subscription-changes) for more
> background.

## Requirements

- Pi
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code), installed and
  on your `PATH`:

  ```sh
  npm install -g @anthropic-ai/claude-code
  ```

- A Claude subscription that you are logged into with Claude Code

## Installation

```sh
pi install https://github.com/earendil-works/pi-claude-code
```

Then pick a model from the **Claude Code Subscription (Experimental)** provider
in Pi.

## Login and logout

Claude Code handles authentication. Log in and out with the Claude Code CLI:

```sh
claude auth login
claude auth logout
```

The extension uses whatever login Claude Code currently has. You don't need
Pi's `/login` or `/logout` for this provider. The extension doesn't read, copy,
or store your Claude Code credentials.

Run `/claude-code-status` in Pi to check that Claude Code is installed and
logged in.

## Configuration

These environment variables are optional:

- `PI_CLAUDE_CODE_COMMAND`: path or name of the Claude Code executable
  (default: `claude`).
- `PI_CLAUDE_CODE_CONFIG_DIR`: Claude Code config directory to use. The
  extension passes it to Claude Code as `CLAUDE_CONFIG_DIR`.
- `PI_CLAUDE_CODE_POWERSHELL`: PowerShell executable used to run a `.ps1`
  `PI_CLAUDE_CODE_COMMAND` (default: `pwsh` when installed, otherwise
  `powershell.exe`).

### Windows

npm installs Claude Code as `claude.cmd` (and a `claude.ps1` for PowerShell).
Both forms work as `PI_CLAUDE_CODE_COMMAND`:

- `.cmd` and `.bat` launchers run through `cmd.exe` with their arguments
  escaped once and passed verbatim, so installation paths with spaces and
  arguments containing quotes or `&` reach Claude Code unchanged.
- `.ps1` launchers run in a noninteractive PowerShell session with
  `-NoProfile` and `-ExecutionPolicy Bypass`. The extension prefers `pwsh`
  (PowerShell 7) because Windows PowerShell 5.1 cannot forward every quoted
  argument from a script to a native program.

### Model discovery and privacy

The extension asks Claude Code for the account's model catalog automatically.
The discovery probe allows Claude Code's normal feature-flag and catalog traffic,
including its usage telemetry, because disabling feature flags hides models such
as Mythos and previous-generation models. This is a
[known Claude Code issue](https://github.com/anthropics/claude-code/issues/105923).
The probe sends no conversation history or generation request. Automatic CLI
updates, official marketplace auto-install, feedback uploads, and error reporting
remain disabled.

To opt out of discovery telemetry, set `DISABLE_TELEMETRY=1` before starting Pi.
Discovery also preserves inherited `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`,
`DO_NOT_TRACK`, and `DISABLE_GROWTHBOOK` settings. These can reduce the discovered
catalog; the extension does not remove them to restore missing models. Claude
Code treats any non-empty `DISABLE_TELEMETRY` or
`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` value as enabled, including `0` or
`false`; unset them to allow normal discovery again. See
[Claude Code's environment variables](https://code.claude.com/docs/en/env-vars#features-that-need-feature-flag-fetching).

Conversation workers still disable telemetry and nonessential traffic regardless
of the discovery settings.

## Usage and billing

Requests made through this extension count against your Claude subscription's
usage quota and rate limits, the same as any other Claude Code usage. The
extension does not give you separate or unlimited usage.

The extension ignores `ANTHROPIC_API_KEY` in Claude Code child processes without
changing your environment. To avoid silently changing how you're authenticated
or billed, it still rejects auth-token, proxy, Bedrock, Vertex, or Foundry
overrides for Claude Code.

## Limitations

- The extension is experimental.
- Branching, editing earlier messages, or changing settings mid-conversation
  starts a fresh Claude Code process that replays the conversation. The first
  response after that can be slower and may use more quota.
- Complex tool schemas (nullable types, `oneOf`, `allOf`, `anyOf`) may be
  simplified before Claude sees them.
- On Windows, Windows PowerShell 5.1 mangles quoted arguments (such as the
  inline MCP configuration JSON) when a `.ps1` launcher forwards them to a
  native program. Install PowerShell 7 or use the `claude.cmd` launcher in that
  case.
- Adding or removing tools in the middle of a conversation keeps the prompt
  cache only on supported models. On other models it may use more quota.

For details on how the extension works internally, see
[ARCHITECTURE.md](ARCHITECTURE.md).
