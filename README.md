DEAD RECKONING

A single-file countdown tracker for a fixed-deadline work sprint. One HTML file, no build step, no dependencies, no backend — state syncs across devices through a secret GitHub Gist.

Dead reckoning is navigation by known speed and elapsed time with no landmarks to check against. That is more or less what a seven-week run at an immovable date feels like.

Live: `[https://<username>.github.io/<repo>/](https://notanotherannie.github.io/Tracker/)`

---

## What it does

Three tabs over one shared state object.

**WORK** — a timeline and a set of strands. Each strand is a named workstream holding dated tasks. The timeline draws each strand as one or more horizontal bars spanning its contiguous date runs, filled by completion percentage, with a glowing vertical line marking today. Gaps longer than eight days split a strand into separate bars, so a workstream that pauses for a fortnight looks paused rather than continuous.

**COLLEGE** — an application shortlist. Static programme data (deadlines, tuition, portfolio and language requirements, post-study visa terms) ships baked into the file; your status, priority, and notes per programme are the mutable part. A REQUIREMENT FLOOR panel computes the strictest demands across the whole list, so you know what to aim at rather than what any single school asks.

**RECORD** — an append-only change log. Every mutation writes an entry. Also holds export (JSON and Markdown), import, reset, and the two sync controls.

The masthead shows days remaining, percentage complete, open task count, and a seven-week progress bar. The status strip adds overdue count, applications submitted, and sync state.

---

## Setup

The file works immediately with no configuration — it just stays on one device. To sync:

### 1 · Make a token

GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate new token.

- **Account permissions → Gists: Read and write**
- **Repository permissions: leave everything at No access**
- Set a long expiry. When it lapses you'll see `PUSH FAILED` in the status strip.

### 2 · First device

Open the site → RECORD → **SYNC SETUP**. Paste the token. Leave the gist ID **blank**. A secret gist is created and its ID shown in an alert — copy that.

### 3 · Every other device

RECORD → **SYNC SETUP** → same token, and the gist ID from step 2.

The SYNC field in the status strip should show a timestamp after your next edit.

---

## How sync works

| | |
|---|---|
| **Store** | One secret gist, one file: `deadreckoning.json` |
| **Upload** | Inside `save()`, which every mutation already calls. Debounced 1200 ms — ticking six boxes quickly makes one request, not six. |
| **Download** | On page load, on `window` focus, and on **SYNC NOW** |
| **Conflict** | Last-write-wins on a `rev` timestamp. No merging. |
| **Local cache** | `localStorage` is still written synchronously on every save, so the app is fully usable offline and instant on load. |

Credentials live under a separate `localStorage` key (`term49b_cfg`) from tracker state (`term49b`), so export, import, and reset never touch them.

**The buttons are not the sync.** SYNC SETUP is one-time configuration; SYNC NOW is a manual pull for when you don't trust that the focus-triggered one fired. Uploads and downloads are automatic. If you'd rather drop both buttons, set the config once from the console instead:

```js
localStorage.setItem('term49b_cfg', JSON.stringify({token:'ghp_...', gist:'abc123...'}))
```

### Reading the SYNC field

| Value | Meaning |
|---|---|
| `LOCAL ONLY` | No token on this device — localStorage only |
| `LOADING` / `SAVING` | Request in flight |
| `14:32` | Last successful push or pull |
| `PULLED 14:32` | Remote was newer; local state was replaced |
| `PUSH FAILED` / `PULL FAILED` | Expired token, offline, or rate-limited |
| `CREATE FAILED` | Token lacks Gists read+write |

---

## Security

Read this part properly.

**The token is stored client-side, in plain text, in that browser's localStorage.** Anyone with access to the device can read it. Scoping it to Gists-only means the worst case is someone reading and editing this tracker — not touching your repositories. Revoke it from GitHub settings if a device is lost.

**A public Pages repo means public seed data.** GitHub Pages from a private repo requires a paid plan. If your repo is public, everything hardcoded in `SEED()` — every task, every programme, every note written into the source — is readable by anyone who finds the URL. The *synced* data is in a secret gist and stays private; the seed baked into the HTML does not. Either strip `SEED()` down to empty strands before pushing, or go private.

"Secret gist" means unlisted, not access-controlled. Anyone with the ID can read it.

**Never commit the token.** It's only ever entered at runtime and never appears in source.

---

## Editing the file

Everything is in `index.html`. Rough map:

| Region | What lives there |
|---|---|
| `<style>` | One accent colour at four opacity levels. `--p` is the accent, `--l2/--l3/--l4` are its fades. Change `--p` and `--bg` to retheme the whole thing. |
| `COLLEGES` | Static programme array. Add entries here. |
| `SEED()` | Initial strands and tasks. Only used on first run or after RESET. |
| Storage layer | `save()`, `pushCloud()`, `pullCloud()`, `makeGist()`, `refresh()` |
| `render()` | Single redraw path — `head()`, `mapv()`, `strands()`, `college()`, `logv()`. Every mutation calls it. |

Editing `SEED()` after you've used the app changes nothing on its own — seed data only applies on first run. Use RESET (destructive) or edit tasks in the UI.

`START` and `END` at the top of the script set the window. The seven-week bar assumes a 49-day span; a different span still works but the week ticks will sit oddly.

---

## Deploying

Push `index.html` to the repo root, then Settings → Pages → deploy from branch → root. Nothing to build.

Cache note: Pages serves aggressively cached assets. After pushing a change, hard-reload (Ctrl/Cmd+Shift+R) or you may sit on the old file for a while.

---

## Limitations

- No merge. Two devices edited simultaneously means the later save wins outright.
- No history or undo beyond the change log, which records events but can't replay them.
- Gist writes count against GitHub's API rate limit. Normal use won't approach it; a script hammering `save()` would.
- Export often anyway. `EXPORT JSON` gives a restorable file; `EXPORT MD` gives a readable snapshot.
