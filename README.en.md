# Jev Gate Lite

[繁體中文](README.md) · **English**

A slimmed-down Jev Gate: cloud Jev judgments only, without the local Laya model.

It gives Claude Code and Codex:

| Component | What it does |
|---|---|
| Hybrid compaction | Fires automatically at 60% of the context window or 200K tokens by default. Jev decides per tool call which calls and results can go; what stays is byte-for-byte unchanged. If that saves under 30%, built-in compaction takes over and summarises |
| Tool-output pruning | Long logs and search results keep only the parts relevant to the task |
| MCP server | Two tools, `jev_evaluate` and `jev_apply_policy`, for classification and review advice (advisory, never authorization) |
| Subagent triage hook | Reserved interface; currently never changes the model (see below) |

## How hybrid compaction works

### When it fires

Checked once at the end of every turn. Whichever of these comes **first** triggers it; no manual command needed:

| Condition | Default | Variable |
|---|---|---|
| Share of the context window | 60% | `JEV_COMPACT_AT_PERCENT` |
| Absolute tokens | 200000 (200K) | `JEV_COMPACT_AT_TOKENS`, 0 disables |

- 200K window: 60% = 120K, so the percentage fires first.
- 1M window: 60% = 600K, so 200K fires first. Long-context quality drops well before 600K, hence the absolute ceiling.

Compaction that Claude Code starts on its own at the wall, and manual `/compact`, go through the same flow.

### What one compaction does

1. Each tool call gets one Jev System One question (about 350 in / 35 out tokens); the batch goes out in one request.
2. Only tool calls and tool results are dropped. User and assistant prose is never dropped or truncated. A call and its result go together.
3. The first turn and the last 6 turns are pinned.
4. What stays is kept verbatim, with no rewriting or summary.

### When built-in compaction takes over

In any of these cases the original messages go back unchanged and Claude Code's built-in compaction summarises:

| Case | Default | Variable |
|---|---|---|
| Saving too small (ratio) | under 30% | `JEV_COMPACT_MIN_REDUCTION_PCT` |
| Saving too small (size) | under 4000 chars | `JEV_COMPACT_MIN_REDUCTION` |
| Too much left after the last Jev pass | over 60000 tokens | `JEV_COMPACT_RESET_ABOVE` |
| Jev judgment failed, timed out, bad key | — | — |

Why the third rule: Jev only drops tool output, so prose accumulates turn after turn. Once the residue crosses the line, one built-in summary clears it, then Jev resumes.

Failure keeps the context; a Jev error never loses content.

### Design basis

The hybrid follows the JetBrains Research paper [The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management](https://arxiv.org/abs/2508.21433) (Lindenbauer et al., arXiv 2508.21433). It compares five model configurations in SWE-agent on SWE-bench Verified:

- Masking old tool output (observation masking) halves cost against the raw agent while matching, and sometimes slightly exceeding, the solve rate of LLM summarization.
- A hybrid that masks first and summarises late cuts cost a further 7% against masking alone and 11% against summarization alone.

Jev dropping tool output per call is the masking step, except Jev judges what can go instead of using a fixed turn window. Built-in compaction is the late summary.

The 60000-token residue threshold comes from Claude Code transcripts (200K window, samples at 140K–200K before compaction, median context on the next turn):

| Compaction in the same session | Jev only | Built-in only |
|---|---|---|
| 1st | 47K | 39K |
| 2nd | 58K | 44K |
| 3rd | 68K | 48K |
| 4th and later | 75K | 51K |

Each Jev pass leaves more behind because prose keeps accumulating. So once the residue passes 60K, one built-in summary clears it, then Jev resumes. We also tried summarising only the middle after a Jev pass; it came out 21% larger, because the built-in summary re-attaches recently read files, so it was not adopted.

## How tool-output pruning works

Pruning runs after a tool returns and before the output enters the model context:

- Long output is split into 16000-char chunks; at most 8 chunks are judged per call, the rest pass through verbatim (never truncated).
- A chunk is dropped only when Jev's drop probability is ≥ 0.7. That threshold was calibrated on real tool output: 0.6 dropped error lines buried in noise.
- Each judgment times out after 2000 ms (`JEV_TIMEOUT_MS`); on timeout the original is kept.
- `JEV_PRUNE_ENABLED=0` turns it off; tool output is then never sent to Jev.

## Subagent triage hook

The `agent.spawn` hook is designed to pick a subagent model by task difficulty (`JEV_TRIAGE_MODEL_CHEAP` / `MEDIUM` / `HARD`). **The hook currently has no Jev transport, so every judgment fails open to "keep the original model": in practice it never changes a subagent's model.** By design it also never downgrades high-risk, low-confidence or large-context work. `JEV_TRIAGE_ENABLED=0` turns it off entirely.

## Differences from the full version

The full [jev-gate-mcp](https://github.com/royalskynet/jev-gate-mcp) also ships the local Laya model, the cascade server, and OpenMausBot / Hermes / OpenClaw integrations. Lite drops them, so:

- No ~540 MB local model download and no `@receptron/laya` dependency.
- Every judgment calls cloud Jev and may be billed.
- `JEV_PROVIDER=laya` fails with `provider laya is not included in this build`.

## Install

Requires Node.js 22 or 24 and a Jev key (OpenCode Zen free tier or OpenRouter).

```sh
git clone https://github.com/royalskynet/jev-gate-lite.git
cd jev-gate-lite
npm install
npm test
```

### Claude Code plugin (compaction + pruning)

Register it as a local marketplace in `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "jev-gate": { "source": { "source": "directory", "path": "/path/to/jev-gate-lite" } }
},
"enabledPlugins": { "jev-gate@jev-gate": true }
```

Or try it for one session: `claude --plugin-dir /path/to/jev-gate-lite`. It takes effect in the next session.

### MCP server (judgment tools)

```sh
claude mcp add jev_gate -s user \
  -e JEV_PROVIDER=openrouter -e JEV_MODEL=typesafe/jev-1.13 \
  -e JEV_API_KEY_FILE=/path/to/OPENROUTER_API_KEY.txt \
  -- node /path/to/jev-gate-lite/gateway/src/stdio.js
```

The server name must come before `-e`. Check: `claude mcp list` shows `jev_gate` ✓ connected.

Codex: `.codex/config.toml` in the repo already configures the MCP server; mark the repo trusted.

## Keys and provider

| Setting | Default |
|---|---|
| Key file | `~/.creds/kv/OPENCODE_ZEN_API_KEY.txt` (override with `JEV_API_KEY_FILE`) |
| Endpoint | `https://opencode.ai/zen/v1/systemone` (Zen free tier) |
| Use OpenRouter | `JEV_ENDPOINT=https://openrouter.ai/api/v1/systemone`, `JEV_MODEL=typesafe/jev-1.13` |

The Zen free tier has a quota and may be rate limited upstream. On failure it cools down and asks the OpenRouter backup (`~/.creds/kv/OPENROUTER_SECOND_API_KEY.txt`, override with `JEV_ZEN_BACKUP_KEY_FILE`); without a backup key it fails closed.

Keys are read from files only, never passed as tool arguments, never logged. State is masked for known key formats before it is sent.

## Local files it writes

| File | Contents | Disable / relocate |
|---|---|---|
| `~/.jevgate/usage*.jsonl` | Token counts and status per Jev call, no content | `JEV_USAGE_LOG=off` |
| `~/.jevgate/teacher-corpus.jsonl` | State, questions and answers for each MCP server judgment (keys masked, file mode 0600) | `JEV_TEACHER_CORPUS=/dev/null` |
| `~/.jevgate/zen-free-status.json` | Zen cooldown state | — |
| `~/.openclaw/state/jev-prune.log` | Pruning ledger | `JEV_LEDGER` |
| `~/.openclaw/state/jev-compact.log` | Compaction ledger | — |

The teacher corpus stores conversation content. To keep none, set `JEV_TEACHER_CORPUS=/dev/null`.

## Switches

| Variable | Effect |
|---|---|
| `JEV_COMPACT_ENABLED=0` | Turn hybrid compaction off; built-in compaction only |
| `JEV_COMPACT_PIN_LAST` | Recent turns always kept, default 6 |
| `JEV_COMPACT_BACKEND` | `auto` (default, picked per transcript shape), `anthropic`, `openai` |
| `JEV_PRUNE_ENABLED=0` | Turn tool-output pruning off |
| `JEV_TRIAGE_ENABLED=0` | Turn the subagent triage hook off |
| `JEV_KILL_SWITCH=1` | Disables the MCP server's judgment tools (plugin hooks unaffected) |
| `JEV_UI_LOG=1` | Show hook log lines in the Claude Code UI |

Trigger and hand-off thresholds are under "How hybrid compaction works" above.

Do not enable this plugin and the standalone fast-jev-compaction plugin together: both answer the same compaction event. When it detects the other one enabled, this one stands down.

## License

MIT, see [LICENSE](LICENSE).
