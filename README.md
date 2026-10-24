# opencode-agent-coordination

Shared mailbox, section-level claims, and a live HTML board for parallel AI coding agents.
OpenCode plugin plus CLI. Node 20+.

```
  agent-alpha  --+
  agent-beta   --+-->  .agent/  <--  http://127.0.0.1:4317  (live board)
  agent-gamma  --+     mail.jsonl  claims/  presence/
```

## What it does

When several AI agents (or several OpenCode tabs) edit the same repo at the
same time, they collide. This package gives them a shared coordination layer:

- Agent mail: send, ask, reply, ack, thread. One append-only JSONL log.
- Section claims: lease `path#Function` or `path#L10-L40` before editing.
  Overlaps are refused.
- Decisions: record split / serialize / extend / replace for shared files.
- Live board: a local HTML page (SSE, no polling) showing messages, claims,
  decisions.
- OpenCode plugin: exposes `agent_mail` and `agent_claim` tool namespaces.

State is shared across git worktrees. It resolves to the main checkout's
`.agent` automatically.

## Install

```bash
npm install -g opencode-agent-coordination
```

Or as a dev dependency:

```bash
npm install --save-dev opencode-agent-coordination
```

## Start the HTML board (see agents talk in real time)

```bash
# starts the server and opens your browser
agent-board --open

# or pick a port and host
agent-board --port 4321 --host 127.0.0.1
```

Then open http://127.0.0.1:4317 in your browser.

You will see:

- a live group chat of every agent message (updates instantly via SSE),
- a sidebar with the roster (working, idle, blocked),
- active claims grouped by file with resolved line ranges,
- decisions and acks.

npm scripts (add to your package.json):

```json
{
  "scripts": {
    "board": "node node_modules/opencode-agent-coordination/bin/agent-board.mjs --open",
    "board:serve": "node node_modules/opencode-agent-coordination/bin/agent-board.mjs"
  }
}
```

## Mail commands (simple)

```bash
# who is around
agent-mail roster

# send a message
agent-mail send --from human --to agent-alpha --body "hello there"

# read your inbox (blocks up to 25s by default)
agent-mail inbox --id agent-alpha --ack

# ask a question and wait for the reply
agent-mail ask --id agent-alpha --to agent-beta --body "which field?" --wait 60

# reply to a message
agent-mail reply --id agent-beta --msg <message-id> --body "this one"

# watch the stream (NDJSON)
agent-mail watch --id agent-alpha --json
```

## Claim commands (before you edit)

```bash
# claim a function and a line range
agent-claim claim --id agent-alpha --task my-task --files src/foo.ts#myFn,src/bar.ts#L10-L40

# see who holds what on a file
agent-claim check --files src/foo.ts

# prove your diff stayed inside your claim
agent-claim verify --id agent-alpha --strict

# record how a shared file is handled
agent-claim decide --id agent-alpha --files src/foo.ts --decision serialize --order agent-alpha,agent-beta

# acknowledge a peer's section
agent-claim ack --id agent-alpha --of agent-beta --files src/foo.ts#theirFn

# release when done (notifies the next agent)
agent-claim release --id agent-alpha
```

## Use as an OpenCode plugin

### Option A: register via opencode.json (recommended)

Create or edit `opencode.json` in your project root (or `~/.config/opencode/opencode.json`
for global). Add the package to the `plugin` array:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "opencode-agent-coordination"
  ]
}
```

OpenCode resolves the package from `node_modules` and loads both the `agent_mail`
and `agent_claim` namespaces automatically. Restart OpenCode after editing.

To use a local checkout instead of the published package:

```json
{
  "plugin": [
    "file:./packages/agent-coordination"
  ]
}
```

Manual steps if auto-load does not pick it up:

1. `npm install opencode-agent-coordination`
2. Create `.opencode/plugins/agent-mail/index.ts` and `.opencode/plugins/agent-claim/index.ts`
3. Put this in each file (adjust the path):

```ts
// .opencode/plugins/agent-mail/index.ts
import { register } from 'opencode-agent-coordination/opencode/mail';
export default { id: 'opencode-agent-coordination.mail', setup: register };
```

4. Restart OpenCode. The namespaces appear in the tool catalog.

### Option B: drop-in adapter files

Copy the adapters into your project and point the imports at the package:

```bash
mkdir -p .opencode/plugins/agent-mail .opencode/plugins/agent-claim
cp node_modules/opencode-agent-coordination/lib/opencode-agent-mail/index.ts .opencode/plugins/agent-mail/index.ts
cp node_modules/opencode-agent-coordination/lib/opencode-agent-claim/index.ts .opencode/plugins/agent-claim/index.ts
```

```ts
// .opencode/plugins/agent-mail/index.ts
import { register } from 'opencode-agent-coordination/opencode/mail';
export default { id: 'opencode-agent-coordination.mail', setup: register };
```

Restart OpenCode. The `agent_mail` and `agent_claim` namespaces appear in the
tool catalog.

## Configuration

| Env var | Purpose |
|---|---|
| `AGENT_MAIL_DIR` | Override the state directory (default `<project>/.agent`) |
| `AGENT_CLAIM_ROOT` | Override the checkout root used for git diff (claim verify) |
| `AGENT_ID` | Default callsign for CLI tools |

State resolves to the main checkout's `.agent` in a git worktree, so agents in
separate worktrees share one mailbox.

## How it works

<details>
<summary>Show the data flow</summary>

```
  agent session A  -->  .agent/  <--  agent-board server  <--  browser
  agent session B  -->  .agent/
                            |
                            +-- mail.jsonl   (messages)
                            +-- claims/      (one file per agent)
                            +-- decisions/   (per-file coordination)
                            +-- presence/    (who is doing what)
                            +-- seen/        (per-agent cursors)
```

</details>

- One append-only log (`mail.jsonl`) is the source of truth for messages.
- Per-agent cursors (`seen/*.json`) give materialized unread inboxes.
  Reads are O(new bytes).
- Claim files (`claims/*.json`) hold `path#section` scopes per agent.
- Decisions (`decisions/*.json`) record how shared files are handled.
- The board watches the dir with fs.watch plus a safety poll and streams
  via SSE.

- One append-only log (`mail.jsonl`) is the source of truth for messages.
- Per-agent cursors (`seen/*.json`) give materialized unread inboxes.
  Reads are O(new bytes).
- Claim files (`claims/*.json`) hold `path#section` scopes per agent.
- Decisions (`decisions/*.json`) record how shared files are handled.
- The board watches the dir with fs.watch plus a safety poll and streams
  via SSE.

## Requirements

- Node.js >= 20
- Git (for `agent-claim verify`)

## License

MIT
