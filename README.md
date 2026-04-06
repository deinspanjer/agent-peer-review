# Agent Peer Review

A Claude Code plugin marketplace for AI-to-AI peer validation. Multiple perspectives catch more issues than one.

## Available Plugins

### codex-peer-review

Symmetric two-AI peer review using OpenAI Codex CLI. Both AIs review the same scope **independently** in a blind pass, then debate per-issue with terminal states until convergence. Catches significantly more issues than single-pass validation.

## Installation

```bash
# Add this marketplace
/plugin marketplace add jcputney/agent-peer-review

# Install the Codex peer review plugin
/plugin install codex-peer-review
```

### Prerequisites for codex-peer-review

**Account Requirements:**
- **Claude Code**: Requires a Claude Pro ($20/mo), Max, Team Premium, or Enterprise subscription
- **OpenAI Codex CLI**: Included with ChatGPT Plus/Pro/Business/Edu/Enterprise plans, or use OpenAI API credits

**Install dependencies:**

```bash
# Codex CLI (0.118.0+ required)
npm i -g @openai/codex
codex login

# jq (required — used to parse Codex JSONL output)
brew install jq   # macOS
# apt install jq  # Debian/Ubuntu
```

**Initialize Codex profiles** (one-time):

```bash
/codex-peer-review init
```

This writes `[profiles.peer-review]` and `[profiles.peer-review-summarizer]` to `~/.codex/config.toml`. Tune the models there if you want to use a different reasoning level or model family.

## How Peer Review Works

The default mode is **blind-debate**:

1. **Round 0 — Blind pass.** Claude and Codex independently review the same scope with the same prompt. Neither sees the other's findings.
2. **Canonicalize.** Findings are merged by content-hashed ID (`sha1(file + claim)`). Duplicates collapse. Findings reported by both AIs get a high-confidence flag. Style issues are dropped from the debate.
3. **Round 1+ — Per-issue debate.** Each issue has a state (`proposed → accepted | rejected | merged | escalated | deferred`). Both sides emit per-issue stances (`accept`, `concede`, `defend`, `dismiss`). Transitions are deterministic.
4. **Convergence is derived**, not declared, from the issue table — no LLM judging another LLM.
5. **Verdict synthesis** categorizes issues by terminal state: Critical, Important, Contested, Dismissed, Style notes.

When the AIs flag a **security**, **architecture**, or **breaking-change** issue, that issue skips the debate and goes straight to external research arbitration. The skill is agnostic about *which* research tool to use — pick the best one available.

A legacy `--mode classic` flag preserves the old single-pass validation behavior for users who prefer it. It is deprecated and will be removed.

## codex-peer-review Features

- **Symmetric blind pass** — both AIs review without priming, dramatically increasing coverage
- **Canonical issue IDs** — same finding from both AIs collapses to one row, doubles as a confidence signal
- **Per-issue state machine** — deterministic convergence, no rationalization loops
- **Configurable via Codex profiles** — model and reasoning effort live in `~/.codex/config.toml`, not in plugin prompts
- **Single source of truth** — full protocol lives in the skill; the agent file is a thin dispatcher
- **Slash command** — explicit `/codex-peer-review` for on-demand validation
- **Auto-trigger reminders** — hooks remind Claude to dispatch peer review before presenting plans, reviews, or recommendations

### Usage

#### Slash Command

```bash
# First-time setup
/codex-peer-review init

# Default — symmetric blind-debate mode, will ask for scope
/codex-peer-review

# Review against a specific branch
/codex-peer-review --base develop

# Review uncommitted changes
/codex-peer-review --uncommitted

# Validate an answer to a broad question
/codex-peer-review "Should we use microservices or a monolith here?"

# Legacy single-pass validation (deprecated)
/codex-peer-review --mode classic
```

#### Automatic Mode

Hooks remind Claude to dispatch the peer reviewer before presenting:
- Implementation plans or designs
- Code review results
- Architecture recommendations
- Major refactoring proposals
- Answers to broad technical questions

The reminder is advisory — Claude decides when to dispatch.

## Codex CLI Compatibility

Tested against `codex-cli 0.118.0`. The plugin uses `codex exec` exclusively for machine-readable output. **`codex review --json` and `codex review -o` do not exist in 0.118.0** — the plugin previously used these and was partially broken; this release fixes that.

Schema enforcement uses prompt templates parsed with `jq`, not `--output-schema` (which is unstable in 0.118.0 under `--json`).

## Permissions

The plugin uses heredoc stdin to minimize permission prompts. On first use, you'll be asked to approve:

| Pattern | Purpose |
|---------|---------|
| `codex exec*` | Run blind-pass and debate prompts |
| `jq *` | Parse Codex JSONL output |

Select "Always allow" to avoid repeated prompts.

## Marketplace Structure

```
agent-peer-review/
├── .claude-plugin/
│   └── marketplace.json           # Marketplace registry
└── plugins/
    └── codex-peer-review/         # Codex CLI peer review plugin
        ├── .claude-plugin/
        │   └── plugin.json
        ├── agents/
        │   └── codex-peer-reviewer.md  # Thin dispatcher
        ├── skills/
        │   └── codex-peer-review/
        │       ├── SKILL.md            # Single source of truth
        │       ├── discussion-protocol.md
        │       ├── escalation-criteria.md
        │       └── common-mistakes.md
        ├── commands/
        │   └── codex-peer-review.md    # Includes init subcommand
        └── hooks/
            ├── hooks.json
            └── *-peer-review-check.sh
```

## Architecture

All peer review work runs in a **subagent context** to keep the main conversation clean. The main context only sees the synthesized verdict.

```
Main Conversation                    Subagent Context
       │                                    │
       │ Claude forms opinion               │
       │                                    │
       ├──── dispatch to agent ────────────►│
       │                                    │ Round 0: blind pass (Claude + Codex)
       │                                    │ Canonicalize issues
       │                                    │ Round 1+: per-issue debate
       │                                    │ External research (if escalated)
       │                                    │ Verdict synthesis
       │◄──── return synthesized verdict ───┤
       │                                    │
       │ Present to user                    │
       ▼                                    ▼
```

## Adding Future Plugins

This marketplace is designed to host multiple peer review plugins. Examples:
- `claude-peer-review` — Claude reviewing Claude (different model versions)
- `gemini-peer-review` — Gemini as the peer reviewer

To add a new plugin, create a new directory under `plugins/` following the same structure.

## Contributing

Contributions are welcome! Please ensure changes maintain the language-agnostic nature of the prompts and examples, and update the protocol in `SKILL.md` rather than duplicating logic in the agent file.

## License

MIT

## Related Resources

- [Claude Code Documentation](https://code.claude.com/docs)
- [OpenAI Codex CLI](https://github.com/openai/codex)
- [Claude Code Plugins Guide](https://code.claude.com/docs/en/plugins)

---

**Why two AIs reviewing in parallel?** Every analysis has blind spots. Asking one AI to "validate" another's work anchors on the proposer's framing. Independent symmetric review catches roughly twice as many issues — and the per-issue debate phase resolves conflicts deterministically without the rationalization loops that plague free-form AI discussion.
