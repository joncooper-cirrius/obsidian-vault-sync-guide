# Agent Runbook — Set up Obsidian ↔ Git vault sync

> **For a Claude Code agent.** Brandon: paste this file (or its raw URL) to your agent and say *"follow this runbook to set up my vault's Git backup."*
>
> Raw URL (fetchable):
> `https://raw.githubusercontent.com/joncooper-cirrius/obsidian-vault-sync-guide/main/AGENT-SETUP.md`
>
> Companion human guide: https://joncooper-cirrius.github.io/obsidian-vault-sync-guide/

---

## 0. Scope — read this first

**You (the agent) DO:**
- Create the private GitHub repo
- Write the `.gitignore` **before** the first commit
- `git init` the vault, make the first commit, push to GitHub
- Write the Obsidian Git plugin config (`data.json`) and register it in `community-plugins.json`
- Verify everything end-to-end

**You DO NOT (hand these back to Brandon — they need a GUI, an app store, or his credentials):**
- Install the Obsidian desktop app, `git`, or the GitHub CLi (`gh`) — assume present; if missing, **stop and ask Brandon to install them**
- Authenticate `gh` (that's Brandon's GitHub login) — verify it's done; if not, ask him to run `gh auth login`
- Install/enable the plugin from Obsidian's GUI or **restart Obsidian** — give Brandon the one toggle + restart at the end
- Anything on the **phone** — that's 100% human (see §7)

**Operating rules:** be idempotent (check state before acting; don't re-init an existing repo or clobber an existing `.gitignore` — append/merge instead). Never commit secrets (see §3). Confirm the decisions in §1 before making changes.

---

## 1. Ask Brandon these first

| Decision | Default / guidance |
|---|---|
| **Vault absolute path** | e.g. `/Users/brandon/notes` (macOS) or `C:\Users\brandon\notes` (Windows). Required. |
| **Repo name** | e.g. `my-vault`. |
| **Private or public?** | **PRIVATE.** A personal notes vault must be private. Do not create a public repo for notes. |
| **Phone path (informational)** | Obsidian Sync (paid, easiest, best on iPhone) **or** free Git (Working Copy on iOS / Obsidian Git plugin on Android). Brandon sets this up himself later — see the human guide §6. |

---

## 2. Verify prerequisites

```bash
git --version            # must exist — else ask Brandon to install git
gh --version             # GitHub CLI — else ask Brandon to install it
gh auth status           # must be logged in — else ask Brandon to run: gh auth login
gh auth setup-git        # makes git use gh's credentials for push/pull (safe to re-run)
```

Confirm Obsidian is installed (macOS: `ls -d /Applications/Obsidian.app`; Windows: check `%LOCALAPPDATA%\Obsidian`). If absent, ask Brandon to install it from https://obsidian.md and re-run you.

Confirm the vault path exists and note whether it is **already** a git repo:
```bash
VAULT="/absolute/path/to/vault"           # from §1
test -d "$VAULT" && echo "vault ok"
git -C "$VAULT" rev-parse --is-inside-work-tree 2>/dev/null && echo "already a git repo" || echo "not a repo yet"
```

---

## 3. Write `.gitignore` BEFORE the first commit

If a `.gitignore` already exists, **merge** these lines in (don't overwrite). Create `"$VAULT/.gitignore"` with:

```gitignore
# Per-device Obsidian UI state (don't sync — causes noise/conflicts)
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# OS junk
.DS_Store
Thumbs.db

# Temp / backup files
*.tmp
*.bak

# NEVER sync secrets (tokens, API keys, env files)
.env
*.secret
.mcp.json
```

**Secret pre-flight (mandatory).** Before the first commit, scan what's about to be tracked and abort if anything looks like a credential:
```bash
git -C "$VAULT" add -A -n | sed 's/^add //' | grep -iE '\.(env|pem|key)$|secret|token|credential|\.mcp\.json' && echo "POTENTIAL SECRET — stop and review with Brandon" || echo "no obvious secrets staged"
```
Reminder to relay to Brandon: once a secret is committed, deleting the file does **not** remove it from history — it must be rotated + scrubbed with `git filter-repo`. Getting `.gitignore` right now avoids that.

---

## 4. Initialize, commit, push

```bash
# only if "not a repo yet" from §2
git -C "$VAULT" init -q
git -C "$VAULT" branch -M main

# set identity if unset
git -C "$VAULT" config user.name  >/dev/null || git -C "$VAULT" config user.name  "Brandon"
git -C "$VAULT" config user.email >/dev/null || git -C "$VAULT" config user.email "brandon@example.com"

# Windows ONLY: prevent line-ending churn / phantom conflicts
# git config --global core.autocrlf input

git -C "$VAULT" add -A
git -C "$VAULT" commit -q -m "Initial vault commit"

# create the PRIVATE repo and push (uses gh's stored credential)
gh repo create "REPO_NAME" --private --source="$VAULT" --remote=origin --push
```

If the push ever errors with auth failure (common on a fresh setup), do one authenticated push then retry:
```bash
git -C "$VAULT" -c credential.helper='!gh auth git-credential' push -u origin main
```

Set the credential helper so future pushes are silent:
- macOS: `git -C "$VAULT" config credential.helper osxkeychain`
- Windows: `git -C "$VAULT" config credential.helper manager`

---

## 5. Configure the Obsidian Git plugin (config files)

The plugin files (`main.js`, `manifest.json`, `styles.css`) come from Obsidian's plugin browser, which is **Brandon's one GUI step** (§7). You handle the **config**:

**a. Register the plugin as enabled.** Ensure `"obsidian-git"` is in `"$VAULT/.obsidian/community-plugins.json"` (it's a JSON array). If the file doesn't exist yet, create it as:
```json
["obsidian-git"]
```
If it exists, add `"obsidian-git"` to the array if missing (preserve existing entries).

**b. Write the plugin settings.** Create/overwrite `"$VAULT/.obsidian/plugins/obsidian-git/data.json"` with this exact config (mirrors Jon's working setup — commit **and** push every 10 min, auto-pull on boot + every 10 min, pull-before-push, merge):

```json
{
  "commitMessage": "vault backup: {{date}}",
  "autoCommitMessage": "vault backup: {{date}}",
  "commitDateFormat": "YYYY-MM-DD HH:mm:ss",
  "autoSaveInterval": 10,
  "autoPushInterval": 0,
  "autoPullInterval": 10,
  "autoPullOnBoot": true,
  "autoCommitOnlyStaged": false,
  "disablePush": false,
  "pullBeforePush": true,
  "showErrorNotices": true,
  "showStatusBar": true,
  "syncMethod": "merge",
  "mergeStrategy": "none",
  "refreshSourceControl": true,
  "refreshSourceControlTimer": 7000,
  "showBranchStatusBar": true
}
```

> **Sequencing note:** if Brandon installs the plugin via the GUI *after* you write `data.json`, the plugin keeps your file (it only writes defaults when none exists). If he already installed it, your write simply updates the settings — they take effect on the next Obsidian restart. Either order is fine; just make sure the restart in §7 happens last.

---

## 6. Verify

```bash
gh repo view "REPO_NAME" --json visibility,url --jq '{visibility,url}'   # expect "PRIVATE"
git -C "$VAULT" remote -v
git -C "$VAULT" log --oneline -1
git -C "$VAULT" status -s                                                # clean = pushed
python3 -c "import json,sys; json.load(open(sys.argv[1]))" "$VAULT/.obsidian/plugins/obsidian-git/data.json" && echo "data.json valid"
```
Confirm: repo is **PRIVATE**, remote points to it, the initial commit is pushed, and `data.json` is valid JSON.

---

## 7. Hand back to Brandon

Tell Brandon exactly this:

1. **Open Obsidian** on this computer → open this folder as a vault if it isn't already.
2. **Community plugins → Browse → search "Git" (by Vinzent) → Install → Enable.** (The settings are already configured.)
3. **Restart Obsidian.** You should see a Git status indicator in the bottom bar and commits named `vault backup: …` appearing automatically.
4. **Second computer:** repeat this runbook there, except instead of creating the repo, clone it (`git clone <repo-url>`), open the folder as a vault, and do steps 1–3.
5. **Phone (you do this — an agent can't):** pick a path from the human guide §6 — Obsidian Sync (easiest) or, free, Working Copy on iPhone / the Obsidian Git plugin on Android.

---

## Quick troubleshooting (relay to Brandon as needed)
- **"Authentication failed" on push:** `gh auth login` then `gh auth setup-git`; or one push with `-c credential.helper='!gh auth git-credential'`.
- **Fresh clone on another device can't push:** same fix — the clone has no stored credential until the first authenticated push.
- **Windows shows every file as changed:** `git config --global core.autocrlf input`.
- **"Updates were rejected" after editing on two devices:** keep *pull before push* ON; resolve any `<<<<<<<` / `>>>>>>>` markers in the conflicting note, save, push.
- **iPhone + Obsidian Git plugin is flaky:** use Working Copy or Obsidian Sync instead.
