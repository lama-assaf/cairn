# Chopin bridge for Cairn

Design draft, 2026-11-08. Precedes any milestone; revised in the open like the other drafts under `ideas/`.

## Intent

Cairn today is invisible to everyone but the person running it: its questions, alternatives, recommendations, and answers live in files a terminal session reads and writes. The bridge puts a Cairn milestone in front of a team as one shared [Chopin](https://githubnext.com/projects/chopin/) page, where teammates open it in a browser, see the goal and the recorded decisions, and answer Cairn's open questions as visual cards, while Claude Code running Cairn on the maintainer's own computer does all of the thinking.

The principle the whole design follows: **Chopin decides, Cairn thinks, git remembers.**

- **Chopin owns the decision.** The card, its options and their visuals, the discussion, the answer, who saved it, reopening, and its history are Chopin's decision records, and they are authoritative. Chopin already does this collaborative part better than Cairn could rebuild it (attributed answers, reopen with retained history, discard, findings, "ask someone" with supports/objects stances).
- **Cairn owns the thinking.** Authoring the questions (the review pass), enumerating distinct and covering alternatives, recommending against the principle store, reconciling dependent picks when an answer undermines them, deriving and completing tasks, finishing the milestone.
- **Git is the memory.** Every answer still lands as Cairn's usual one-commit-per-answer record, authored by the teammate who saved it. When git and Chopin disagree, Chopin wins and the bridge brings git back in line. The mirror keeps principle capture, release notes, and the milestone itself alive if Chopin, a research prototype, breaks or disappears.

**Cairn is the source of truth for how things work on the Claude side** (the user). The bridge is a translator between Chopin and Cairn and carries no copy of Cairn's logic:

- Every Claude session the bridge runs loads the installed Cairn plugin and does its work by invoking Cairn's skills by name (`review-milestone-requirements`, `provide-alternatives-to-all-open-questions`, `recommend-all-open-questions`, the three answer skills, and so on). The bridge never restates a skill's steps in its own prompts, and it never calls Cairn's tools in its own sequence where a skill already defines one.
- The bridge reads Cairn's state only the way Cairn allows: the question document only through the open-question tool's read subcommands (`list`, `locate`, `lift`), and `requirements.md` and the task files as prose. It never parses `open_questions.xml` itself and never writes any milestone file except by running a Cairn skill.
- Where the bridge needs something Cairn cannot yet do, the gap is closed in Cairn first, through Cairn's own milestone flow, as a host-neutral capability that names no Chopin. The bridge never works around Cairn with logic of its own. A Cairn upgrade then changes the bridge's behaviour with no bridge release.
- What the bridge does own is only the translation: which Chopin event triggers which Cairn skill, how a Cairn question is shown as a card, how a card's state is read back, and the safety limits on who may trigger what.

Chopin is a research prototype that warns of breaking changes. The investment is deliberate: the theory (co-written plans, attributed decisions, visual options, agents as participants) is the right one for Cairn, and the bridge is built to absorb Chopin's churn rather than to wait for it to stabilize.

## Goals

- A teammate with repository access opens the milestone's Chopin page and answers every open question without installing anything or knowing Cairn exists.
- Each Cairn open question appears as one Chopin choice decision: its alternatives as the options, the recommended one marked, an option carrying a visual preview where the alternative is easier seen than read.
- A saved answer becomes, with no action from the maintainer beyond having their machine connected, a recorded decision in `requirements.md` and one commit authored by the teammate, under the subject Cairn's provenance rules give it.
- Reopening a card in Chopin reopens the question in Cairn (the answer commit reverted, the question back as a card).
- New questions that an answer raises appear as new cards in the same page without anyone running a skill by hand.
- Later slices: the task list shown as Chopin's implementation graph, and Chopin's **Build plan** executing tasks through Claude Code on the maintainer's machine.

## Non-goals

- Moving Cairn's storage into Chopin's database. `open_questions.xml`, `requirements.md`, and the task files stay where they are and keep their only-writer tools.
- Teaching Cairn's core about Chopin. Nothing under `core/` names Chopin, and Cairn without the bridge behaves exactly as it does today on all three hosts. Cairn may still gain host-neutral capabilities the bridge needs, through its own milestones (see the rule on Cairn as the source of truth).
- Supporting Antigravity or Codex in Chopin mode. The bridge is Claude-only to start.
- Replacing Chopin's own Planner prompts or UI. The bridge works through Chopin's published agent connection and MCP contract only; no fork of Chopin.

## Anti-goals (a failure even if it works)

- **A teammate's words silently lost.** No sync ever overwrites text a person typed into the Chopin page. An edit the bridge cannot place becomes something Cairn surfaces (a review item), never something it discards.
- **Two records that disagree with nobody noticing.** Any divergence between a Chopin decision and git is either repaired toward Chopin or reported; never left standing quietly.
- **Cairn's guarantees bent to fit.** Fold-before-remove, one commit per answer, the strip-on-doubt reconciliation, and provenance subjects hold in Chopin mode exactly as in the terminal.
- **The maintainer's laptop becomes a hidden single point of failure without saying so.** If the machine is away, the page says answers are waiting for it, and nothing is lost when it returns.

## Constraints

- **Everything runs on the maintainer's Mac; teammates come in through a tunnel** (the user). The Chopin server (Bun, PostgreSQL in Docker), the `chopin` connector, the bridge, and Claude Code all run on one Mac. Teammates reach the Chopin page through a Microsoft Dev Tunnel. The tunnel is opened with the maintainer's GitHub login (`devtunnel user login -g`) and is persistent (`devtunnel create`), so its `https://…devtunnels.ms` address stays the same across restarts and can be registered once as the GitHub App's homepage and callback and as Chopin's `APP_ORIGIN`. The tunnel allows anonymous access, because Chopin's own GitHub sign-in and repository-access checks are the gate; it exposes only Chopin's port. No Cloudflare, no paid hosting. The price is that the page is up only while the Mac is awake and connected, and the bridge says so wherever a teammate would otherwise wait silently.
- **Chopin's settings.** `AGENT=on`, `HARNESS=remote`, GitHub App sign-in (not `AUTH_MODE=local`, which a machine-hosted Planner does not support), and a decisions-classifier key, which Chopin requires whenever `AGENT=on` (it refuses to start without one) and uses for its small typed judgments: turn-taking, visual routing, titles, and build gates. Claude cannot fill this slot: Chopin takes only a classifier-type model from Atomic's registry, and the registry has no Claude classifier. The classifier is OpenAI's `openai-decisions/gpt-6-luna` through `OPENAI_API_KEY` for now (the user), with `DECISIONS_MODEL` left unset. Once a `JEV_API_KEY` is set up, Chopin picks `typesafe/jev-latest` automatically, since its config prefers Jev when that key is present. Teammates need pull access to view and push access to answer.
- **Claude is the maintainer's installed Claude Code** (the user). The bridge drives it through the Claude Agent SDK with `pathToClaudeCodeExecutable` pointing at the installed `claude`, so it runs under the maintainer's own Claude login and settings, with Cairn loaded as a local plugin (`plugins: [{ type: "local", path: <installed Cairn> }]`). No Copilot and no separate API key. Its connection to Chopin is outgoing, so Claude needs no tunnel.
- **Chopin ships no first-party Planner backend.** Its agent connection (`wss://<origin>/api/v1/agent`, subprotocol `chopin.agent.v1`) is documented in `docs/agent-connection.md` of `githubnext/chopin`, with `@chopin/agent-link/client` as the reference TypeScript client. The bridge is that backend.
- **The bridge is TypeScript with dependencies, Cairn is stdlib Python.** The bridge therefore lives in its own repository (`cairn-chopin`, my call), the way the benchmark does, and invokes Cairn's tools and skills rather than being part of them.
- **Chopin decision limits.** A questionnaire carries 1 to 16 questions of up to 32 options; an option `preview` is Markdown up to 16,000 characters (image, Mermaid, small table, code sketch). A Cairn question with more than 32 alternatives is out of contract and is refused into the advisory rather than truncated.

## How it works

### The pieces

```
teammates' browsers ──► Dev Tunnel (https://…devtunnels.ms, GitHub-login, persistent)
                              │
maintainer's Mac:       Chopin server (Postgres in Docker, GitHub App, HARNESS=remote)
                              ▲  WebSocket from the connector
                              │
                        chopin connect --planner-command "cairn-chopin" -- claude-agent-acp
                              │
                 cairn-chopin (Planner backend)  ── runs Claude (Agent SDK) with
                              │                     Chopin's host tools + Cairn's skills
                              ▼
                 checkout of the repo: milestones/<milestone>/…  ── git commits, push
```

- **`chopin connect`** on the maintainer's computer pairs the machine with the server. Everything is outgoing; no port opens on the machine.
- **`cairn-chopin`** is the `--planner-command`. Chopin starts it once per milestone document in a checkout of the repository, hands it `CHOPIN_URL`, `CHOPIN_DOCUMENT`, and a short-lived Planner credential (renewed through `CHOPIN_PLANNER_TOKEN_FILE`), and it attaches to the document over the agent connection with `resumable-sessions`, so a sleeping laptop keeps its sessions and its open cards.
- **Each Planner turn** Chopin sends (a member's `@chopin`, a saved answer, an automatic follow-up) arrives with a prompt, instructions, and the list of Chopin host tools for the turn (`read_plan`, `edit_plan`, `open_decisions`, `ask`, `upload_plan_image`, and the rest). `cairn-chopin` runs Claude with those host tools plus Cairn's own skills and tools on the checkout, relays tool calls back over the connection, and streams the reply into Chat.
- **`claude-agent-acp`** (npm `@agentclientprotocol/claude-agent-acp`) is the ACP agent `chopin connect` runs for builds and investigations. It is not yet a Chopin-tested agent; slice 1 verifies it.

### One milestone, one Chopin document

A Cairn milestone maps to one Chopin channel and its document, titled from the milestone's `## Goal`. Which document belongs to which milestone is recorded on the Cairn side (my call: one line in the milestone's `requirements.md` is wrong because that file is prose only; a small bridge state file in the bridge's own state directory, keyed by repository and milestone id, holds it). Child documents Chopin creates for research stay Chopin's.

### What flows which way

| What | Direction | Rule |
|---|---|---|
| Goal, Relevant starting state, Decisions, Out of Scope | git → Chopin document | `requirements.md` is rendered into the document through `edit_plan`; Cairn's text wins for these sections |
| Text a person types into the document | Chopin → Cairn as input | quoted in Chat with its author and handed to Cairn as a review input before the page is re-rendered (see "The page is a copy of `requirements.md`") |
| Open question + alternatives + recommendation | Cairn → Chopin decision card | one choice decision per question, through `open_decisions` (or a questionnaire `input-request`) |
| A saved answer | Chopin → git | recorded as below, one commit, authored by the saver |
| Reopen | Chopin → git | the answer commit is reverted and the question returns to Cairn as an open block |
| Discard | Chopin → Cairn | the question is treated as dropped; the review pass decides whether the gap it covered still needs a question |
| Chat, comments, findings, takes | stay in Chopin | discussion, not decision; Cairn reads recent Chat as context for its turns, as any Planner does |

### The page is a copy of `requirements.md`

The Chopin document always says what `requirements.md` says; teammates see exactly what gets built (the user's choice).

- After every recorded answer, and whenever `requirements.md` changes in git, the bridge re-renders the page's sections from the file through `edit_plan`.
- Chopin's automatic write-up of a saved answer is a Planner job, and under `HARNESS=remote` the bridge is the Planner, so the bridge writes it with Cairn's fold text. No second writer is involved. Slice 1 verifies that the write-up job reaches the remote client; if Chopin instead writes it itself, the bridge's next re-render replaces it, since only the Planner wrote it.
- **Text a teammate types into the page is never lost, even though the page is rewritten.** Before a re-render, the bridge diffs the page against its last render. It posts each human edit in Chat, quoted with its author, and hands it to Cairn as a review input: a change to the goal goes the `modify-milestone-goal` way, which only the maintainer runs, and anything else is folded into the next review pass. Only then does it re-render. The edit leaves the page, but its words stay in Chat, attributed, until Cairn acts on them.

### Questions as cards

For each block in `open_questions.xml` carrying alternatives:

- **Question text** → the card's question. The Short Title → the card's header.
- **Each `<alternative>`** → one option: its label is the alternative's name, its description the what-it-is with its advantage and drawback condensed, its `preview` a visual when the alternative is a shape, a flow, a layout, or a comparison (the bridge asks Claude to draw one with Chopin's Mermaid, table, or wireframe syntax, or to upload an image with `upload_plan_image`; text-only alternatives get no preview).
- **The `<recommendation>`** → its option is listed first, its label suffixed `(Recommended)`, and the recommendation's rationale goes in the card's context. `<applied-principle>` and `<depends-on>` stay in the XML; they are Cairn's bookkeeping and are not shown.
- **A block with no alternatives yet** is not shown as a card until the alternatives pass has run; the bridge runs that pass itself (see "When Cairn runs").
- **Each card is anchored** beside the `### ` subsection of `## Decisions` it will land in when one exists, otherwise beside the Decisions heading.
- **Idempotency** comes from Chopin's stable request keys: the key is the block's id plus a generation counter (`<id>-g<n>`), so re-sending a question never makes a second card, and a discarded card is never recreated by its key.
- **Cards are never edited in place**, because Chopin refuses a key that describes a changed card. A card that has gathered any discussion, draft, or finding is never withdrawn and replaced either, because that loses teammates' words. When a pick is stripped, the card stays as it is and the bridge posts one Chat note saying its recommendation is withdrawn pending re-recommendation. A card is replaced (cancelled, then reopened under the next generation key) only when its alternative set changed and the card is still untouched.

### Reading the decision state

The bridge never relies on parsing the free-text prompt Chopin writes for an answer turn. On every turn, and on a timer between turns, it reads the document with `read_plan`, or over MCP `read_document` with the maintainer's token when no turn is running. It takes each card's state (open, answered with option and saver, reopened, discarded) from the decision projections in the source. It then reconciles that state against git. Chopin's records own the decision and the projection is how a client reads them. Slice 1 verifies that the projection carries the chosen option and the saver's login; if it does not, the answer-turn text is the fallback and the parser is pinned by a contract test.

The timer (my call: every two minutes while the bridge runs) is what notices Reopen and Discard, which start no Planner turn.

### Answers back into git

Chopin hands every saved answer to a Planner turn naming the question, the chosen option, and who saved it. The bridge turns that into Cairn's existing answer path, with the teammate as the author:

| What the teammate did | Cairn path | Commit subject |
|---|---|---|
| Picked the recommended option | `answer-open-question-with-recommendation` | `Recommendation-answer: <Short Title>` |
| Picked another option | `answer-open-question-with-alternative` | `Alternative-answer: <Short Title>` |
| Typed their own answer (written decision or "other") | `answer-open-question` | `Manual-answer: <Short Title>` |

- The commit's author is the teammate. Chopin gives the saver as a GitHub login, and the bridge resolves it to `<id>+<login>@users.noreply.github.com` through the GitHub users API. The committer is the maintainer's machine. Capture then learns who overrode which recommendation, which is better signal than it has today. If the lookup fails, the commit is authored by the maintainer with a `Decided-by: <login>` trailer, so the attribution is never lost.
- All of Cairn's answer rules hold unchanged: fold before remove, the `--option` and `--undermined` reconciliation, strip on doubt, one commit per answer. A hand answer that undermines another standing pick strips that pick; its card is handled as in "Questions as cards".

### Reopen, Discard, and answers given elsewhere

- **Reopen** → `git revert` of that question's answer commit, which brings the block back as Cairn's revert-then-re-answer expects. If the revert does not apply cleanly, or tasks have already been derived from the decision, the bridge changes nothing in git. It posts in Chat that this reopen needs the maintainer and lists it in its status as a standing divergence, reported rather than repaired, as the anti-goal requires. Picks stripped by the original answer's cascade are not restored by the revert; the next recommendation pass re-picks them, as it does today.
- **Discard** → recorded through `answer-open-question` as a `Manual-answer:` whose decision states that the question was dropped in Chopin by the saver and is not decided in this milestone. The review pass then sees the gap as covered and does not re-raise it, and git keeps the record, with no list of suppressed questions held anywhere else.
- **An answer recorded from the terminal** while the milestone is attached to Chopin is allowed. On its next reconciliation the bridge finds the question's block gone with its card still open. It withdraws the card with a Chat note naming the commit, because a Planner cannot save a card on anyone's behalf. Chopin's history then shows the question closed outside it, which is the reported divergence.

### Where the bridge works and what it pushes

- The bridge works in its own worktree at `CHOPIN_WORKTREE`, never in the maintainer's own checkout, so Cairn's dirty-tree guards never meet the maintainer's uncommitted work.
- It commits on a branch named `cairn/<milestone_id>` and pushes after every commit, so teammates see each answer in the repository at once. The maintainer merges that branch with a merge commit (the same rule as outside contributions), which keeps every answer commit for capture and release notes.

### What a teammate's `@chopin` may run

A teammate's `@chopin` runs Claude on the maintainer's machine with the maintainer's push rights, so it is limited.

- **From anyone with push access:** questions about the milestone (`ask-in-milestone-context`), `review-milestone-requirements`, and the two annotating passes.
- **From the maintainer's own login only:** `derive-tasks`, the task-completion skills, `modify-milestone-goal`, `finish-current-milestone`, and any free instruction.

The Agent SDK session for a teammate's turn is given only Cairn's tool commands, git restricted to the bridge's worktree, and read-only file access. Nothing else.

### The document's lifecycle

- `cairn-chopin open` creates the milestone's document over MCP `create_document` with the repository provenance, records the mapping, and prints the `chopin connect --planner` line to run. In slices 1 and 2 the maintainer runs that line by hand.
- After `finish-current-milestone`, the bridge archives the document over MCP. The document stays readable, and nothing is deleted.
- Answers saved close together arrive in one Planner turn (Chopin batches them); the bridge records them one commit each, in Cairn's walk order.
- After recording, the bridge pushes (see "Left to the builder" for the push policy) and re-renders the Decisions section of the document so the page shows the decision as prose.

### When Cairn runs

The bridge runs Cairn's passes on its own at three moments, so nobody needs to type a slash command:

1. **The document is first attached:** render the milestone, run review → alternatives → recommendations, open a card per question.
2. **After answers are recorded:** run review once (an answer can close or raise questions), then alternatives and recommendations for any new blocks, then open their cards. This is the loop Cairn runs today by hand, triggered by Chopin's answer turn instead.
3. **`@chopin <anything>`:** a free instruction runs with Cairn's skills available; `@chopin review`, `@chopin derive tasks`, and similar map onto the matching skill.

Convergence (no open question left) is announced in Chat with Cairn's convergence verdict.

### Tasks and builds (later slices)

- `derive-tasks` still writes `TASKS_TODO.md` through the task-list tool; the bridge then mirrors the list into Chopin's implementation graph with `edit_implementation_graph`, keeping order as dependency edges.
- **Build plan** in Chopin runs on the maintainer's machine through `claude-agent-acp`; each task is carried out by Cairn's `complete-task` path, so `TASKS_DONE.md` and the `Task-completion:` commits stay the record, and the bridge reports task and pull-request progress to Chopin's lifecycle.
- Chopin never blocks a build on open decisions; Cairn's `derive-tasks` precondition (no open question) still holds, so the bridge refuses to draft tasks while a card is open and says why in Chat.

## Slices

1. **Spike (throwaway).** Self-host Chopin with `HARNESS=remote`; connect the maintainer's machine with `claude-agent-acp`; a minimal `cairn-chopin` that attaches, answers a turn with Claude, and opens one hard-coded decision card. Proves the agent connection, the credential renewal, and Claude over ACP before anything real is built.
2. **Questions and answers.** Cards from `open_questions.xml`, answers back as authored commits, reopen as revert, the "after answers are recorded" loop. The milestone document shows `requirements.md` read-only in spirit (human edits detected and surfaced, not yet folded).
3. **The document.** Full two-way handling of the document: human edits turned into review inputs, the Decisions section re-rendered after each answer, visual previews on options.
4. **Chat as evidence ("Jev listens, Cairn decides").** With `CONVERSATION_PLAN=on`, Chopin's classifier turns ordinary Chat into suggested cards: questions, options, reasons, support, and objections, each linked to the message it came from. The bridge never lets one of these become a recorded decision on its own:
   - **A suggested question** is a candidate gap for `review-milestone-requirements`, which authors the real question, merging duplicates; Cairn stays the only author of questions.
   - **A suggested option** outside the card's alternative set re-runs the alternatives pass for that question, through `strip` and a re-run, so the set stays covering.
   - **Reasons, support, and objections** are evidence the recommendation pass weighs beside the principle store and cites.
   - **Inferred agreement** records nothing; only Save is an answer.

   When Cairn takes a suggestion in, the bridge closes the suggested card with a note naming Cairn's card, so teammates never see one question twice. **This slice needs one host-neutral Cairn capability first, built as its own Cairn milestone:** a per-milestone discussion-evidence input written only by a new Cairn tool, which review, alternatives, and recommendation read as cited evidence and never as decided. It names no Chopin and serves evidence pasted from anywhere.
5. **Tasks and builds.** The task graph mirror and Build plan through `complete-task`.

## Decided

- Chopin is the authority for decisions; git is the mirror; Chopin wins on disagreement (the user).
- Teammates open the same Chopin page and answer Cairn's questions there (the user).
- Claude Code on the maintainer's own computer does the work, connected outward to the Chopin server (the user).
- Investing in Chopin despite its prototype status (the user).
- The Chopin page is a copy of `requirements.md` that Cairn keeps in sync; teammates' edits become review inputs, preserved in Chat (the user).
- Cairn is the source of truth for how things work on the Claude side; the bridge only translates and invokes Cairn's skills (the user).
- Everything runs on the maintainer's Mac, including the Chopin server, and teammates reach it through a GitHub-login Dev Tunnel; Claude is the installed Claude Code (the user).
- The classifier is OpenAI for now, switching to Jev automatically once a Jev key is set up (the user).
- Team chat feeds Cairn as evidence through Chopin's conversation cards, and Cairn stays the only author of questions and answers (discussed with the user; slice 4).

## My calls (check these)

- The bridge lives in its own repository, `cairn-chopin`, not under `core/`.
- Claude-only to start; Antigravity and Codex are not supported in Chopin mode.
- The recommended option is listed first and labelled `(Recommended)`; principles and dependencies are not shown on the card.
- The milestone-to-document mapping lives in the bridge's own state, not in `requirements.md`.
- Discard in Chopin is recorded as a `Manual-answer:` saying the question was dropped, so git keeps it and review does not re-raise it.
- The decision state is read from the document's decision projections plus a two-minute timer, not parsed from answer-turn text.
- Reopen is a `git revert` when clean; otherwise it is reported and git is left alone.
- An answer recorded from the terminal withdraws its card with a Chat note.
- Commit author resolved from the saver's login, with a `Decided-by:` trailer as the fallback.
- The bridge works in its own worktree, on `cairn/<milestone_id>`, pushing after every commit.
- Teammates' `@chopin` is limited to questions, review, and the annotating passes; everything else is the maintainer's alone.
- `cairn-chopin open` creates the document over MCP; finishing a milestone archives it.

## Left to the builder

- The model and effort the bridge runs Claude at, and how it structures the Agent SDK session per turn.
- How visual previews are chosen and drawn per alternative.
- Retry and backoff details beyond what Chopin's contract requires.
- How the GitHub user lookup is cached.
- Logging and the bridge's own status output.
