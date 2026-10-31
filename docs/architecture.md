# Architecture

## Data flow

1. **Ingest.** `copilot/slack_reader.py` reads the channel history. Each message is sent once to Nemotron Nano, which returns deadlines, decisions and feedback as JSON. A table of processed message hashes makes later runs incremental, and a failed extraction is retried next time.
2. **Memory.** `copilot/memory.py` stores items in SQLite with a status (active, done, cancelled, superseded). A new deadline for a topic supersedes the old one and keeps the previous date for "moved from" notes.
3. **Plan.** `copilot/planner.py` builds the due list, recent changes and the next build from memory. It uses no LLM.
4. **Notebooks.** `copilot/notebook/` turns an outline into cells. Routine cells use Nano and code cells use Super, with a per-section cache so a crash does not repeat paid calls. Validation by execution is opt-in.
5. **Drafts and approval.** `copilot/approvals.py` holds drafts. A draft moves from pending to approved or discarded exactly once, and every step is logged.

## Approval gate

The Hermes plugin (`agent/plugins/content-copilot/core.py`) registers the `copilot_*` tools and two hooks.

- `copilot_approve` runs only with a fresh, single-use Approve answer recorded from a real clarify result. The result must be answered, name exactly one draft, and the question shown to the person must contain the exact draft text.
- The pre-tool hook blocks direct Slack posting, `hermes send` and shell calls to the approve command.
- The post-tool hook writes a redacted JSON line per call to `outputs/tool_audit.jsonl`, including why a clarify answer was or was not accepted.

## Sandbox

NemoClaw runs Hermes inside an OpenShell sandbox. The filesystem is read-only except `/sandbox` and `/tmp`, and outbound traffic is limited to an allow list. Model calls go through `inference.local`, where the gateway holds the real API key. The copilot detects that endpoint and uses a placeholder key. Slack is not on the allow list in the demo sandbox, so a post attempt there is blocked at the network level and the draft stays pending.

Dependencies inside the sandbox are installed with `pip install --target /sandbox/pylibs`, and the plugin adds that folder to the path for its CLI calls.

## Failure behavior

- If the gateway is down, Slack messages sent in that window are lost. The service restarts with WSL and reconnects within seconds.
- If Slack is unreachable, approve reports "NOT posted" and the draft stays pending.
- If the model returns unusable JSON, the message is not marked processed and is retried.
