# Feedback Log

Notes on the tools used while building this project.

## Token Factory

| Date | Issue / Feedback | Resolution |
|------|-----------------|------------|
| Oct 5 | Setup: Nano (30B-A3B) and Super (120b-a12b) both respond on Token Factory. OpenAI-compatible client works out of the box. Nano model ID needed casing fix (nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B). | Working. |
| Oct 16 | Nemotron returns its reasoning in separate `reasoning` and `reasoning_content` fields, and a small `max_tokens` (we used 128) can be spent before any answer, so `content` comes back empty with no error and `finish_reason` may look normal. Cost us a debugging round. | Use 1024+ for simple calls and 4096 for extraction. Request: document this and consider a clear signal when the budget ends inside the reasoning. |


## Hermes Agent

| Date | Issue / Feedback | Resolution |
|------|-----------------|------------|
| Oct 14 | Hermes `hermes model` custom endpoint wizard verified Token Factory via /v1/models (25 models listed) and a first chat with Super worked. Model list is useful, but Nemotron IDs have inconsistent casing across models (NVIDIA-Nemotron-3-Nano-30B-A3B vs nemotron-3-super-120b-a12b). | Worked. Copy IDs from the wizard list. |
| Oct 14 | Hermes gateway deliverable mode uploaded a .csv from a plain-text path in the reply but not a .ipynb (path stayed as text). The docs do not list which extensions are shipped or how to add one. | Workaround: own uploader using Slack files_upload_v2. Request: a config list of extra allowed extensions. |
| Oct 14 | `/hermes sethome` was rejected by Slack as an unknown slash command in our workspace, so the channel could not be set as the home channel that way. | Workaround: set `SLACK_HOME_CHANNEL` in the environment. Request: document this path next to the slash command. |
| Oct 14 | The agent ignored our `channel-update` skill and improvised a status update with open-ended questions, so a skill is advice and not enforcement. | Built a plugin with tools, a pre-tool block hook and a post-tool audit hook. The hooks worked well. Request: document the hook payloads and the exact `clarify` result shape (`responses[]` with `question`, `choices_offered`, `user_response`, `status`), which we had to discover by logging. |
| Oct 16 | `clarify` choices are free text the model controls. The model added "(Recommended)" to a button label, which broke exact matching of the answer. | Normalized the label in our gate. Request: a way to mark a recommended option without changing the returned value. |
| Oct 16 | `hermes gateway restart` waits for in-flight turns (up to about 30 minutes by default) and one long tool call held it for minutes. | Made the slow tool incremental. Works as designed, though a visible timeout hint in the CLI would help. |
| Oct 31 | After a gateway restart the Slack Socket Mode link logged "transport disconnected; reconnecting" within about a minute, and events sent while the gateway was down or reconnecting were not replayed, so early messages got no reply. | Documented the limit. Request: replay or acknowledge missed events, or log "connected" at INFO so a user can tell when the bot is ready. |
| Oct 31 | Skills installed by symlink trigger "skill file is outside the trusted skills directory" warnings in the gateway log. | Harmless but noisy. Request: a documented way to trust a symlinked skill directory. |


## NemoClaw / OpenShell

| Date | Issue / Feedback | Resolution |
|------|-----------------|------------|
| Oct 16 | Preflight flags 3.7 GiB for the container runtime (recommends 8) and offers a swap file. The sandbox image still built, in about 12 minutes. The warnings were clear and the install flow was smooth. | Continued with defaults. |
| Oct 16 | `nemohermes launch` failed its health check because host port 8642 was held by our existing Hermes gateway, which uses the same default port. The message names the fix command but `status` printed different errors (`docker_unreachable`, then "could not resolve the automatically selected gateway port"). | Worked around by chatting from inside the sandbox. Request: a flag to choose the forwarded port, and consistent status errors. |
| Oct 16 | `openshell sandbox upload <name> <dir> <dest>` creates `<dest>/<dir-name>` instead of putting the contents in `<dest>`, and uploading `dir/.` to an existing destination did not replace the older files. Single-file uploads worked and overwrote. | Uploaded individual changed files. Request: document the nesting rule and add an overwrite or sync mode. |
| Oct 16 | The `pypi` preset allows only listed binaries (`pip3`, `python3`, `curl`), so `uv` and a new venv's Python cannot reach PyPI. This is a good least-privilege default but surprising. | Installed with `python3 -m pip install --target /sandbox/pylibs`. Request: show the allowed binaries in the preset docs. |
| Oct 16 | `inference.local` worked with our custom Token Factory endpoint and kept the API key outside the sandbox. It lists the whole provider catalog on `/v1/models` and accepted both Nano and Super model IDs. | Used for the copilot's own model calls inside the sandbox. This is a strong feature. |
| Oct 16 | The Balanced tier opens `github.com` with full access through `brew-balanced`, which is broader than we wanted. Slack is off by default, which gave us a clean blocked-egress demo. | Noted. Request: a tier between Restricted and Balanced without GitHub write access. |
