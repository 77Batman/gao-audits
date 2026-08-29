# Methods

How the template audit was run. The article states its findings; this states the procedure
that produced them, one level deeper, so the work can be criticised on its method rather than
only on its conclusions.

---

## 1. Selection

**Enumeration, not browsing.** The platform's template library offers no popularity sort and
displays no view counts in its listing. The full library — 11,788 public workflows — was
enumerated through the public API and ranked client-side.

Two facts about that enumeration are themselves findings, reported in the article: the API
accepts a sort parameter and ignores it, and the platform runs two view counters that disagree
by 1.26× to 3.52× for the same template. All popularity figures are therefore approximate, and
the article cites them that way.

**Slate design: one specimen per business function**, rather than the five highest view counts.
The point of the audit was coverage of the failure classes — a booking flow, a customer-facing
chatbot, an outreach sequence, a client report, a knowledge-base assistant — and taking the
top five by raw views would have produced five of whatever category happened to dominate the
library.

Selection criteria, applied in order:

1. **Popularity floor** — drawn from the top of the rankings. All five rank in the top 200 of
   11,788.
2. **Client-facing surface** — the template must touch a lead, customer, or client-visible
   output. Internal utilities were excluded.
3. **At least three of five containing an AI/agent step**, matching where the market is going
   and where silent failure is richest.
4. **Function coverage** — one per slot, no slot doubled.

**Two departures from strict popularity ordering were made and logged**, both where the
highest-ranked candidate in a category could not exercise the failure class under test — a
9-step utility that suggests calendar slots to an operator cannot test customer-facing booking
integrity, whatever its view count. Departures are recorded rather than absorbed, because a
selection rule bent silently is not a selection rule.

## 2. Two-pass structure

**Pass 1 — static grading.** Each template graded against the checklist from its exported
configuration, node by node. 18 statically-gradable checks across six categories, 100 graded
cells across the slate. No grade exists without a node-level evidence reference.

**Pass 2 — dynamic probes.** Four of the five imported into a local instance and executed:
two with live model calls against capped, single-purpose keys created for the run and revoked
after it; two entirely credential-free, with mock steps standing in for external APIs.

**Predictions were written down before the probes ran** and committed to a file, so
reconciliation afterwards was against a fixed prior rather than a memory of one. Where a run
contradicted a prediction, the register records both.

**Every probe input was synthetic.** No real client data, no real recipient addresses or
phone numbers, no live external accounts connected. Where a probe needed known inputs — the
report-arithmetic test — ground truth was computed by script *before* the run, precisely so
that no published figure was a number somebody remembered.

## 3. Deviations

Four substitutions were forced during the probes, each recorded with its reason and its effect
on the finding under test:

- The knowledge-base template's pinned embedding model had been retired by its provider seven
  months earlier and could not be called at all.
- Its pinned chat model had also been retired. The first replacement chosen was **listed by
  the provider's own API as available, and was not callable** — a 404 telling us to use a
  different one. That became a finding in the article.
- No credential existed for the template's document source, so synthetic documents were fed
  through the same ingestion path.
- One trigger type is not executable from the command line and was replaced upstream of the
  logic under test.

The standing rule was that a deviation is acceptable only where it cannot affect the finding,
and where it might, the verdict is scoped to it explicitly. The contradiction-handling verdict
is scoped to the substitute model in the article, with its caveats attached.

## 4. Evidence rules

Three rules governed everything:

**Every claim ties to an artifact** — a configuration path, an execution log, a run
transcript. A finding without one does not exist.

**Findings do not outrank runs.** Where execution contradicted static analysis, the run won
and the published grade changed. This happened more than once, in both directions: one grade
was revised *down* when a probe showed the system behaving better than predicted, and one
finding's blast radius was revised *up* when a probe showed it behaving worse.

**Blocked is not null.** A probe that could not run — expired credential, exhausted quota —
was recorded as blocked, with the static grade left standing, never reported as a
confirmation.

## 5. The error-record practice

The audit keeps a written record of **every mistake made during it, by any layer, caught
before publication.** It has 14 entries across four layers: the agents doing the work, the
layer reviewing them, the layer that captured the exhibits, and the drafting of the article
itself.

**Thirteen of the fourteen ran in the same direction — toward the more publishable claim.**
Not one ran toward a duller finding. That distribution is what an error record looks like when
the people making the errors have a stake in the answer, and it is the entire argument for
gating findings on evidence rather than on review.

The record exists for a specific reason. An audit business whose product is catching
unverified claims has no standing to keep its own unverified claims private. Publishing the
record is also the only honest way to answer the obvious objection to checklists — *"skilled
people are already looking."* Every layer here was skilled and already looking, and each one
failed the same way: a claim asserted without being compared to the record. Each was caught
the same way too — by going back to the artifact. **A review layer is not a
direction-corrector; it is another place the same bias operates. What corrects direction is an
artifact that can contradict you.**

## 6. What this method does not establish

Stated because a method section that lists only strengths is marketing:

- **The slate is five templates.** It supports claims about these systems and about the
  patterns they share. It does not support a population-level claim about the library.
- **The contradiction-handling result rests on two successful runs**, on a two-chunk corpus
  where both versions necessarily surfaced, under a substituted model. It is enough to refute
  the stronger claim we had predicted. It is not enough to characterise how often the failure
  is disclosed in the wild, and the article says so.
- **Configuration review cannot see operations.** Checks marked live-stack-only were graded
  N/A, not guessed.
- **One trigger frequency could not be established** without live credentials and is marked
  unresolved rather than estimated.
- **No probe observed** the pattern we most expected to find — a workflow reporting success
  while a downstream write silently failed. The article publishes that as a null.

---

*The checklist used here is in [`checklist/`](checklist/), released under CC BY 4.0. The
article is [`template-audit-2026-08.md`](template-audit-2026-08.md). The evidence file in
[`evidence/`](evidence/) reproduces the knowledge-base finding offline, with no access to any
account.*
