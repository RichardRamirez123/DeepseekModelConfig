# DeepSeek + Claude Code Setup Guide (desktop → laptop transfer)

Copy-paste instructions to reproduce this machine's Claude Code configuration on another
computer. Three pieces: (A) the model picker config, (B) the PowerShell profile that points
Claude Code at DeepSeek, (C) expected behavior + verification.

> **Security:** this config stores a DeepSeek API key in plaintext in a PowerShell profile.
> Keep this file (and profile contents) out of anything shared. Never paste the real key
> into a chat or a file that syncs to people you don't trust.

---

## What the final setup gives you

| Launch / pick | Main model | Subagents & background | Sees images? |
|---|---|---|---|
| plain `claude` (or `/model` → **DeepSeek V4 Flash**) | `deepseek-v4-flash` | `deepseek-v4-flash` | no |
| `/model` → **DeepSeek V4 Pro** (session switch) | `deepseek-v4-pro[1m]` | stays flash | no |
| `/model` → **DeepSeek V4 Flash Vision** (session switch) | `deepseek-v4-flash-vision-exp` | stays flash (main chat only) | yes |
| `claude-pro` (optional launcher) | `deepseek-v4-pro[1m]` | `deepseek-v4-pro[1m]` | no |
| `claude-vision` (optional launcher) | `deepseek-v4-flash-vision-exp` | vision model | yes |

Rules that make it safe:
- **Everything defaults to flash.** All four model tiers (opus/sonnet/haiku/subagent) are
  pinned to `deepseek-v4-flash`, so nothing can silently bill pro.
- `/model` rows are sent **verbatim** — picking "DeepSeek V4 Pro" runs exactly that model,
  nothing else. Pro is only ever used when you deliberately pick it.
- If Claude Code ever sends DeepSeek an id it doesn't recognize, DeepSeek silently falls
  back to text-only flash (no error, no pro charge). A session that "can't see images"
  usually means the vision model isn't actually selected.

---

## Part A — `/model` picker shows exactly your three models

Requires **Claude Code ≥ 2.1.243** (check with `claude --version`; update with
`claude update` if older — this was built on 2.1.259).

File: `C:\Users\<you>\.claude\settings.json` (user-level settings)

Replace the file contents with:

```json
{
  "model": "haiku",
  "modelPicker": {
    "options": [
      {
        "model": "deepseek-v4-flash",
        "label": "DeepSeek V4 Flash",
        "description": "Default — fast & cheap. Text only."
      },
      {
        "model": "deepseek-v4-pro[1m]",
        "label": "DeepSeek V4 Pro",
        "description": "Flagship tier (1M context). Text only."
      },
      {
        "model": "deepseek-v4-flash-vision-exp",
        "label": "DeepSeek V4 Flash Vision",
        "description": "Experimental vision model — can see images. Priced same as flash."
      }
    ],
    "replaceBuiltInOptions": true
  },
  "effortLevel": "xhigh",
  "autoUpdatesChannel": "latest",
  "theme": "dark"
}
```

Notes:
- `"model": "haiku"` only matters if `ANTHROPIC_MODEL` env is unset (it isn't below) — then it
  resolves through the haiku-tier env var to flash. Keep it.
- `"effortLevel"`, `"autoUpdatesChannel"`, `"theme"` are personal preferences — adjust or drop.
- `replaceBuiltInOptions: true` removes the built-in Opus/Sonnet/Haiku rows so there's nothing
  confusing left to click.
- If the new rows don't appear in `/model` in an already-open session, restart Claude Code once.

---

## Part B — PowerShell profile: point Claude Code at DeepSeek

### Find your profile path(s)

Claude Code inherits env vars from the shell that launches it. On Windows there can be two
profiles — apply the block to whichever shells you launch `claude` from:

```powershell
# Inside each shell, this prints where ITS profile lives:
echo $PROFILE
```

Typical locations (Documents may be OneDrive-redirected, as on this machine):

- Windows PowerShell 5.1: `C:\Users\<you>\OneDrive\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1`
- PowerShell 7 (pwsh):     `C:\Users\<you>\OneDrive\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`

Create the file (and folder) if it doesn't exist.

### Option 1 — Minimal (recommended): your original file with two lines changed

If you switch models via `/model` only, you don't need any launcher functions. This version
is **exactly the profile as originally written on this machine**, with one fix: the two tier
lines that pointed Opus/Sonnet-tier requests at **pro** now point at **flash**.

Original, for reference (do NOT copy — this is the version that billed pro):

```powershell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="YOUR_DEEPSEEK_API_KEY"
$env:ANTHROPIC_MODEL="deepseek-v4-flash"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"      # ← leak
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"    # ← leak
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_EFFORT_LEVEL="max"
$env:CLAUDE_CODE_AUTO_COMPACT_WINDOW="786432"
```

Fixed — copy this one. The only differences are those two lines:

```powershell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="YOUR_DEEPSEEK_API_KEY"
$env:ANTHROPIC_MODEL="deepseek-v4-flash"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-flash"        # was pro[1m]
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-flash"      # was pro[1m]
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_EFFORT_LEVEL="max"
$env:CLAUDE_CODE_AUTO_COMPACT_WINDOW="786432"
```

Why those two lines must change (the whole reason for this guide): Claude Code makes some
internal requests by "tier" (opus/sonnet) rather than by your chosen model. The original
pointed every opus- or sonnet-tier request at `deepseek-v4-pro[1m]`, so those requests billed
pro even though the main model was flash. **Don't delete the two lines either** — an unpinned
tier request goes out as a literal `opus`/`sonnet` id and relies on DeepSeek's fallback
behavior. Keep all four tier lines, all pointing at flash. Everything else in the file stays
exactly as you originally had it.

### Option 2 — Launcher functions (optional)

Only needed if you want **whole-session** switches, where subagents are on the chosen model
too (e.g. agents that themselves must see images, or a pro session where Explore/Plan also
run pro). `/model` alone switches only the main conversation model — subagents and
background tasks stay on flash regardless. If that's fine for you, skip this section entirely.

```powershell
# Opt-in: run a whole session on pro (main model, agents, sonnet/opus-tier
# requests). Background haiku-tier chores stay on flash.
function claude-pro {
    $env:ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
    $env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
    $env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
    $env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-pro[1m]"
    claude @args
}

# Opt-in: vision session (deepseek-v4-flash-vision-exp). Priced the same as
# flash but can see images — text-only flash/pro silently ignore image blocks,
# so put EVERY tier on the vision model for the session.
function claude-vision {
    $env:ANTHROPIC_MODEL="deepseek-v4-flash-vision-exp"
    $env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-flash-vision-exp"
    $env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-flash-vision-exp"
    $env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash-vision-exp"
    $env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash-vision-exp"
    claude @args
}
```

In a new terminal: `claude-pro` / `claude-vision` launch whole sessions on that model;
plain `claude` always starts on flash.

Model ids in current use (Sept 2026): `deepseek-v4-flash`, `deepseek-v4-pro`,
`deepseek-v4-pro[1m]`, `deepseek-v4-flash-vision-exp`. The `[1m]` = 1M-context variant.
Vision (`deepseek-v4-flash-vision-exp`) is **separate** — flash and pro are text-only and
silently ignore image blocks.

---

## Part C — Verify on the laptop

In a **new** terminal (profiles load only at shell startup):

```powershell
# 1. Env is live?
echo $env:ANTHROPIC_BASE_URL     # https://api.deepseek.com/anthropic
echo $env:ANTHROPIC_MODEL        # deepseek-v4-flash
echo $env:ANTHROPIC_DEFAULT_SONNET_MODEL   # deepseek-v4-flash  (NOT pro!)

# 2. Launcher functions exist? (skip if you chose Option 1 only)
Get-Command claude-vision, claude-pro

# 3. Claude Code sees it
claude --version                 # >= 2.1.243
```

Then inside a Claude Code session:
1. `/model` → you should see "DeepSeek V4 Flash / Pro / Flash Vision" rows only.
2. Pick **DeepSeek V4 Flash Vision** with `s` (session-only) → paste/screenshot an image →
   it should describe it. If it can't, the session is running flash text-only (check the
   status line / `/context` model).
3. `/model` → **DeepSeek V4 Pro** with `s` → confirm via the status line. Background tiers
   stay flash by design — `/model` switches only the main conversation model.
4. Exit and run plain `claude` → always starts on flash again (`ANTHROPIC_MODEL` env beats any
   `/model` "save as default").

---

## Optional housekeeping (this machine also had)

A stale **"Fable"** row (a real Claude model from an earlier Anthropic-API period) may linger
in `/model`. It lives in `~/.claude.json` under `additionalModelOptionsCache` — internal cache,
**not** a supported config surface. Clicking it is harmless (unknown id → DeepSeek falls back to
flash). To remove it: close ALL Claude Code sessions, delete the entry from the JSON array,
restart. Don't edit that file while a session is running — Claude Code rewrites it and will
clobber your edit.
