# Chopin Bridge Spike (Slice 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove, on the maintainer's Mac, that a teammate can open a Chopin page through a Dev Tunnel and that the maintainer's installed Claude Code, with Cairn loaded, can act as that page's Planner: reply in Chat, open a decision card, and read back what a teammate answered.

**Architecture:** The Chopin server runs locally at a pinned commit with `HARNESS=remote`, exposed to teammates by a persistent Microsoft Dev Tunnel. A throwaway Bun program, `cairn-chopin`, is started by `chopin connect --planner-command`, attaches over Chopin's agent connection using Chopin's own reference client (vendored), and runs each turn through the Claude Agent SDK pointed at the installed `claude`, with Chopin's host tools forwarded as an in-process MCP server and Cairn loaded as a local plugin. Every message the server sends is logged so the spike's findings come from evidence.

**Tech Stack:** Bun 1.4.4, TypeScript, `@anthropic-ai/claude-agent-sdk` 0.3.319, `@modelcontextprotocol/sdk`, Chopin `packages/agent-link` (vendored), `@agentclientprotocol/claude-agent-acp` 0.101.0, Microsoft `devtunnel` CLI, Docker.

**Spec:** `ideas/chopin_bridge_for_cairn.md` (slice 1, "Spike (throwaway)").

## Global Constraints

- Chopin pinned at commit `386bf21c918e1e3e14c29e74944853e1d4c1d6a8` of `githubnext/chopin`; Bun `1.4.4` (Chopin's `mise.toml` pins it).
- Chopin `.env`: `AGENT=on`, `HARNESS=remote`, `AUTH_MODE` unset (GitHub App sign-in), `APP_ORIGIN=<the tunnel's https URL>`, `OPENAI_API_KEY` set and `DECISIONS_MODEL` unset, so Chopin uses `openai-decisions/gpt-6-luna` (a `JEV_API_KEY`, if one is set up later, takes over automatically), `CONVERSATION_PLAN=on` for Task 5's A6 check only.
- Agent connection: `wss://<APP_ORIGIN>/api/v1/agent`, subprotocol `chopin.agent.v1`, protocol `version: 1`; request features `decision-option-previews`.
- Claude runs as the installed Claude Code (2.1.319) under the maintainer's own login, via `pathToClaudeCodeExecutable`; no `ANTHROPIC_API_KEY` is set for the bridge.
- Cairn is loaded only as a plugin from its installed copy; the spike never edits Cairn or any milestone file.
- Everything spike-side lives in `../cairn-chopin` (a sibling of this repository, its own git repo); nothing is added to this repository except the findings document.
- Secrets (GitHub App secret, session key, classifier key, tunnel tokens) live only in Chopin's untracked `.env` or the OS keychain, never in a commit.

## Review Focus

- **The tunnel rewrites the Host header or origin.** Chopin checks `Origin` against `APP_ORIGIN`; expected: sign-in and the WebSocket both work through the tunnel URL. Task 2's verification signs in through the tunnel, not `127.0.0.1`.
- **The run outlives the 15-minute Planner credential.** Expected: the backend re-reads `CHOPIN_PLANNER_TOKEN_FILE` when the link drops or a reattach is refused with 401, and is never left detached. Task 4 runs the backend for 20 minutes and checks it is still attached.
- **The Mac sleeps mid-turn.** Expected: on wake, the backend reattaches and the open card is still on the page. Task 5 closes the lid for two minutes with a card open.
- **A teammate with only pull access tries to answer.** Expected: Chopin refuses the Save; nothing reaches the backend. Task 5 records what the teammate sees.
- **The OpenAI key is missing or rejected.** Expected: Chopin refuses to start with its "A decisions model is required" message, or logs the classifier failure, rather than hanging a turn. Task 2 Step 3 checks the startup log names the decisions model.

---

### Task 1: Dev Tunnel and GitHub App

**Files:**
- Create: `../cairn-chopin/SETUP.md` (the tunnel id, its URL, the App slug, the test repository; no secrets)

**Interfaces:**
- Produces: a stable tunnel URL `https://<id>-8787.<region>.devtunnels.ms` (call it `TUNNEL_URL`) and a GitHub App installed on one test repository the maintainer can push to and one teammate account can push to.

- [ ] **Step 1: Install and log in.** Run `brew install --cask devtunnel`, then `devtunnel user login -g`. Expected: `devtunnel user show` prints the maintainer's GitHub login.
- [ ] **Step 2: Create the persistent tunnel.** Run `devtunnel create cairn-chopin --allow-anonymous`, then `devtunnel port create cairn-chopin -p 8787 --protocol https`, then `devtunnel show cairn-chopin`. Expected: the output lists port 8787 with a `devtunnels.ms` URL; record it as `TUNNEL_URL` in `SETUP.md`.
- [ ] **Step 3: Register the GitHub App** following Chopin's README "Run locally", with `TUNNEL_URL` in place of `http://127.0.0.1:8787` for the Homepage, Callback (`/auth/github/callback`) and Setup (`/auth/github/setup`) URLs. Use the repository permissions and settings listed there exactly (expiring user tokens on, OAuth during installation off, webhooks off). Install it on the test repository.
- [ ] **Step 4: Verify.** Run `devtunnel host cairn-chopin` and, from a phone on mobile data, open `TUNNEL_URL`. Expected: a tunnel error page saying nothing is listening on 8787 (the tunnel works; the server comes in Task 2). Stop hosting with Ctrl-C.
- [ ] **Step 5: Commit** `SETUP.md` in `../cairn-chopin` (`git init` first): `git add SETUP.md && git commit -m "Record the spike's tunnel and GitHub App"`.

### Task 2: Chopin server on the Mac, reachable through the tunnel

**Files:**
- Create: `../chopin` (clone of `githubnext/chopin` at the pinned commit; its `.env` untracked)
- Modify: `../cairn-chopin/SETUP.md` (startup commands)

**Interfaces:**
- Consumes: `TUNNEL_URL`, the GitHub App's slug, client id and secret.
- Produces: a running Chopin at `TUNNEL_URL` with one channel (document) on the test repository; its canonical URL is `DOC_URL`.

- [ ] **Step 1: Clone and pin.** `git clone https://github.com/githubnext/chopin ../chopin && git -C ../chopin checkout 386bf21c918e1e3e14c29e74944853e1d4c1d6a8`, then `mise install` and `bun install` inside it. Expected: `bun --version` prints `1.4.4`.
- [ ] **Step 2: Configure `.env`** from `.env.example` with the Global Constraints values (`CONVERSATION_PLAN=off` until Task 5), the App's slug, client id and secret, and `SESSION_ENCRYPTION_KEY` from `openssl rand -hex 32`.
- [ ] **Step 3: Start the server.** `bun run db:up && bun run migrate && bun run build && bun run start`, and in a second terminal `devtunnel host cairn-chopin`. Expected: the server log says it listens on 8787 and names `openai-decisions/gpt-6-luna` as the decisions model.
- [ ] **Step 4: Sign in through the tunnel.** Open `TUNNEL_URL`, sign in as the maintainer, create a channel on the test repository, and type `@chopin hello`. Expected: Chat says no Planner is attached and the request is saved (correct for `HARNESS=remote` with nothing attached). Record `DOC_URL` in `SETUP.md`.
- [ ] **Step 5: Teammate check.** A second GitHub account with push access opens `DOC_URL` through the tunnel in another browser profile. Expected: it signs in and sees the channel live (presence shows both accounts).
- [ ] **Step 6: Commit** the updated `SETUP.md`.

### Task 3: Claude Code as the builder machine

**Files:**
- Modify: `../cairn-chopin/SETUP.md` (connect command and result)

**Interfaces:**
- Consumes: `TUNNEL_URL`.
- Produces: the Mac paired as a Chopin machine running `claude-agent-acp`.

- [ ] **Step 1: Install.** `npm install -g @agentclientprotocol/claude-agent-acp@0.101.0` and the `chopin` CLI from the README's install line. Expected: `claude-agent-acp --help` and `chopin --help` both print usage.
- [ ] **Step 2: Connect.** `CHOPIN_URL=$TUNNEL_URL chopin connect -- claude-agent-acp` and approve the pairing in the browser. Expected: `/machines` shows the Mac as Connected.
- [ ] **Step 3: Exercise it.** On a decision card in the channel, choose Research → **Investigate the codebase** with a one-line focus. Expected: the investigation runs on the Mac (its log shows Claude working in a clone of the test repository) and a finding lands on the card. Record pass or the exact failure in `SETUP.md`.
- [ ] **Step 4: Commit** the updated `SETUP.md`.

### Task 4: The minimal `cairn-chopin` Planner backend

**Files:**
- Create: `../cairn-chopin/package.json`, `../cairn-chopin/tsconfig.json`
- Create: `../cairn-chopin/vendor/agent-link/` (copy of `packages/agent-link/src/{client,protocol,index}.ts` at the pinned commit, with Chopin's `LICENSE.md`)
- Create: `../cairn-chopin/src/host-tools.ts`
- Create: `../cairn-chopin/src/turn.ts`
- Create: `../cairn-chopin/src/main.ts`
- Create: `../cairn-chopin/src/log.ts`
- Test: `../cairn-chopin/src/host-tools.test.ts`

**Interfaces:**
- Consumes: `connectAgent`, `TurnContext`, `AgentOptions` from `vendor/agent-link/client.ts`; `query`, `Options` from `@anthropic-ai/claude-agent-sdk`.
- Produces:
  - `hostToolServer(ctx: TurnContext): McpSdkServerConfigWithInstance` in `src/host-tools.ts`: an in-process MCP server named `chopin` that lists exactly the tools in `ctx.message.tools` with their JSON `inputSchema` unchanged, and answers each call with `ctx.callHostTool(name, input)`, mapping `isError: true` to an MCP error result.
  - `runTurn(ctx: TurnContext, env: BridgeEnv): Promise<void>` in `src/turn.ts`.
  - `type BridgeEnv = { claudePath: string; cairnPluginPath: string; checkout: string; logFile: string }`.
  - `logMessage(logFile: string, direction: "in" | "out", message: unknown): void` in `src/log.ts`: appends one JSON line with an ISO timestamp.

- [ ] **Step 1: Write the failing test** `host-tools.test.ts` with a fake `TurnContext` whose `message.tools` is `[{ name: "read_plan", description: "Read", inputSchema: { type: "object", properties: { offset: { type: "integer" } } } }]` and whose `callHostTool` records its arguments and returns `{ output: { ok: true }, isError: false }`:
  - `test("lists exactly the turn's host tools")`: connect an MCP client to `hostToolServer(ctx).instance` over an in-memory transport; `listTools()` returns one tool named `read_plan` whose `inputSchema` deep-equals the one given.
  - `test("forwards a call and its result")`: `callTool({ name: "read_plan", arguments: { offset: 3 } })` makes `callHostTool` receive `("read_plan", { offset: 3 })` and returns content whose text parses to `{ ok: true }`.
  - `test("maps a host error to an MCP error")`: with `callHostTool` returning `{ output: { error: "nope" }, isError: true }`, the result has `isError: true` and its text contains `nope`.
- [ ] **Step 2: Run** `bun test src/host-tools.test.ts`. Expected: FAIL, `hostToolServer` not defined.
- [ ] **Step 3: Implement `hostToolServer`** using the MCP SDK's low-level `Server` with `ListToolsRequestSchema` and `CallToolRequestSchema` handlers, so the host tools' JSON schemas pass through without a Zod conversion; wrap it as `{ type: "sdk", name: "chopin", instance }`.
- [ ] **Step 4: Run** `bun test src/host-tools.test.ts`. Expected: PASS (3 tests).
- [ ] **Step 5: Implement `runTurn`**: call `query` with `prompt: ctx.message.prompt`; `options`: `pathToClaudeCodeExecutable: env.claudePath`, `cwd: env.checkout`, `plugins: [{ type: "local", path: env.cairnPluginPath }]`, `mcpServers: { chopin: hostToolServer(ctx) }`, `systemPrompt: { type: "preset", preset: "claude_code", append: ctx.message.instructions ?? "" }`, `permissionMode: "default"`, `allowedTools: ["mcp__chopin__*", "Read", "Glob", "Grep", "Skill"]`, `disallowedTools: ["Bash", "Edit", "Write"]` (the spike writes nothing locally), an `AbortController` tied to `ctx.signal`. Emit each assistant text block as `text-start` / `text-delta` / `text-end` parts and end with `finish` (`finishReason.unified: "stop"`), shaped exactly as `vendor/agent-link/protocol.ts` `streamPartSchema` requires.
- [ ] **Step 6: Implement `main`**: read `CHOPIN_URL`, `CHOPIN_DOCUMENT`, and the token from `CHOPIN_PLANNER_TOKEN_FILE` (fallback `CHOPIN_PLANNER_TOKEN`); `connectAgent` with `resumable: true` and `handlers.turn = ctx => runTurn(ctx, env)`; `env.claudePath` from `CAIRN_CHOPIN_CLAUDE` (default: output of `whence -p claude`); `env.cairnPluginPath` from `CAIRN_CHOPIN_PLUGIN`; log every server message in and client message out with `logMessage` to `spike-log.jsonl`; on a dropped link, re-read the token file and reconnect with the held `resumption`; exit on SIGTERM.
- [ ] **Step 7: Run it.** `chopin connect --planner $DOC_URL --planner-command "bun run $PWD/src/main.ts" -- claude-agent-acp`. In Chat send `@chopin which Cairn skills can you use?`. Expected: a streamed reply naming Cairn's skills (the plugin loaded); `spike-log.jsonl` shows `attached`, `turn`, and a `host-tool-call` to `read_plan` if Claude read the page.
- [ ] **Step 8: Credential renewal.** Leave it attached 20 minutes, then send `@chopin hello again`. Expected: a reply, and the log shows any reattach using a renewed token.
- [ ] **Step 9: Commit**: `git add package.json tsconfig.json vendor src && git commit -m "Spike: minimal Chopin Planner backend over Claude Code"`.

### Task 5: Decision cards — the three unverified assumptions

**Files:**
- Create: `ideas/chopin_bridge_spike_findings.md` (in this repository)

**Interfaces:**
- Consumes: the running backend from Task 4 and its `spike-log.jsonl`.
- Produces: one findings document answering A1–A5 below with log excerpts, which slice 2's plan is written from.

- [ ] **Step 1: Anchored card.** In Chat: `@chopin open one choice decision with open_decisions asking "Ship format?" with options Markdown and HTML; give Markdown a Mermaid preview`. Expected: the card appears beside a passage with both options and Markdown's diagram.
- [ ] **Step 2: Teammate answers it.** The teammate account picks HTML and saves. Then the maintainer sends `@chopin read the plan and quote the decision record for "Ship format?" verbatim`. Record, from the log:
  - **A1:** whether the answer-handover turn arrives, and its `prompt` verbatim;
  - **A2:** whether `read_plan`'s source shows the chosen option and the saver's login in the decision projection;
  - **A3:** whether a write-up of the answer arrives at this backend (a `start` with `mode: "isolated"` or a turn mentioning the write-up), or whether Chopin writes the prose itself.
- [ ] **Step 3: Questionnaire route.** In `src/main.ts`, add a spike-only probe: when a turn's `prompt` contains `#probe-ask`, the handler skips Claude, calls `client.ask(ctx.message.session, { method: "questionnaire", params: { questions: [{ question: "Probe: which?", header: "Probe", options: [{ label: "One", description: "first" }, { label: "Two", description: "second" }] }] } })`, logs the result, and ends the turn. Send `@chopin #probe-ask`; the teammate answers. Record **A4:** the `input-result` verbatim, and whether it says who answered.
- [ ] **Step 4: Reopen and Discard.** The teammate reopens the Step 2 card, then discards another card. Record **A5:** whether either produces any message to the backend, and what `read_plan` shows for each afterwards.
- [ ] **Step 5: Chat-suggested cards.** Restart Chopin with `CONVERSATION_PLAN=on`. The two accounts discuss an undecided point in Chat without `@chopin` until a suggested card appears. Then send `@chopin read the plan and list every suggested card with its source messages`. Record **A6:** whether the backend can read the suggested card, its options, and the links to the quoted messages, through `read_plan` or another host tool, and whether it can close the card with a note.
- [ ] **Step 6: Review Focus checks.** Close the Mac's lid for two minutes with an open card, wake it, and check that the card and the attachment survive. Have a pull-only account try to Save. Record both outcomes.
- [ ] **Step 7: Write the findings** into `ideas/chopin_bridge_spike_findings.md`: one section per assumption (A1–A6), each stating confirmed, refuted, or partial, with the log lines that show it, followed by what each result changes in `ideas/chopin_bridge_for_cairn.md`.
- [ ] **Step 8: Commit** in this repository: `git add ideas/chopin_bridge_spike_findings.md && git commit -m "Record the Chopin bridge spike findings"`.
