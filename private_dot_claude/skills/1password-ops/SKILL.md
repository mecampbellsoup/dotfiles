---
name: 1password-ops
description: 'Matt''s 1Password CLI and credential-handling rules — op run vs bare gh, Touch ID behavior in Claude Code, the 1Password Environments MCP, live web-session logins via the persistent Playwright profile (and when to hand off authentication rather than materialize credentials), sharing vault access with family, the fact that `op item move` prints plaintext secrets, the strip-then-move recipe for SSO ("sign in with") items that `op item move`/`delete` refuse, and the `env -u OP_SERVICE_ACCOUNT_TOKEN` escape hatch that reaches the built-in Private vault the service-account token structurally cannot see. Load before any `op` command, before logging into a site on Matt''s behalf, before granting vault access, and whenever a task needs a credential, token, API key, or password. Triggers: 1password, op run, op item, vault, credential, secret, API key, token, log me in, sign in to.'
---

## 1Password CLI

**`gh` commands:** `gh` authenticates via the macOS keychain independently, so `op run --` buys it nothing — don't add the wrapper to a bare `gh` call. But it isn't harmful either, and plenty of project skills/docs prescribe `op run -- gh`; following them is fine, not a violation of this rule.

**When an `op run -- <cmd>` call errors, retry `<cmd>` without the prefix before diagnosing anything else.** `op run` failures surface as though they came from the wrapped command, so they get attributed to the wrong tool. The real case (2026-07-26): `op run -- gh` failed repeatedly with `dial tcp ...: connect: network is unreachable` while `curl https://api.github.com` returned 200, git-over-ssh worked, and plain `gh` succeeded instantly. A subagent read that as a genuine outage, correctly applied the 2-attempt escalation rule, and stopped — costing an agent stop on a PR that was ready to merge. The failure is transient (the same commands worked before and after; an 11-command sweep later passed clean, even with `op whoami` reporting no session), so the fix is retry-without-the-prefix, never "this tool is broken" or "the guidance prescribing it is wrong."

**1Password Environments (MCP):** The `1password` MCP server is configured. Use `mcp__1password__*` tools for Environments (`.env` file management). Requires one Touch ID prompt per CC session via `mcp__1password__authenticate`.

**General `op` vault queries** (`op item get`, `op read`, `op item list`): These prompt for Touch ID on every call from within Claude Code (no-TTY by design). Use them when needed and expect a prompt. Do not wrap with `op run --` — it doesn't help.

**The service-account token is one auth mode, not a ceiling — `env -u OP_SERVICE_ACCOUNT_TOKEN op …` reaches vaults it cannot.** In webapp worktrees `.envrc` exports that token, so every `op` call authenticates as the service account and sees only the four shared vaults. Unset it for one command and `op` falls back to desktop-app integration, authenticating as Matt: five vaults, including the account's built-in `Private`. Touch ID prompts, which is the whole cost.

That matters because the gap is invisible from the CLI and reads as absence. `op vault get Private` answers `"Private" isn't a vault in this account` — a vault plainly present in the app one pane over, and a message indistinguishable from a misspelled name. `Private` is a vault **class** a service account cannot be granted, so no `op vault user grant` closes it; the sidebar tell is that it carries no shared-vault (people) icon.

Reach for `env -u` whenever a lookup comes back empty and the item is one a person would have saved from a browser — a 1Password browser extension's default save vault is per-profile and defaults to `Private`, so items land there routinely and are then unreachable by every `op://` reference, `.envrc` lookup, and automated read. Confirmed 2026-08-25: five months of browser saves sat invisible until this was tried, and the fix is to relocate them into a shared vault (see the SSO recipe below — such items are usually "sign in with" logins).

**Live web-session logins (need to actually authenticate into a site, not just reference a field):** The Playwright profile persists across sessions — check whether it's already logged in before ever prompting for a login. Navigate first to a page that only renders when authenticated (account home, order history, dashboard) for that site; if it loads real account content, the session is still live — proceed directly, no login prompt. A redirect to a sign-in-shaped URL is not automatically a credential wall: if that page shows the account pre-recognized (email pre-filled, a one-click "Sign in as [name]" / "Continue as [name]" button) with no password/2FA field visible, it's a session-confirmation interstitial — click it through yourself, no handoff needed. Only hand off if, after that, an actual password or 2FA field appears (or the page shows a fully logged-out state with no recognized identity at all): don't attempt to print 1Password item fields (username/password) directly to stdout first — the permission classifier blocks this as credential materialization, regardless of intent — go straight to Playwright, open the login page, then hand off authentication to the user (they type credentials/2FA themselves) rather than trying to retrieve-then-type the credential yourself. **Close out sessions when the task concludes:** once a Playwright-driven task is done (dispute filed, cancellation confirmed, form submitted), call `browser_close` — don't leave an authenticated tab (bank, subscription account, etc.) sitting open indefinitely. Skip the close only if another step in the same turn will keep using the same browser, or the user asks to leave it open for their own review.

If you see `missing required scopes`, check your GitHub CLI token scopes.

**Sharing vault access (Business/Teams accounts, e.g. adding a family member to a shared vault):** the pattern is `op vault create <name>` → `op vault user grant --vault <name> --user <email> --permissions allow_viewing,allow_editing` → `op item move <item> --current-vault <src> --destination-vault <name>`. A brand-new vault is isolated by default (no auto-grant to a `Team Members`-style group) — verify with `op vault group list <name>` before granting individual users. If a grant 400s ("structure of request was invalid") for a specific user but the identical command works for another already-`ACTIVE` user, the cause is usually the target user still being `PENDING` (accepted the invite but hasn't been confirmed/completed setup) — check `op user get <email>` for `state`, not a syntax issue.

**`op item move` prints full plaintext secret values in its JSON output, even without `--reveal`.** Unlike `item get`/`item list`, which conceal fields by default, a move dumps every field's actual value (passwords, security tokens) into stdout — a live secret-leak vector, same class as the "never pipe a secret's raw value into tool output" rule under § Workflow, just a different mechanism. Don't echo or summarize that output back to the user; treat the command's own stdout as sensitive and let it pass silently. Redirect it (`>/dev/null`) rather than relying on discipline.

**Two other places a value hides when dumping an item's structure:** `password_details.history` carries previous plaintext passwords, and `--reveal` puts every concealed value in the JSON. A field-by-field dump that masks only `value` still leaks the history array — mask `password_details` too, or print lengths (`len=%d`) instead of values.

**An SSO item ("sign in with Google/GitHub/Apple") blocks `op item move` AND `op item delete` — strip the field first.** Both fail with `validateVaultItem failed to Validate: ... has unsupported field type: ssoLogin`, exit 8, nothing written. That reads as "the CLI can't handle this item, use the app," and it isn't: the validator objects to one field, not the item. The CLI surfaces that field as type `UNKNOWN` with an empty value, so filtering on the type is exact.

**Prefer deleting that one field by name over the whole-item round-trip below.** `op item edit` accepts assignment statements, and `[delete]` in place of the fieldType removes a custom field — so the offending field goes without decrypting anything:

```sh
op item get <id> --vault <src> --format=json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); \
      print('\n'.join(f\"{f.get('section',{}).get('id','')}.{f['label']}\" \
            for f in d['fields'] if f.get('type')=='UNKNOWN'))"

op item edit <id> --vault <src> '<Section_id>.sign in with[delete]=' >/dev/null
op item move <id> --current-vault <src> --destination-vault <dst> >/dev/null
```

Two reasons this is the form to reach for, and the second is the one that decides it. No concealed value is ever decrypted, so the leak surface is gone rather than merely contained. And Claude Code's auto-mode classifier **denies** the round-trip below as credential materialization — correctly, since it decrypts every field to rewrite one — so inside Claude Code that recipe cannot run at all. Confirmed 2026-08-25: five items sat unmovable through five blocked attempts until this form was found.

```sh
op item get <id> --vault <src> --format=json --reveal \
  | python3 -c "import json,sys; d=json.load(sys.stdin); \
      d['fields']=[f for f in d['fields'] if f.get('type')!='UNKNOWN' and (f.get('id') or f.get('label'))]; \
      d['sections']=[s for s in d.get('sections',[]) \
                     if any(f.get('section',{}).get('id')==s.get('id') for f in d['fields'])]; \
      json.dump(d,sys.stdout)" \
  | op item edit <id> --vault <src> >/dev/null
op item move <id> --current-vault <src> --destination-vault <dst> >/dev/null
```

`--reveal` is mandatory — without it concealed values come back empty and the edit blanks them — but the value never leaves the pipe, so nothing reaches a transcript. The `(f.get('id') or f.get('label'))` half is a second, unrelated block: `op item edit` rejects any field with neither, failing with `a field to edit must have either an ID or a Label`. Real instance — a stray 1-character field referenced as `op://Matt/Uber/`, addressable by nothing.

**Stripping the SSO field costs nothing when the item is already moving vaults, and everything otherwise.** 1Password links only work within one vault, and a move removes links to or from the moved item ([docs](https://support.1password.com/link-items/)) — so post-move the field is a dead pointer either way. On an item staying put, that field is live and dropping it destroys a working link. Re-linking is app-only: the CLI can't write the type at all. That is worth stating precisely, because both halves fail *quietly*. Reading: `additional_information` names the **provider** — `Signs in with GitHub` — on both `op item list` and `op item get`, so which service an item uses is a one-command lookup and must never be inferred from the account's email domain. What the `UNKNOWN` field hides is narrower: the *linked item*. It comes back with no value and no target ID, so which specific 1Password entry it points at is not knowable from the CLI — and neither is whether that link still resolves. A dead link and a live one are byte-identical in `op item get` (`id+label+reference+section+type`, no value), so a `Missing Item` shows up only in the app. Note the `reference` path uses the item's ID rather than its title when the title is non-unique or contains parentheses; that tracks title shape, not link health, and reads convincingly like a corruption signal. Writing: `op item create` **accepts** a field typed `ssoLogin` (or `UNKNOWN`) and exits 0, then stores it as a plain `STRING` holding whatever text you passed — `additional_information` reads `—` instead of `Signs in with Google`. Verified both ways with throwaway probes, 2026-08-20. So delete-and-recreate cannot restore a link, and a CLI "fix" would look successful while producing a text field that does nothing.

**Never guess the provider — read `additional_information`.** Case, 2026-08-22: a session ran `op item list --format=json`, filtered the output down to title/vault/category/urls, and discarded `additional_information` — the field that said `Signs in with GitHub`. It then read the `UNKNOWN` field, correctly found no target, and instead of reporting "the CLI can't tell me which provider," guessed Google from a Workspace email domain and stated it as the finding. Clicking that guess created a second, wrong Comet account under a personal Google identity, with no access to the workspace the task needed.

The link is only ever created by 1Password's own save/update flow — the "Save Item → Update Existing" dialog the extension raises after you sign in with the provider. Which means a stripped item repairs itself the next time that sign-in happens, with no separate manual step; it does not need to be recreated.

**Moving an item changes its ID** (`rw273yhd…` → `i27ihqb2…`) — it is a create-and-delete, not a relocation. Any stored reference to the old ID breaks, which is the mechanism behind vault-move link breakage. Attachments and TOTP fields *do* survive a move, contrary to community reports; verified with a throwaway probe item, 2026-08-20.

