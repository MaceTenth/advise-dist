# Installing advise for a teammate

advise turns finished Claude Code sessions into shared memory: when a session ends it is condensed, redacted and pushed
to the team's cloud; the next session starts with a brief of what the team learned in that project, and (optionally)
gets prior fixes injected when a command fails. Ten minutes, five commands.

## What you need
- macOS or Linux, Claude Code, Node 18+ (for `npx`), and a GitHub account. Python is only needed for the from-source route.
- A beta seat: the operator approves GitHub logins (`advise admin approve --login <you>`). If you sign up before that,
  your request is recorded and `advise signup` succeeds once you are approved.
- Joining an existing team instead: a one-time **invite code** from the team owner (`advise admin invite`), then step 2b.

## 1 · Get `advise`

**With Node (recommended, no Python needed):**
```bash
npx advise-cli@latest doctor        # first run downloads the prebuilt binary for your platform (~9 MB), verifies its checksum, caches it in ~/.advise/bin
alias advise='npx advise-cli'           # or: npm i -g advise-cli
```
Prebuilt for macOS (Apple Silicon and Intel) and Linux (x64 and arm64). The binary contains no source; the source
repository stays private.

**From source (needs Python 3.11+ and access to the private repository):**
```bash
git clone https://github.com/MaceTenth/trajectory-graph.git ~/trajectory_graph
cd ~/trajectory_graph && python3 -m venv .venv && .venv/bin/pip install -e .
ln -sf ~/trajectory_graph/.venv/bin/advise /usr/local/bin/advise
```

## 2 · Sign in
**2a · Beta tester (your own tenant):**
```bash
advise signup            # shows the beta terms, then a one-time code to enter at github.com/login/device
advise status
```
Sign-in uses GitHub's device flow with a public client id: advise reads your login and email, never your repositories,
and the GitHub token is exchanged once for an advise device token, then discarded. Approved logins get a personal tenant
`u/<login>` and an owner device; the token lands in `~/.config/advise/config.json` (mode 600) and expires after 180 days
(run `advise signup` again to renew). Not approved yet: you see "access requested"; nothing else happens.

**2b · Joining a team with an invite code:**
```bash
advise register --api https://<api-id>.execute-api.<region>.amazonaws.com --invite <code> --user <your.name> --accept-terms
```
The invite is single-use and bound to the team's tenant.

## 3 · Consent, then push your history (optional, recommended)
```bash
advise contribute on      # consent switch; until this is on, nothing is ever pushed (unset means off)
advise push --dry-run     # shows what would be uploaded: condensed, redacted turns; never raw transcripts
advise push               # interesting turns only (tool errors or 6+ tool calls); add --all for everything
```
`advise contribute off` stops pushing at any time; briefs and lookups keep working. `advise erase --me` deletes everything you
ever pushed from the team memory and rebuilds the cards it contributed to.
Pushing 200 sessions takes a few minutes; extraction in the cloud costs about $0.014 per turn and finishes within a
minute per turn. Only turns with tool work are pushed; a record in `~/.local/share/advise/pushed.json` prevents re-sends.

## 4 · Wire the hooks into Claude Code
```bash
advise install                    # Stop (push), SessionStart (project brief), UserPromptSubmit (similar past tasks, per prompt), PostToolUse (recall on failures)
advise install --no-recall        # leave failure-time recall off
advise install --no-prompt-brief  # leave the per-prompt brief off
```
Every injection starts with the same guardrail line marking it as recorded data, not instructions; the agent is told to
verify anything it uses against the repository. `advise brief "<prompt>"` prints what the per-prompt hook would inject, for
agents without hooks (Codex, scripts): prepend it to the prompt.
This edits `~/.claude/settings.json`, keeps everything already in it, and writes a backup next to it. Every hook
fails open: if the cloud is slow or down, the session simply proceeds without memory. `advise install --uninstall` removes
exactly what advise added. Use `--project <dir>` to install for one repository instead of user-wide.

## 5 · Verify
```bash
advise doctor             # config, API, hooks, last push, python version, and a live recall
advise dashboard --open   # the team memory dashboard (activity, known failures, what memory did)
```
Start a Claude Code session anywhere: the brief (if the project has history) appears as hook context at the start, and
`~/.local/share/advise/hooks.log` gets one line per hook event, including each push. Only *interesting* turns are pushed
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
| `not registered with a advise cloud` on install | run step 2 first, or `advise install --force` to install hooks that will start working once registered |
| `advise doctor` says API unreachable | first call after 15 idle minutes wakes the database; wait a minute and retry |
| brief never appears | the project has no trajectories yet: it appears from the second session on; check `hooks.log` for `SessionStart` lines |
| prompt brief never appears | it fires only for prompts that look like a task (12+ characters, not an acknowledgement or a slash command) and only for trajectories not yet shown in the session; `hooks.log` has a `UserPromptSubmit` line per prompt |
| a hook seems slow | recall takes 3–5 s warm, up to a minute cold; sessions are not blocked, the context just arrives late |
| you want memory off for one session | `ADVISE_MODE=local claude` (hooks read local mode as "do nothing remote") |
| you want your data gone | `advise erase --me`; the operator can do the same for a departed teammate |
