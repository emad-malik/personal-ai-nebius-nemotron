# Content Dev Copilot

A private, always-on AI copilot for bootcamp content development. It reads a team Slack channel, remembers deadlines, decisions and reviewer feedback, plans the day, builds lecture notebooks, and drafts channel updates. It never posts to the channel without a human pressing Approve.

## What it does

- **Remembers.** Extracts deadlines, decisions and feedback from the channel into SQLite. A new date for the same topic supersedes the old one, and every item keeps its source message.
- **Plans.** Builds the daily plan from memory with plain code, no LLM call, so the due list is exact.
- **Builds notebooks.** Turns a topic into a teaching notebook (`.ipynb`) from an outline, and applies remembered decisions such as "Titanic only" or "matplotlib only".
- **Drafts, never posts alone.** Status updates and messages become drafts. A person approves, edits or discards them with buttons. Only an Approve answer lets a post go out.
- **Stays on.** Runs as a service, answers in Slack DMs and channel mentions, and sends the daily plan at 6:00 PM.

## Stack

| Layer | Tool |
|-------|------|
| Models | NVIDIA Nemotron 3 (Nano 30B for routine extraction, Super 120B for planning and code) via Nebius Token Factory |
| Agent runtime | Hermes Agent, Slack gateway (Socket Mode), cron |
| Sandbox | NVIDIA NemoClaw / OpenShell |
| Memory | SQLite |
| Notebooks | nbformat + nbclient |

## How the approval gate works

The agent is not trusted to behave. Three layers enforce it in code.

1. **Tools, not shell.** The agent posts only through `copilot_*` tools (`copilot_draft_status`, `copilot_draft_post`, `copilot_approve`, `copilot_discard`, `copilot_edit`, `copilot_plan`, `copilot_refresh_memory`), provided by the Hermes plugin in `agent/plugins/content-copilot/`.
2. **Approve needs a real button press.** `copilot_approve` is refused unless the person just answered Approve in a clarify question that showed the exact draft text. The grant is single use and expires after 15 minutes. Editing a draft clears it.
3. **Other routes are blocked.** A pre-tool hook rejects shell commands and calls that would post some other way (Slack API calls, `hermes send`, calling the approve command directly).

Every tool call, including blocked ones, is written to `outputs/tool_audit.jsonl`. The approval queue and its decisions are in the SQLite `approvals` and `audit_log` tables.

```
Slack channel --> ingest (Nano) --> SQLite memory --> plan / notebooks
                                        |
Hermes (Slack gateway) --> copilot_* tools --> draft --> Approve / Edit / Discard
                                                            |
                                              only Approve posts to the channel
```

For a stricter setup, the same plugin runs inside an OpenShell sandbox where outbound network access is limited to an allow list, so even a bypass of the plugin cannot reach Slack. See `docs/architecture.md`.

## Quick start

```bash
git clone https://github.com/emad-malik/personal-ai.git
cd personal-ai
uv sync            # or: python -m venv .venv && pip install -e .
cp .env.example .env
# fill in NEBIUS_API_KEY and the Slack values

uv run python -m copilot.llm                       # check the model connection
uv run python -m copilot.cli ingest --seed --reset # load the demo channel
uv run python -m copilot.cli plan --today 2026-10-14
uv run pytest -q -k "not nano_responds"            # offline tests
```

Other commands: `build`, `build-next`, `send-file`, `draft-status`, `draft-post`, `approvals`, `audit`, `tool-audit`, `memory`. Run `python -m copilot.cli --help`.

## Run it with Hermes

```bash
ln -s "$PWD/agent/plugins/content-copilot" ~/.hermes/plugins/content-copilot
for s in agent/skills/*; do ln -s "$PWD/$s" ~/.hermes/skills/$(basename "$s"); done
hermes plugins enable content-copilot
hermes gateway
```

Create the Slack app from `demo/slack_app_manifest.yaml`, then put the bot and app tokens, the channel id and `SLACK_ALLOWED_USERS` (the people who may approve) in `.env`. The skills are `daily-plan`, `lecture-notebook` and `channel-update`.

## Always on

On Windows with WSL2, run the gateway as a user service and keep WSL alive.

1. `sudo loginctl enable-linger $USER` so the service survives logout (Hermes installs `hermes-gateway.service`).
2. Stop Windows sleeping on mains power: `powercfg /change standby-timeout-ac 0`.
3. Keep WSL from stopping when idle with a hidden login task that runs `wsl.exe -d <distro> --exec sleep infinity`.
4. Schedule the daily plan from the DM so delivery goes there: `hermes cron list` shows the job.

Limits: the PC must be on and logged in once after a reboot, and Slack events sent while the gateway is down are not replayed.

## Project layout

```
copilot/          core modules: llm, slack_reader, memory, planner, approvals, notebook/, cli
agent/
  plugins/content-copilot/   Hermes plugin: gated tools, block rules, tool audit trail
  skills/                    daily-plan, lecture-notebook, channel-update
  openshell/policy.yaml      sandbox policy notes
demo/             seed channel messages, expected plan, Slack app manifest
tests/            offline tests for memory, planner, approvals, plugin gate
docs/             architecture and tool feedback log
```

## License

MIT
