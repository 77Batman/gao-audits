# The safety layer is a polite request to a language model

*We audited five popular automation templates on n8n — one per core business function: the workflows thousands of businesses install as their lead bots, their client chatbots, their reporting pipelines. Every claim below traces to a committed artifact — a workflow's JSON, an execution log, or a run transcript — and our own errors are part of the record. Including the ones we made writing this.*

---

## Before we audited a single template, the metrics failed the audit

We started with a boring task: rank n8n's public template library by popularity and pick a slate of the most-installed client-facing workflows. The library holds 11,788 community workflows. That ranking task produced our first two findings — neither of them about templates.

The library UI offers no popularity sort and shows no view counts, so we enumerated all 11,788 templates through the public API. The API accepts a `sortBy` parameter — and ignores it. `views`, `popularity`, and no value at all return byte-identical orderings. A sort control that exists, accepts input, and does nothing.

Then we noticed the platform runs two view counters that disagree. The search endpoint and the per-workflow endpoint return different `totalViews` for the same template — differing by 1.26× to 3.52×, with no documentation for either. Every popularity number in this piece is therefore approximate by construction, and we cite it that way.

Probe preparation later added two more. An n8n embedding node's on-screen notice states the default model produces 768-dimensional vectors; the node's current default model emits 3,072, and neither n8n nor its underlying library ever transmits the parameter that could change that. A builder following the interface's own guidance would create an index whose dimension can't match what the node sends — a mismatch we infer from the source rather than tested, and mark as such. And the model dropdown is populated from a provider listing that advertises models the caller cannot invoke: one listed model, shown with full capabilities, returns a 404 instructing you to use a different one. The builder discovers this at runtime, not at configuration time.

Four instances, one shape: **an indicator that doesn't match the thing it indicates.** Keep that shape in mind. It's the whole story.

## What we did

We selected five templates — slot-based, one per function, drawn from the top of the library's rankings (all five rank in the top 200 of 11,788), with two deliberate, logged departures where strict popularity would have missed the function under test: a lead-to-booking flow with an AI scheduling agent (~23,000 views), a customer-facing WhatsApp chatbot (~332,000 views by one counter; ~529,000 by the other), a bulk email campaign with follow-up sequencing (~21,000 views), an AI-generated analytics report (~36,000 views), and a RAG knowledge-base chatbot (~95,000 / ~335,000 views). Together they're the default architecture for a lead bot, a support bot, an outreach engine, a client report, and a company-knowledge assistant — the stack a typical agency deploys for its clients.

We graded each against a written checklist — 18 statically checkable failure classes across six categories: booking integrity, sequence liveness, consent and claims, reporting truth, blast radius, and context integrity. Grades were *prevents*, *permits*, or *causes*, and no grade exists without a node-level evidence reference. We then imported four of the five into a local instance and ran them — two with live model calls against capped throwaway keys, two entirely credential-free with mock side effects standing in for real APIs.

Two rules governed the whole exercise. Every claim ties to an artifact — a JSON path, an execution log, a run transcript. And **findings don't outrank runs**: when execution contradicted our static analysis, the run won and the published grade changed. It happened more than once. We'll show you where.

We pre-registered the outcomes, including the null: if the templates came back clean, we'd publish that. They didn't come back clean. Of 18 checks, 17 found at least one structural instance; measured properly, against applicable check-template pairs, 48 of 52 — 92%. (Our original pre-registered metric was itself later filed as imprecise in our register; that correction is part of the record too.) Across 100 graded cells: 26 *causes*, 22 *permits*, 4 *prevents*, 48 not applicable.

One result was universal. All five templates express their safety-relevant rules — don't promise what you can't verify, report accurately, behave correctly — as prompt instructions to a language model, with nothing downstream that enforces, checks, or measures compliance. **In five of the platform's most popular automation templates, the safety layer is a polite request to a language model.**

*(A note on "we": this audit was run by one human and a small team of AI agents operating under a written delegation protocol — evidence gates, logged deviations, an error record. The error record below spans every layer of the operation, this article's drafting included.)*

## Finding 1: The knowledge base that never forgets — including what you replaced

The RAG template wires Google Drive to a vector store: drop documents in a folder, and a chatbot answers questions from them. It even has a refresh path — edit a document and a trigger re-ingests it. That refresh path is what makes it dangerous.

The ingestion node runs in `insert` mode with the store's clear-before-insert option not enabled. So when you edit a document, the new version is *appended* alongside the old one. Nothing is replaced. Nothing is ever deleted.

We ran it. One synthetic HR policy, ingested: one vector. Edit the policy — parental leave changes from 12 weeks to 20 — and re-ingest: two vectors. Query "how many weeks of parental leave do employees get?" and the store returns both versions, 0.003 apart in relevance:

> `0.8485 — "…entitled to 20 weeks of paid parental leave."`
> `0.8455 — "…entitled to 12 weeks of paid parental leave."`

**Exhibit [E1](exhibits/E1.png)**

The current version happened to rank first — by 0.003, a margin that is similarity noise, not policy. Nothing structural prefers current over superseded: no deduplication, no recency weighting, no version metadata, no conflict signal — and both versions were served to the model regardless. Every edit your company ever makes to its documents becomes a permanent contradiction in the store.

What happened next is worth reporting carefully, because it went against our prediction. We predicted the chatbot would silently blend the versions or silently pick one. It didn't — in both successful runs, the model disclosed the conflict unprompted. Run 2, verbatim: *"one policy statement mentions 12 weeks of paid parental leave, while another mentions 20 weeks… Please consult with HR directly to clarify which duration applies."* So we revised that grade down, from *causes* to *permits*, and we're attaching the caveats plainly: two runs, a two-chunk corpus where both versions necessarily surfaced, and a substituted model — because the model the template ships with was retired by its vendor, which means what the template did *as published* is permanently unanswerable.

Sit with the corrected claim, because it's worse than the dramatic one: **the design manufactures contradictions and contains nothing that detects them; whether the user is told depends on the model's disposition.** Today's model happened to be careful. That care is a vendor's property, not your system's — and vendors retire models on month scales. Which brings us to:

**The template cannot embed anything as shipped.** Its pinned embedding model was shut down by its provider on January 14, 2026 — seven months before this audit. That fact is verified: the retirement is public, the pin is in the template's JSON. What follows from it, we state as the inference it is — we cannot see anyone's deployment: a business that deployed this template near-verbatim would have a broken ingestion path from that date, a knowledge base frozen at its last successful sync, and a chatbot still answering from it, with no error surfacing anywhere a non-technical operator would look.

## Finding 2: The bot that reports your booking to no one

The booking template's AI agent qualifies a lead over SMS, checks availability, and books the appointment through a calendar tool. Then it hands off to a node that saves the session — and that node's expressions reference a node called "Appointment Scheduling Agent," while the node in the workflow is named "Appointment Scheduling Agent1." One character of drift. The reference points at nothing.

The result, reproduced in execution: the booking tool fires and returns ACCEPTED — in our probe, a mock standing in for the calendar API, returning a synthetic id; in a production deployment, this is the step that creates the real event. The agent composes its reply: *"You're booked for Tuesday at 10am. A confirmation email is on its way."* Then the session node throws `Referenced node doesn't exist`, and the send-reply node never executes at all.

**Exhibits [E2](exhibits/E2.png), [E2b](exhibits/E2b.png)**

So the booking step completes, the customer is told nothing, and the session is never recorded. And because the follow-up logic reads its state from records that were never written, the wiring implies one more consequence — this one an inference from the workflow's structure, since our probe was a five-node mock without the follow-up branch: the sequence would treat this booked customer as an unbooked lead and keep chasing them. The failure is visible in the execution history, if anyone reads it; on the canvas, where builders actually look, the workflow appears intact.

We confirmed the execution behavior twice, on two different installations, facts identical. It is not an exotic edge — it's what a rename without a reference-check does to any workflow whose confirmation precedes its verification.

## Finding 3: The client report where the percentages are the model's opinion

The analytics report template is the most instructive of the five, because it proves it knows better. Its pipeline computes the level metrics — page views, sessions, revenue — with deterministic aggregation nodes, and computes derived metrics like CTR, CPM, and ROAS in eight code nodes of clean date arithmetic and math. Then the template hands those computed numbers to a language model with a table whose Percentage Change column contains, literally, the placeholder string "Percentage Change" — for the model to fill in by doing the arithmetic itself. The one column that carries the report's meaning — *are we up or down?* — is generated, not calculated. From there the design routes the output through three model hops (our probe exercised the first) before the result goes to the client.

We ran that first hop three times on byte-identical input, against ground truth we computed by script. Six of eight percentage cells were wrong in at least one run. Two — Total Page Views and Total Purchases — were wrong in all three. Six of eight changed across runs whose inputs did not change. Two of eight were correct and stable.

**Exhibits [E3](exhibits/E3.png), [E3b](exhibits/E3b.png)**

Now, proportion: the errors were small — 0.01 to 0.03 percentage points. 12.64% instead of 12.67% turns no business decision, and anyone quoting this as "the AI invents your metrics" is overstating three runs. The finding is not the magnitude. **The finding is that a pipeline which computes its inputs perfectly then passes them through an unverified generative step — and a 0.03-point drift and a transposed digit pass through that step with exactly the same silence.** Nothing in the workflow would notice either.

And the materially misleading failure wasn't arithmetic at all. In one of three runs, the report dropped the *(min)* unit label from Average Session Duration — emitting "3,12" bare, for a metric conventionally read in seconds. A site with 3.12-minute sessions and one with 3.12-second sessions are different businesses. No static analysis catches a label that usually appears; only running the thing three times did.

One more, from reading the code the template *did* write: it builds its "previous year" comparison window as same-date-minus-seven-days — which compares different weekdays across years — and constructs two of its three source windows in UTC against a workflow configured for Europe/Berlin. The report disagrees with itself about what "last week" means, per data source, before the model ever touches it.

## Finding 4: The campaign that emails no one and errors where no one looks

The email campaign template reads leads from a spreadsheet and sends staged sequences with follow-ups for non-repliers. Its code nodes guard against a missing company column — with a predicate that checks for *undefined*. An empty cell passes that guard. A missing key throws. We assumed, from static reading, that a bad row would split the batch: earlier rows sent, later rows stuck. The run corrected us, in both directions at once.

The failure is batch-atomic. One row missing the key and the per-item node emits nothing — the send node never executes for *anyone*. Zero emails, from the first message on, on every hourly trigger, forever. How often a real spreadsheet integration produces a missing key rather than an empty cell is something we couldn't establish without live credentials, and we mark it unresolved. What the runs establish is the mechanism: the workflow errors every hour into an execution log nobody reads, while the sheet accurately shows zero rows emailed — a record that is truthful and unconsulted. Our originally drafted claim ("earlier rows send; it looks like a campaign in progress") was contradicted by a two-minute run, and this paragraph is the corrected version.

The same template contains our favorite small finding of the audit — a static one, read from the workflow's wiring rather than triggered live: when a prospect replies — the one event the entire campaign exists to produce — the workflow detects it, correctly uses it to stop the sequence, and then discards it. No notification, no task, no record anyone would see. The system succeeds and throws away the success.

That shape — *functioning systems with no concept of reporting their function* — is Finding 2 and Finding 4 wearing different clothes, and once you've seen it you'll find it in your own stack.

## The null result

We went in expecting to capture the most damning pattern in automation: a workflow that reports success while a downstream write silently failed. **We did not observe it.** Not in any probe. The booking failure reported an error (on a screen nobody watches, but an error). A platform bug that allegedly routes errors to the success branch did not reproduce on the current version. We're reporting that plainly, because an audit that only finds what it came for isn't an audit.

## Our errors are part of the evidence

Our register keeps an error record: every mistake made during this audit, by any layer, caught before publication. It has 14 entries. Thirteen of the fourteen ran in the same direction — toward the more publishable claim. A grade nearly asserted from a config value without a run. A finding almost filed on an unverified platform bug that would have made Finding 2 far more dramatic. An exhibit nearly published as showing a defect it doesn't contain. Our own tooling reported a failed install as a success — the pipeline returned the wrong process's exit code — a defect of exactly the class we were auditing for, in our own instruments.

Entry 14 is this article. The first draft asserted a real calendar event in Finding 2 — the run was a mock, clearly labeled in our own logs. It reversed the direction of the document edit in Finding 1. It called an erroring workflow "green-looking." Every one of those errors ran toward drama, and every one was caught the same way as the other thirteen: line-verification of the draft against the register. The piece you're reading is the corrected version, and the correction process is in the record.

The entries span four layers: the agents doing the work, the layer reviewing them, the capture layer that filed the exhibits, and the drafting of this piece. Each layer failed for the same underlying reason, and each catch was identical: *compare the artifact to the record.* Never by care, never by being smarter. Which answers the obvious objection to checklists — "skilled people are already looking." Every layer here was skilled and already looking. The checklist isn't a substitute for skill; it's the record that makes the comparison possible at all. And a review layer, we can now report from direct experience, is not a direction-corrector — it is another place the same bias operates. What corrects direction is an artifact that can contradict you.

## What this means if you run these things

These templates aren't badly built by their authors' lights — they're built to demo, and they demo beautifully. The gap is that a demo's success is narrated ("You're booked!") and production success is verified (the calendar entry exists, the send log confirms, the number recomputes). Every finding above is that gap in a different costume: confirmations that precede their writes, metrics that are generated rather than calculated, knowledge stores that accumulate rather than update, campaigns whose one meaningful event is discarded, and a platform whose own indicators — sort parameters, view counters, dimension notices, model listings — don't match the things they indicate.

If your business runs on workflows like these — and if an agency built your automation, it very likely does — the questions to ask are mechanical, not philosophical. Does anything reconcile what your bots *say* against what your systems *record*? Would a dead sequence announce itself, or just go quiet? Do the numbers on your client dashboard recompute from raw logs? When did anyone last check that the models your stack pins still exist?

We audit exactly this — the same checklist, the same evidence rules, the same error record turned on your stack instead of the ecosystem's defaults. If you'd rather run the comparison yourself, everything above tells you how. Either way: close the books on your automation once. The templates never will.

---

*Method notes: five templates selected slot-based (one per business function) from a full enumeration of the public library's rankings, with two logged departures from strict popularity ordering; 18-check static grading with node-level evidence; dynamic probes on a local instance — two specimens with live model calls under capped throwaway keys, two credential-free with mock side effects; pre-registered thresholds (met: 17 of 18 checks fired; 48 of 52 applicable check-template pairs, 92%); contact with the artifacts produced seven new failure modes, which became four new checks, revisions to three existing ones, and two new governing rules; all grades revised where runs contradicted static analysis, and one revised downward. Templates are identified by function and approximate view count rather than by author — the findings are about the ecosystem's defaults, not about individual builders. Platform findings were observed on n8n 2.35.7 in August 2026.*
