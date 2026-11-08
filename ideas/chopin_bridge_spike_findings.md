# Chopin bridge spike findings

Slice 1 of `ideas/chopin_bridge_for_cairn.md`, run 2026-11-08/09 on the maintainer's Mac following `ideas/chopin_bridge_spike_plan.md`. Chopin at `386bf21` with `HARNESS=remote`, reached through a Dev Tunnel; the Planner backend is the throwaway `cairn-chopin` (its `SETUP.md` records the setup) running the installed Claude Code 2.1.319 with Cairn 1.9.0 loaded. One GitHub account (`lama-assaf`) played both maintainer and teammate; checks that need a second account are marked as waiting for one.

## The premise holds

The installed Claude Code, driven through the Agent SDK with Chopin's host tools forwarded as an in-process MCP server and Cairn loaded as a local plugin, attaches as a document's Planner and works there: it answered `@chopin` in Chat, listed every Cairn skill (and, unprompted, kept itself to the read-only and discussion ones as Planner), read the document with `read_plan`, edited it with `edit_plan` (repairing its own first failed operation), and opened decision cards with `open_decisions`, including a Mermaid preview that renders. The machine connector, running `claude-agent-acp`, carried out a codebase investigation on a decision card on the Mac and returned a correct finding. Chopin also routes its own background jobs to the attached Planner as `mode: "isolated"` sessions (document summary, title upkeep, decision write-ups), so the bridge sees and can shape all of them.

## A1 — Does an answer reach the Planner? Confirmed

A Save starts a Planner turn whose prompt is fixed, labelled prose:

```
@chopin: A decision you asked people in Decisions has a saved answer. Act on it now.
Decision id (data): "1DKNKDJB58VVYSR0BCV1TXF8AN"
Question (data): "Which format should exports use?"
Answer saved by @lama-assaf (data): "HTML"
Chosen option "HTML" means (data): "Export the plan as a rendered HTML page."
Chopin rewrites the passage beside this decision separately, so leave that passage to it.
```

It names the decision id, the saver, and the chosen option. The bridge never needs to parse it (A2 gives the same facts structurally); it serves as the trigger.

## A2 — Can the Planner read the decision state structurally? Confirmed, two ways

- `read_plan` source: the card is a `<Questionnaire>` element. After a Save it carries `status="decided" by="lama-assaf" at="2026-11-08T22:41:52.000Z"` and an `<Answer value="HTML" choices="<option id>"/>` inside its `<Question>`, and the `read_plan` header lists `1DKNK…: answered by lama-assaf: <question>`. Every question and option has a stable id.
- `read_decision_scope` (offered in the write-up job): JSON with `status: "answered"`, `answers` keyed by question id, `choices` as option ids, `generation`, and `by: "lama-assaf"`.

The bridge maps option ids straight onto Cairn's alternatives: chosen option is the recommended one → `Recommendation-answer:`, another option → `Alternative-answer:`, a written answer → `Manual-answer:`, authored by `by`.

## A3 — Who writes an answer into the page? The Planner, so the bridge

Each Save queues a `[Background job: decision-followup]` isolated session to the attached Planner with `read_decision_scope` and `write_decision_scope`; Claude wrote the passage ("Exports use HTML: …") and Chopin accepted it (`ok: true`, new revision). Under the design's "page is a copy of `requirements.md`" rule the bridge answers this job with Cairn's own fold text, so there is no second writer. The answer turn itself also tells the Planner to leave that passage to the job.

## A4 — Does the questionnaire route say who answered? No

A questionnaire sent by the client (`input-request`) shows as an ordinary card, but its `input-result` is `{status: "answered", value: {answers: [{question, kind: "option", answer: "Two"}]}}`, with no saver. Cairn cards therefore use `open_decisions`, never the questionnaire route.

## A5 — Do Reopen and Discard reach the Planner? No, neither

Reopen showed the card as open again ("Previously chosen by @lama-assaf: HTML") and Discard removed it from the page and moved it to "resolved, discarded by @lama-assaf" in the Decisions tab, but neither sent the Planner anything (nothing within 25 seconds of either), and Reopen left the written-up passage saying HTML. The bridge must notice both by re-reading the document (`read_plan` in a turn, or MCP `read_document` on a timer), as the design already assumed.

## A6 — Can the Planner read chat-suggested cards? Waiting for a second account

With `CONVERSATION_PLAN=on` and one person chatting without `@chopin`, Chopin produced no suggested card: automatic participation handed the conversation to the Planner as a turn ("The human has handed you the conversational turn"), and Claude opened a proper card itself (three options, including the combination "final decisions plus a separate history file", and a passage recording the speaker's lean as a proposal). Chopin's docs say it waits its turn in group discussion, which is where the classifier's suggested cards come from, so this needs two people. For the bridge the solo path is already usable: a chat discussion arrives as a Planner turn the bridge can hand to Cairn's review.

## Operational findings

- **Tunnel headers.** Dev Tunnels rewrite `Origin` and `Host` to `localhost` by default, which fails Chopin's same-origin check ("origin is not allowed"). The fix must be stored on the port (`devtunnel port update … --origin-header unchanged --host-header unchanged`); the same flags on `devtunnel host` alone did not take effect.
- **Stream contract.** Chopin rejects a turn ending in `finish` with no preceding `finish-step` ("Harness adapters must emit `finish-step` before `finish`"), shown as "Chopin's reply failed".
- **Headless permissions.** The backend runs Claude with `permissionMode: "dontAsk"` and an allow-list, since nobody can answer a prompt, and `settingSources: ["project"]`, so none of the maintainer's own plugins or hooks enter Chopin turns.
- **Document renames.** The title job renamed the document; the connector kept working through the old address, and `read_plan` lists `earlier urls`.
- **Connector restarts are fragile.** After a Chopin server restart the connector printed "Missing connection token" and exited, while a re-executed copy of itself stayed alive, unparented, and started about 25 Planner processes at once; one kept the document's Planner seat, so the next connector's Planner was refused ("lama-assaf is connected to this document") until that orphan was stopped and its connection timed out. A real bridge needs a supervisor that stops the whole process group, and the ownership rules (one Planner per document, held through a dropped link) have to be designed around.
- **Port collisions.** Ports 8787 and 8790 were already in use on the Mac (the main Chopin and a file server); mapping them to an anonymous tunnel would have published them. The tunnel exposed the main Chopin dev server for about a minute before this was caught; its sign-in callback and admission list made sign-in impossible through it.

## Not yet checked

- A pull-only account trying to Save (needs a second account).
- Chat-suggested cards in a group discussion (A6, needs a second account).
- The Planner credential's 15-minute renewal over a long attachment, and a Mac sleep with a card open (Review Focus); the connector restarts above interrupted the long run.

## What this changes in the design

- **Reading answers:** use `read_plan`'s `<Questionnaire>` projection (status, `by`, `<Answer choices>`) as the source of truth, triggered by the answer turn, with a timer for Reopen and Discard. Drop the answer-turn text parser as a fallback; it is not needed.
- **Cards:** `open_decisions` only, keyed by Cairn's block id; never the questionnaire route.
- **Writing the page:** the decision-followup job is where the bridge writes Cairn's fold text, which settles the design's question of a second writer.
- **Background jobs:** the bridge receives Chopin's summary and title jobs too and must answer them (cheaply), or Chopin's own features degrade.
- **Process supervision:** the bridge needs its own supervisor for the connector and Planner, robust to Chopin restarts, before slice 2.
