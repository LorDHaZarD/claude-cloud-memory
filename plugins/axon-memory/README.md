# axon-memory plugin

Cloud Claude sessions (Claude Code on the web, the desktop app's Cloud sessions and project threads) recall and retain through a self-hosted [Hindsight](https://github.com/vectorize-io/hindsight) memory node, the way devices running the same hca hooks do. Their memories go to the same bank, with the same document and tag layout, stamped by the gateway as a lane of their own.

A cloud session runs in a VM with no tailnet, no `~/.hindsight` config and no key file, and project threads don't load a repository's hooks. This plugin carries one script, [`bin/cloud-memory-hook`](bin/cloud-memory-hook), that runs the pinned hindsight-coding-agents (hca) hooks inside the session. It does nothing unless `CLAUDE_CODE_REMOTE=true` and the environment sets `AXON_DOMAIN=personal`, an `https://` `AXON_MEMORY_URL` and a well-formed `AXON_MEMORY_BANK`, so on a device it never doubles hca's own hooks, and a work session never reaches the personal node.

| Event | What runs |
|---|---|
| `SessionStart` | Installs the hca pin into `~/.cache/axon-memory` once per VM (npm, scripts off), then hca's session-start hook. After `compact` or `clear`, it clears hca's per-session cache so the next prompt injects again |
| `UserPromptSubmit` | hca's recall, once per session, 2 s budget |
| `Stop` | hca's retain of the new turns into this session's own document |
| `PreCompact` | The same retain, before the transcript is summarised |

Every failure exits 0 without output: a slow or missing node never blocks a session.

## How the key stays out of the VM

The cloud environment holds the gateway key as a **network secret** for the door's host. Anthropic's agent proxy adds the `Authorization` header after a request leaves the VM, so the key is never in an environment variable or file the session can read. The hca config this plugin writes has no `apiToken` (hca then sends no header of its own), and hca's `fetch` goes through the proxy because the hook sets `NODE_USE_ENV_PROXY=1`. Any `HINDSIGHT_*` variable in the session is dropped before hca runs, so nothing in the session can point hca at another server or key.

The gateway knows the key as `claude-cloud`. It stamps `trust:agent`, `caller:claude-cloud` and `via:claude-web` on everything the session retains, and its `recall_tags` make every recall find only `trust:reviewed` memories unless the recall asks for `trust:agent`. The hook adds `env:claude-project` in a project thread and `env:claude-web` elsewhere; those are self-reported, `via:` is the trusted label.

## One injector, one writer per session

- **Claude's project memory stays separate.** A project's `MEMORY.md` and its files are coordination notes for its threads; Hindsight is the shared dev memory. hca strips `<system-reminder>` blocks (where `MEMORY.md` is injected) and its own injected memory before it retains, so threads don't store the project index or echo old recalls.
- **Parallel threads never share a document.** hca retains each session into a document keyed by its session id and appends to it, and the gateway refuses edits to documents another caller wrote.
- **Cloud memories stay unreviewed.** They carry `trust:agent`, so other sessions don't recall them unless they ask for it; promoting one to `trust:reviewed` is a separate, reviewed step.

## Set it up

You need a Hindsight memory node behind a gateway that knows this key as an
agent caller (it stamps `trust:agent`, `caller:claude-cloud` and
`via:claude-web` on retains and limits recalls to `trust:reviewed`). Give
`caller:claude-cloud` memories a retention rule, since every thread retains
its turns on each stop and compaction.

Publish the gateway on a public HTTPS door that forwards only these routes:

| Method | Path |
|---|---|
| `GET` | `/v1/version` |
| `GET` | `/v1/default/banks/<bank>/config` |
| `POST` | `/v1/default/banks/<bank>/memories` (retain) |
| `POST` | `/v1/default/banks/<bank>/memories/recall` |
| `GET` | `/v1/default/banks/<bank>/operations/<id>` |

hca also tries knowledge pages and a one-row document listing; the door
refusing them only costs a diag line.

In Claude's settings (claude.ai/code or the desktop app):

1. Create a cloud environment and select it in the project's settings (or as
   the default for web sessions).
2. Add environment variables `AXON_DOMAIN=personal`,
   `AXON_MEMORY_URL=https://<door host>` and `AXON_MEMORY_BANK=<bank>`.
   Without the domain the hooks do nothing.
3. Add a network secret: type Bearer, allowed website the door's host, header
   `Authorization` with prefix `Bearer`, value the gateway key.
4. Add this marketplace (`LorDHaZarD/claude-cloud-memory`) and enable the
   `axon-memory` plugin under the project's Plugins settings.

Check it from a new thread: a recall of a seeded reviewed memory comes back,
and a retain shows `via:claude-web` in your memory viewer.

## Pin

`HCA_VERSION` in the script pins hindsight-coding-agents. Bump it on purpose,
never to latest.
