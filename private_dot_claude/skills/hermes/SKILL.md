---
name: hermes
description: Troubleshoot Hermes Agent (@Hermes in Slack). Use when Hermes is misbehaving — not responding, auth errors, wrong model, config not taking effect. Layer-first triage, not a setup guide.
user_invocable: true
---

# Hermes Troubleshooting

Diagnostic runbook for Hermes Agent (@Hermes in Slack). For setup and config details, see `docs/HERMES_SETUP.md` and memory file `project-hermes-agent.md`.

## Protocol

### 1. Gather evidence (always start here)

```bash
hermes status
hermes doctor
hermes logs --since 15m --level WARNING
```

### 2. Triage — which layer is broken?

Read the evidence output and identify the failing layer:

| Layer | Signals in evidence output |
|---|---|
| Gateway | PID missing, "not running", crash stack in logs |
| Auth | `invalid_auth`, `token_expired`, `401`, refresh errors |
| Provider | Wrong model name in response, `fallback` mentions in logs, empty output |
| Slack | `channel_not_found`, `not_in_channel`, no events arriving |
| Config | Setting visible in `hermes config show` but behavior unchanged |

### 3. Match symptom → fix

#### Gateway

| Symptom | Fix |
|---|---|
| Gateway not running | `hermes gateway restart` |
| Crashes after update | `hermes doctor --fix` then `hermes gateway restart` |
| LaunchAgent not starting | `hermes gateway install` (reinstalls plist) |
| LaunchAgent stale after uninstall | `launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.gateway.plist` then `rm` the plist |

#### Auth

| Symptom | Fix |
|---|---|
| Codex OAuth expired | `hermes auth add codex --type oauth` (device code re-auth) |
| `invalid_auth` on Slack | Verify tokens in `~/.hermes/.env` match Slack app config |
| API key rejected / revoked | `hermes status` to see which key; replace in `~/.hermes/.env`; `hermes gateway restart` |

#### Provider

| Symptom | Fix |
|---|---|
| Wrong model answering | `hermes model` to check/change; `hermes gateway restart` |
| Fallback chain firing unexpectedly | `hermes fallback list` — check primary is healthy; `hermes logs --since 15m` for why primary failed |
| Empty output / retries | Usually upstream SDK issue; `hermes update` then `hermes gateway restart` |
| Want to add/remove fallback | `hermes fallback add` / `hermes fallback remove` |

#### Slack

| Symptom | Fix |
|---|---|
| `channel_not_found` | `/invite @Hermes` in the channel |
| Slash commands not working in thread | Slack limitation — type commands as plain text in threads |
| No events arriving | Check `hermes status` Slack section; verify bot event subscriptions in Slack app config |
| Dual-bot race (inconsistent responses) | Only one bot per Slack app token (`A0ACK7QMT1N`); stop the other |

#### Config

| Symptom | Fix |
|---|---|
| `hermes config set` didn't take effect | `hermes gateway restart` (not all settings hot-reload) |
| Want to see current config | `hermes config show` |
| Want to edit config directly | `hermes config edit` (opens in `$EDITOR`) |

### 4. Verify the fix

Re-run the evidence commands from step 1. If the fix was gateway-related, also test in Slack (send a message to @Hermes).

### 5. Escalate

After 2 failed attempts at the same approach, stop and tell the user what you tried and what failed. Don't keep cycling.

## Key CLI commands

```bash
hermes status              # full component status
hermes doctor              # diagnostics with recommendations
hermes doctor --fix        # auto-fix what it can
hermes logs --since 15m    # recent logs (add --level WARNING to filter)
hermes gateway restart     # restart after config/auth changes
hermes update              # pull upstream updates
hermes fallback list       # show fallback provider chain
hermes config show         # show current config
hermes sessions list       # list recent sessions
hermes insights            # usage analytics
```

## References

- Setup & installation: `docs/HERMES_SETUP.md`
- Current config & state: memory file `project-hermes-agent.md`
- Official docs: https://hermes-agent.nousresearch.com/docs/
