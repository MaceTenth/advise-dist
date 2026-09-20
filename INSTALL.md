# Installing tg for a teammate

tg turns finished Claude Code sessions into shared memory: when a session ends it is condensed, redacted and pushed
to the team's cloud; the next session starts with a brief of what the team learned in that project, and (optionally)
gets prior fixes injected when a command fails. Ten minutes, five commands.

## What you need
- macOS or Linux, Claude Code, and Node 18+ (for `npx`). Python is only needed for the from-source route.
- From the operator: the team **API URL** and the **admin key** (the value of the `tg/admin` secret). Send it over a
  channel you trust; it is only used once, to register your device.

## 1 · Get `tg`

**With Node (recommended, no Python needed):**
```bash
npx tg-memory@latest doctor        # first run downloads the prebuilt binary for your platform (~9 MB), verifies its checksum, caches it in ~/.tg/bin
alias tg='npx tg-memory'           # or: npm i -g tg-memory
```
Prebuilt for macOS (Apple Silicon and Intel) and Linux (x64 and arm64). The binary contains no source; the source
repository stays private.

**From source (needs Python 3.11+ and access to the private repository):**
```bash
git clone https://github.com/MaceTenth/trajectory-graph.git ~/trajectory_graph
cd ~/trajectory_graph && python3 -m venv .venv && .venv/bin/pip install -e .
ln -sf ~/trajectory_graph/.venv/bin/tg /usr/local/bin/tg
```

## 2 · Register this device
```bash
tg register --api https://<api-id>.execute-api.<region>.amazonaws.com --admin-key '<admin key>' --user <your.name>
tg status
```
This writes `~/.config/tg/config.json` (mode 600) with a device token. The admin key is not stored.

## 3 · Consent, then push your history (optional, recommended)
```bash
tg contribute on      # consent switch; until this is on, nothing is ever pushed (unset means off)
tg push --dry-run     # shows what would be uploaded: condensed, redacted turns; never raw transcripts
tg push               # interesting turns only (tool errors or 6+ tool calls); add --all for everything
```
`tg contribute off` stops pushing at any time; briefs and lookups keep working. `tg erase --me` deletes everything you
ever pushed from the team memory and rebuilds the cards it contributed to.
Pushing 200 sessions takes a few minutes; extraction in the cloud costs about $0.014 per turn and finishes within a
minute per turn. Only turns with tool work are pushed; a record in `~/.local/share/tg/pushed.json` prevents re-sends.

## 4 · Wire the hooks into Claude Code
```bash
tg install --no-recall    # Stop (push every session) + SessionStart (project brief); recall on failures stays off
tg install                # same, plus PostToolUse recall (inject prior fixes when a tool call fails)
```
This edits `~/.claude/settings.json`, keeps everything already in it, and writes a backup next to it. Every hook
fails open: if the cloud is slow or down, the session simply proceeds without memory. `tg install --uninstall` removes
exactly what tg added. Use `--project <dir>` to install for one repository instead of user-wide.

## 5 · Verify
```bash
tg doctor             # config, API, hooks, last push, python version, and a live recall
tg dashboard --open   # the team memory dashboard (activity, known failures, what memory did)
```
Start a Claude Code session anywhere: the brief (if the project has history) appears as hook context at the start, and
`~/.local/share/tg/hooks.log` gets one line per hook event, including each push. Only *interesting* turns are pushed
(a tool error, or six or more tool calls), so a trivial session logs `0 episodes to push`; that is the filter working.

## What leaves your machine, and what does not
- Sent: for each turn, the user's request text and one line per tool call with a truncated result (about 260 + 160
  characters each), after regex redaction of keys, tokens and passwords. Plus token counts, model name and duration.
- Never sent: raw transcripts, file contents beyond those snippets, thinking, screenshots, attachments, environment
  variables, or anything from files you did not touch in the session.
- Stored: in the team's own AWS account, encrypted; episodes expire after 90 days, extracted trajectories stay.
- The operator can delete a user's data (`service/README.md`, "forget a user's data").

## Troubleshooting
| symptom | cause / fix |
|---|---|
| `not registered with a tg cloud` on install | run step 2 first, or `tg install --force` to install hooks that will start working once registered |
| `tg doctor` says API unreachable | first call after 15 idle minutes wakes the database; wait a minute and retry |
| brief never appears | the project has no trajectories yet: it appears from the second session on; check `hooks.log` for `SessionStart` lines |
| a hook seems slow | recall takes 3–5 s warm, up to a minute cold; sessions are not blocked, the context just arrives late |
| you want memory off for one session | `TG_MODE=local claude` (hooks read local mode as "do nothing remote") |
| you want your data gone | `tg erase --me`; the operator can do the same for a departed teammate |
