# Spiral CLI reference

Spiral (`@every-env/spiral-cli`) drafts copy in a user's brand voice. `ce-promote` uses it as an **optional enhancement** — every call must be wrapped so a missing, unauthed, or erroring CLI never blocks the skill.

## Detection — three states

```bash
which spiral
spiral auth status 2>/dev/null
```

- **Ready** — binary exists AND auth status prints a key like `spiral_sk_...`. Use Path A.
- **Unauthed** — binary exists, no key printed (or an unauthenticated message, exit non-zero). → Path 0 (offer to connect).
- **Absent** — `which spiral` finds nothing. → Path 0 (offer to install + connect).

Any error or timeout → treat as not-ready and continue; never block.

## Path 0 — Offer setup (first run, declinable)

When Spiral is unauthed or absent, offer setup once. First check the opt-out so this never nags.

### Check the opt-out

Read the project config (resolve the repo root, never CWD):

```bash
cat "$(git rev-parse --show-toplevel 2>/dev/null)/.compound-engineering/config.local.yaml" 2>/dev/null || echo '__NO_CONFIG__'
```

If the contents include `ce_promote_spiral_optout: true`, **skip Path 0** and go straight to Path B. Otherwise, offer setup.

### Ask

Use the platform's blocking-question tool: `AskUserQuestion` in Claude Code (call `ToolSearch` with `select:AskUserQuestion` first if its schema isn't loaded), `request_user_input` in Codex, `ask_user` in Gemini / Pi. If no blocking tool exists or the call errors, present the same options as a numbered list in chat and wait for a reply — never silently skip.

For the **unauthed** state, prefer the lightest path: run `spiral login` and let the user authenticate by opening the link it prints — don't make them hunt for a pairing code. The blocking question is mainly the escape hatch.

Use the question stem to teach the mechanic, offer the escape hatch, AND disclose that declining is durable (so the permanent side effect isn't hidden behind a transient-sounding label): "Spiral personalizes and humanizes the copy in your voice. [It's installed but not signed in / It isn't installed yet] — sign in now, or have the agent draft directly without Spiral? (Declining drafts your copy now and won't bring up Spiral again in this project; you can set it up anytime by asking.)"

Offer exactly **two** options (labels must be self-contained):

- **Unauthed** state: `Sign in to Spiral` · `Draft directly without Spiral`
- **Absent** state: `Install Spiral` · `Draft directly without Spiral`

There is deliberately no separate "don't ask again" option: **dismissing is itself the opt-out.** A single first-run decline records the flag and the offer never recurs in this repo. This is what keeps a per-ship skill from nagging — never make the user choose a special variant to stop being asked.

### Act on the choice

- **Sign in to Spiral** (installed, unauthed) — run `spiral login`. It prints a link to create an API key:
  ```
  API key required. Usage: spiral auth login --token <spiral_sk_...>
    Create a key at: https://app.writewithspiral.com/settings/api-keys
  ```
  Surface that link and ask the user to create a key and run `spiral auth login --token spiral_sk_...` **themselves, in their own terminal**. **Never have the user paste the API key into the chat, and never run the login command with their key as an argument** — an API key in the transcript is exposed to the model, tool logs, and any session-history tooling. The user authenticates on their own; the agent only re-runs `spiral auth status` afterward to confirm. If a `spiral_sk_...` key now appears, proceed with Path A; otherwise fall to Path B. (In v1.6.1 `spiral login` is the create-a-key link flow, not a one-click browser approve.)
- **Install Spiral** (absent) — the pairing-code command installs and connects in one step. Direct the user to Settings → Connect an Agent at https://app.writewithspiral.com to copy their command, which looks like:
  ```bash
  npx @every-env/spiral-cli@latest setup --pairing-code <code>
  ```
  The pairing code is single-use and expires in ~15 minutes, so the user must fetch a fresh one from the web app — do not hardcode it. Once installed, if still unauthed, run `spiral login` and follow the link flow above. If the user can't or won't install, go to Path B.
- **Draft directly without Spiral** — record the opt-out (below) so the offer never re-prompts in this repo, then go to Path B. (A failed/abandoned **sign-in or install** attempt does NOT record the opt-out — only an explicit "draft directly" dismissal does — so a user whose auth didn't complete still gets one clean re-offer next run.)

### Record the opt-out (best-effort)

Resolve the repo root, then add `ce_promote_spiral_optout: true` as a top-level key to `<root>/.compound-engineering/config.local.yaml`, using the native file-write/edit tool:

- **File already exists:** append the key if it isn't already present.
- **File absent:** create it (and its `.compound-engineering/` directory) with the key, AND ensure the machine-local config is gitignored — mirroring `ce-setup`, the canonical creator of this file. Check whether the path is already ignored (`git check-ignore -q .compound-engineering/config.local.yaml`); if it isn't, append `.compound-engineering/*.local.yaml` to the repo-root `.gitignore`. Without this, a user who runs `/ce-promote` before `/ce-setup` ends up with an un-ignored config holding machine-local opt-out state that can get committed by accident.

If the root can't be resolved or any write fails, proceed to Path B anyway; the opt-out is a convenience, never a blocker.

After recording, confirm it in one line so the write isn't silent and the user knows how to undo it — e.g. "Got it — I won't bring up Spiral here again (saved to `.compound-engineering/config.local.yaml`, gitignored). Want it back later? Just ask, or remove the `ce_promote_spiral_optout` key." Keep it to a single line; don't belabor it.

## Generate

```bash
spiral write "<prompt>" --instant --num-drafts <1-5> --json
```

- `--instant` — skip clarifying questions. **Always use it**; this is a headless context with no human mid-call.
- `--json` — machine-readable output. Always use it.
- `--num-drafts <1-5>` — number of drafts (single-channel mode only; see gotcha).
- `--workspace <uuid>` — scope to a brand-voice workspace. List with `spiral workspaces`. Use only if the user names one.
- `--style <uuid>` — pin a specific voice/style. Use only if the user names one.

### Output shape

JSON with (verified against CLI v1.6.1):

```json
{
  "session_id": "uuid",
  "status": "complete | needs_input",
  "drafts": [
    { "id": "uuid", "title": "...", "content": "markdown", "channel": "x",
      "url": "https://app.writewithspiral.com/chat/<session>?draft=<id>", "display_hint": "inline | expandable" }
  ],
  "text": "pipeline commentary — DO NOT show the user unless drafts is empty",
  "style_used": null,
  "quota_remaining": 42
}
```

- `channel` (lowercase) is one of `x`, `linkedin`, `email`, `newsletter`, `blog`, `instagram_tiktok`, `research`, or `null`.
- `url` opens that draft in the Spiral web app for editing. Drafts persist to the user's account — surface `session_id` + each `url` in your output (Phase 4).
- **Do not surface the `text` field** to the user — it's internal pipeline commentary. Only fall back to it if `drafts` is empty.
- With `--instant`, `status` should be `complete`. If it comes back `needs_input` (rare with `--instant`), don't relay Spiral's questions to the user — either answer from the context you already have via a `--session` follow-up, or fall back to Path B for that channel.

If parsing fails or `drafts` is empty, fall back to direct drafting for the affected channels.

## The multi-channel / cue-word gotcha (important)

Multi-channel output is **phrasing-driven, not a flag.** Spiral enters "campaign mode" when the prompt contains **≥2 channel keywords** (tweet/X, LinkedIn, email, blog, …) **OR** any cue word: `campaign`, `across`, `multi-channel`, `everywhere`, `cross-post`.

Two consequences to encode:

### (a) To get N variations of ONE channel

Ask for `"3 tweet options for <feature>"` and:

- **Avoid** the cue words above. Ironically, a prompt literally containing `campaign` or `multi-channel` trips campaign mode — so describe the task **without** those words.
- Pass `--num-drafts 3`.

If you accidentally include a cue word, Spiral decides it's a single campaign piece and returns **1 draft**, ignoring `--num-drafts`.

✅ `spiral write "3 tweet options for one-click CSV export" --instant --num-drafts 3 --json`
❌ `spiral write "a tweet campaign for CSV export" --instant --num-drafts 3 --json`  (collapses to 1 draft)

### (b) To get a real multi-channel set

Phrase the prompt with the multiple channels named. Spiral returns **one set of drafts per channel**, each draft carrying its `channel`. In this mode **`--num-drafts` is ignored** — per-channel counts apply.

✅ `spiral write "announcing one-click CSV export — a tweet and a LinkedIn post" --instant --json`
✅ `spiral write "a campaign across email, LinkedIn, and Twitter for CSV export" --instant --json`

This one-call cross-channel set is the ideal fit for `ce-promote` when the user wants to announce across surfaces.

**Spiral picks per-channel counts itself.** In campaign mode the count per channel is Spiral's call, not yours — e.g. "a tweet and a LinkedIn post" (verified, v1.6.1) returned 3 X drafts + 2 LinkedIn drafts (5 total), each tagged with its `channel`. Group the returned `drafts` by `channel` for Phase 4; don't assume one per channel.

## Failure handling

Detection that comes back not-ready routes through Path 0 above. Once on Path A, any of these → fall back to direct drafting (SKILL.md Path B), silently, for the affected channels:

- `spiral write` exits non-zero, hangs, or emits non-JSON
- `drafts` is empty or missing expected fields

Never surface raw Spiral errors to the user as a blocker. The skill always produces drafts.
