# Delivery across CLI sessions

The durable mailbox is `<run>/events.jsonl`. All participants need the same local
filesystem and repository access (Linux/macOS, Git, Node 18+, POSIX shell for checks).
This is not a network transport or a launcher. Native subagent tools may create
sessions, but do not replace the mailbox's revision verdicts.

## Choose by capability

| Session | Delivery | Setup |
|---|---|---|
| Claude Code with background task completion notifications | Background `wait` | No adapter or plugin configuration |
| Codex with `queue --resume` and reachable session service | `codex-queue` + watcher | Exact thread ID; verify resume works |
| OpenCode V2 with `api session.prompt` | `opencode` + watcher | Exact session ID and correct running server |
| Older CLI, missing service, unknown runtime | Pull | Bounded active wait or manual resume |

Never select an adapter from a CLI name alone. An inherited `CODEX_THREAD_ID` does
not mean the session is Codex. Bind explicitly. None of these modes changes approval,
sandbox, authentication, model or tool permissions.

## Claude Code

Run `node "<run>/team.mjs" wait --as <id> --timeout 3600 --soft-timeout` with the
shell tool's `run_in_background: true`. End the turn only once the background task
is registered. A completion notification starts the next turn. Process and acknowledge
the batch, then register the next background wait. On `state: idle`, re-arm; on
`run closed` or exit 4, stop. Do not use `nohup` alone: it cannot notify the model.
If background completion notifications are unavailable, use pull mode.

Do not write to Claude's internal team inboxes or inject terminal keystrokes. Native
Claude Teams are useful within Claude; they are not the shared cross-CLI protocol.

## Codex

First inspect `codex queue --help` in the watcher environment. It must support
`--thread`, `--message` and `--resume`. Use the actual current thread UUID from the
session environment or the host's session information; never select the latest thread.

```sh
node "<run>/team.mjs" bind --as <id> --wake codex-queue --thread <thread-id>
```

For Codex only, explicit `--wake codex-queue` may use `CODEX_THREAD_ID` or
`CODEX_SESSION_ID` when `--thread` is omitted. Remote services can be selected with
`--server <ws-or-unix-url>` (mapped to Codex `--remote`); do not put credentials in URLs.
Use the CLI's authentication configuration in the watcher environment.

The watcher calls `codex queue --resume`. A zero exit status can still include a
resume warning. A running reachable service and a round-trip probe are required.
Do not start a competing `codex exec resume` process on an interactive thread.

## OpenCode V2

Inspect `opencode --version` and `opencode api --help` in the watcher environment.
This adapter targets V2, not V1's `prompt_async` API. Obtain the exact current session
ID from OpenCode's session UI or session listing; do not infer it from the repository.

```sh
node "<run>/team.mjs" bind --as <id> --wake opencode --thread <session-id>
```

For an explicitly selected server, append `--server http://127.0.0.1:4096` using its
actual URL. Without it the OpenCode CLI resolves its normal background service.
The adapter invokes `opencode api session.prompt --param sessionID=... --data ...`
with `{ "text": "<fixed reminder>", "delivery": "queue", "resume": true }`.
Queue delivery avoids interrupting active work. The API surface is experimental;
validate it after OpenCode upgrades. Credentials stay with the CLI.

## Watcher and verification

Keep one watcher per run in a persistent terminal, or a host background task whose
lifetime survives model turns:

```sh
node "<run>/team.mjs" watch
```

Use the usual approval mechanism if that environment needs access outside a sandbox.
A background command tool returning a process ID does not necessarily mean the process
survives the end of the turn. If it cannot, use a user-owned persistent terminal.

Before relying on wake, the driver posts a NOTE to each seat asking for a single probe
reply; each recipient reads, replies with `--seen`, then waits. Include the driver's
seat in the return path. Confirm `status` shows acknowledged messages in both directions.
Run the probe while the receiving session is idle. Passing while active does not prove
idle wake. A probe checks delivery, not approval of the plan.

Reminders carry paths and seat IDs, never peer-authored message bodies. Only mailbox
acknowledgements prove receipt. The watcher batches bursts, preserves pending reminders
on restart and retries with a cap. After `WAKE_FAILED`, check the service, permissions,
session binding and `status`; manually read/acknowledge or rebind to recover. A retry
may produce a duplicate reminder, so consumers must tolerate repeated event IDs.

## Pull and recovery

```sh
node "<run>/team.mjs" bind --as <id> --wake none
node "<run>/team.mjs" wait --as <id> --timeout 0
```

Read at useful checkpoints and before publishing a verdict. During an active turn a
bounded foreground wait is possible, within the host's timeout limit. If there is no
way to notify an idle session, say so and ask the user to resume it when ready. Never
promise autonomous operation or spend indefinite model turns polling. Existing messages
remain available until explicitly acknowledged.

After compaction, read the role, protocol, `status` and pending messages. After moving
to a new conversation, reviewers use `join --rejoin <id>` then `bind`; the driver
uses `bind --as d`. The watcher resets retry state when the binding changes.
