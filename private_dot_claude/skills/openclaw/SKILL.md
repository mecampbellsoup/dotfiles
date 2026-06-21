---
name: openclaw
description: Troubleshoot OpenClaw bot (@OpenClaw in Slack). Use when OpenClaw is misbehaving — not responding, auth errors, wrong model, config not taking effect. Layer-first triage, not a setup guide.
user_invocable: true
---

# OpenClaw Troubleshooting

Diagnostic runbook for OpenClaw bot (@OpenClaw in Slack).

**Key paths:**
- Config: `~/.openclaw/openclaw.json`
- Workspace: `~/Documents/OpenClaw` (SOUL.md, TOOLS.md, skills, memory)
- Logs: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- LaunchAgent: `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
- Auth profiles: `~/.openclaw/agents/main/agent/auth-profiles.json`

## Protocol

### 1. Gather evidence (always start here)

```bash
openclaw gateway status --deep
openclaw models status
openclaw doctor --deep --yes
tail -100 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -i "error\|fail\|warn" | tail -20
```

### 2. Triage — which layer is broken?

Read the evidence output and identify the failing layer:

| Layer | Signals in evidence output |
|---|---|
| Gateway | PID missing, "not running", crash stack in logs, `startup-sidecars-pending` |
| Auth | `FailoverError`, `Claude CLI failed`, `chain_exhausted`, `Missing auth` in models status |
| Billing | `Third-party apps now draw from your extra usage`, `[disabled:billing Xh]` in models status |
| Provider | Wrong model in response, `model_fallback_decision` in logs, empty output |
| Slack | `socket disconnected`, `channel_not_found`, no events arriving, warning emoji responses |
| Config | Setting visible in `config get` but behavior unchanged, `.bak` reversion |

### 3. Match symptom → fix

#### Gateway

| Symptom | Fix |
|---|---|
| Gateway not running | `openclaw gateway restart` |
| Crashes after update | `openclaw doctor --deep --yes` then `openclaw gateway restart` |
| LaunchAgent not starting | Check plist: `launchctl print gui/$(id -u)/ai.openclaw.gateway` |
| `startup-sidecars-pending` in logs | Gateway still initializing — wait 10s, retry |
| Module not found errors after update | `npm update -g openclaw` (corrupted dist), then `openclaw gateway restart` |

#### Auth

| Symptom | Fix |
|---|---|
| `FailoverError` / `Claude CLI failed` | `openclaw models auth login --provider anthropic --method cli --set-default` (requires interactive TTY), then `openclaw gateway restart` |
| `Missing auth` for anthropic in models status | Same as above — auth profile was lost on restart |
| `chain_exhausted` with no fallback | Primary auth failed and no fallback configured — fix primary first |
| API key rejected | Check `openclaw models status` for which provider; re-add credentials |
| `deactivated_workspace` (OpenAI 402) | Codex OAuth bound to wrong workspace. Re-auth: `openclaw models auth login --provider openai --method codex` and pick the active account |

**Pitfall:** Auth profiles can be lost after gateway restart. If the bot was working and stops after a restart, check `openclaw models status` for `Missing auth` before anything else.

#### Billing

| Symptom | Fix |
|---|---|
| `Third-party apps now draw from your extra usage` | Anthropic separated Agent SDK/`claude -p` usage from subscription limits (May 2026). Either add funds at claude.ai/settings/usage, or switch to OpenAI as a bridge provider |
| `[disabled:billing Xh]` in models status | Billing circuit breaker — repeated auth/billing failures disable the provider for hours. Fix the underlying auth issue, then re-auth to clear it |
| Want to check extra usage balance | Log into claude.ai/settings/usage — the "Extra usage" section shows balance, spend cap, and auto-reload status |

**Context:** Max 5x+ subscribers get a $100/month Agent SDK credit starting June 15, 2026 (must be claimed at claude.ai/settings/usage). Until then, third-party tools need either extra usage funds or a bridge to another provider (e.g., `openai/gpt-5.4-mini` via Codex plugin).

#### Provider

| Symptom | Fix |
|---|---|
| Wrong model answering | `openclaw config get agents.defaults.model.primary` to check; `openclaw config set agents.defaults.model.primary <provider/model>` to change |
| Model fallback firing unexpectedly | Check logs for `model_fallback_decision` — the `reason` and `errorPreview` fields explain why primary failed |
| Want to change model | `openclaw config set agents.defaults.model.primary <provider/model>`; this hot-reloads (no restart needed) |
| `Requested agent harness "codex" is not registered` | OpenAI gpt-5.x models need the Codex plugin: `openclaw plugins install @openclaw/codex`, then restart |

#### Slack

| Symptom | Fix |
|---|---|
| Warning emoji response (`:warning: Something went wrong`) | Usually auth or billing failure, not Slack — check Auth and Billing layers first |
| `socket disconnected` in logs | Transient — OpenClaw auto-retries (12 attempts). If persistent, check tokens |
| Slack shows "not configured" after update | Slack plugin is external since 2026.5.8+. Reinstall: `openclaw plugins install @openclaw/slack --force`, then restart |
| Bot not responding in a channel | Channel must be enabled: `openclaw config set channels.slack.channels.<CHANNEL_ID>.enabled true` then `openclaw gateway restart` |
| No typing indicators | Check `typingMode` — `"thinking"` requires `reasoningLevel: "stream"`; use `"instant"` instead |
| Bot not threading replies | `openclaw config set channels.slack.replyToMode all` |
| Pairing required after fresh install | `openclaw pairing approve slack <CODE>` |

**Pitfall:** Enabling a new channel requires a gateway restart — hot reload doesn't pick it up.

#### Config

| Symptom | Fix |
|---|---|
| Config change didn't take effect | `openclaw gateway restart` (most changes require restart; `agents.defaults.model.primary` hot-reloads) |
| Config reverted to old values | Check for `.bak` file: `ls ~/.openclaw/openclaw.json.bak`. If gateway had validation errors, it may have auto-restored from backup. Remove `.bak` before re-setting: `rm ~/.openclaw/openclaw.json.bak` |
| Want to see current config | `openclaw config get <key>` or read `~/.openclaw/openclaw.json` |
| Duplicate JSON keys after edit | Parse error — read the file, find and remove the duplicate block |
| Workspace files not being read | Verify path: `openclaw config get agents.defaults.workspace` — should be `~/Documents/OpenClaw` |

**Pitfall:** When config has validation errors, the gateway silently restores from `.bak` on restart. Your edit appears to vanish. Always remove `.bak` before re-applying a fix.

### 4. Verify the fix

Re-run the evidence commands from step 1. Then test in Slack — send a message to @OpenClaw in the bot's DM (`D0B18G90UMN`).

### 5. Escalate

After 2 failed attempts at the same approach, stop and tell the user what you tried and what failed. Don't keep cycling.

## Key CLI commands

```bash
openclaw gateway status --deep    # full gateway health
openclaw gateway restart          # restart after config/auth changes
openclaw models status            # configured models + auth state
openclaw doctor --deep --yes      # diagnostics with auto-fix
openclaw config get <key>         # read config value
openclaw config set <key> <value> # write config value
openclaw plugins list             # installed plugins + status
openclaw plugins install <pkg>    # install/reinstall a plugin (--force to overwrite)
openclaw channels list            # channel providers + connection state
openclaw skills list              # loaded skills
openclaw skills check             # skill health
```

## Enabled Slack channels

| Channel | ID | Notes |
|---|---|---|
| Bot DM | `D0B18G90UMN` | Direct messages with OpenClaw |
| #dev | `C08852LKJ1G` | General development |
| #bots | `C0AKCK0F6AV` | Bot testing |
| DM w/ Riley | `D087GL2KXL6` | Core team comms |

## References

- Workspace files: `~/Documents/OpenClaw` (SOUL.md, TOOLS.md, USER.md, skills/)
- Current config & state: memory file `project-openclaw-eval.md`
- Official docs: https://docs.openclaw.ai
