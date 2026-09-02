# The Silent Failure Checklist

**Version 0.3 (public edition) · Licensed CC BY 4.0**

A checklist for auditing automated workflows — the kind assembled from AI agents, webhooks,
spreadsheets, CRMs and messaging APIs — for failures that produce **no error anyone sees**.

It exists because the failures that hurt automated businesses are rarely crashes. A crash
gets noticed. What doesn't get noticed is a booking confirmed to a customer that was never
written, a campaign that quietly sends to nobody, a report whose percentages are generated
rather than calculated, or a knowledge base that answers from a policy you replaced last
year.

---

## Grading semantics

Every check is graded per system, and every grade carries evidence.

| Grade | Meaning |
|---|---|
| **PREVENTS** | The structure makes the failure impossible, or guarantees detection |
| **PERMITS** | Nothing stops it; it occurs under ordinary operating conditions without detection |
| **CAUSES** | The design actively produces the failure |
| **N/A** | Not applicable to this system's function — with the reason stated |

**The evidence rule is a hard gate.** Every grade cites a specific node, configuration path,
or run transcript. *A grade without an artifact reference does not exist.* Null grades —
"could not assess" — are recorded as null, never rounded to PERMITS to make a report look
complete.

**BLOCKED is not null.** A check that could not be run — missing credentials, unavailable
dependency, out of budget — is recorded BLOCKED, and any static grade stands unchanged. A
blocked check is not a failed check and must never be reported as confirming anything.

**Findings do not outrank runs.** Where an executed test contradicts a static reading, the
run wins and the grade is revised, in writing, with the run as the evidence.

## Applicability flags

- **`[S]`** — statically gradable by reading the workflow's configuration
- **`[D]`** — requires actually running the system to settle
- **`[L]`** — answerable only against a live production stack (history, access logs, change
  records); graded N/A in a configuration-only review, and said so

---

## The governing principle

### Grade the semantics of a mechanism, never its existence

A check that asks *"is there an X?"* passes any system that has a broken X — and fails the
system that needs the finding most, the one whose operator believes the problem is solved.

Two examples from the audit that produced this version:

- A knowledge base **had** a refresh path. What that path actually did was *append* each
  edited document beside its predecessor, so the more diligently the operator updated their
  documents, the more contradictory the store became. The mechanism existed and manufactured
  the failure the check was written to catch.
- A messaging flow **had** opt-out handling. It matched case-sensitively on a single word,
  so the lowercase spelling — the common one — did not stop anything.

Where a check names a mechanism, it must name the semantics that make it work, and the grade
is assigned against those.

### The predicate corollary

The same error one level down: **where a check names a guard, filter, or condition, grade the
predicate — not the fact that a predicate is present.**

Conditions that routinely fail to mean what they read as:

| Written | Actually admits |
|---|---|
| `x == undefined` | passes `""`, `0`, `false`, `NaN` as valid |
| `!x` | rejects legitimate `0`, `""`, `false` |
| `x?.y` | converts a missing branch into a silent null rather than an error |
| case-sensitive `contains "STOP"` | misses `stop`; matches `STOP BY LATER` |
| `.isEmpty()` on an absent key | throws rather than returning true |

---

## A. Transaction integrity — *does the system's word match the state?*

**A1 · Phantom confirmations `[S][D]`**
Locate the customer-facing success message and the state-changing write. **CAUSES** if the
message is sent before the write's result is consumed; **PERMITS** if they run in parallel or
the write's response is unchecked; **PREVENTS** only if the confirmation is gated on a
verified write. *Not restricted to bookings — any claim a system makes about state falls
here, including a report asserting completeness it never checked.*
Probe: kill the downstream API mid-run and see what the customer is told.

**A2 · Duplicate and impossible transactions `[S][D]`**
Does a re-fired trigger — webhook retry, double submit — produce a second write? Look for
dedupe keys and existence checks, and grade whether the key covers the actual retry path. Is
availability re-checked at commit time, or assumed from an earlier step?
Probe: fire the trigger twice and count the writes.

**A3 · Time semantics `[S]`**
Timezone handling in date and schedule logic; hardcoded offsets; DST-naive arithmetic; the
timezone a customer is quoted versus the one written. Check whether a timezone is configured
at all — an unset one resolves to the host's, which is the deployer's server, not the
client's market.

## B. Sequence liveness — *which paths can die without anyone knowing?*

**B1 · Dead branches `[L]`**
Live check against execution history. Configuration analog: branches unreachable by
construction; disabled steps shipped looking enabled; test fixtures left in place where
re-enabling them would overwrite live data.

**B2 · Fan-out / fan-in reconciliation `[S][D]`**
For every loop or branch over records: is there a terminal accounting — processed + failed +
skipped = entered? **PERMITS** if items can exit with no count; **CAUSES** if the design
drops known categories.

**B3 · Delivery swallowing `[S][D]`**
For every send step: is the response or error consumed? Error-tolerance with no error branch
is swallow-by-design (**CAUSES**). No error handling at all is **PERMITS**, with the
halt-behavior note: an unhandled failure stops the run silently, so everything downstream —
including follow-ups already promised — never happens.

**B4 · Escalation black holes `[S]`**
Where the flow hands off to a human, what guarantees the human saw it? Fire-and-forget
notification with no acknowledgement path = **PERMITS**. *Absence of any handoff is itself
gradable.* And grade the inverse: a flow that **detects** the event a human needs, acts on it
internally, and tells nobody.

**B5 · Dangling references `[S]`**
Does any expression reference a step, variable, credential, or field that does not exist?
Mechanically checkable: extract every reference and diff against the declared set. **CAUSES**
on a customer-facing path — the failure is invisible in the visual editor (the connection
renders correctly; only the reference text is wrong) and fires at runtime, potentially after
upstream side effects have already committed.

**B6 · Silent classification drops `[S]`**
Does any classification, filter, parse or decode step return the same value for *"no, and
that is correct"* as for *"I could not tell"*? Where a failure path and a legitimate negative
are indistinguishable downstream, items are dropped with no error and no count.

**B7 · Batch-atomic failure `[S][D]`**
When a per-item step throws partway through a batch, does it emit the items already
processed, or nothing at all? **CAUSES** where one malformed item suppresses the entire
batch. *A static reviewer cannot infer this, and the intuitive inference — that earlier items
proceed — is often wrong. Know it per platform, or probe it.*

## C. Consent and claims — *what could get you fined, blocked, or sued?*

**C1 · Consent basis `[S]`**
Does the system ingest recipient lists with no consent, source, or last-contact fields? Is
anything filtered on consent before sending? A bulk sequence accepting a bare list is
**PERMITS** at minimum; **CAUSES** where the bare schema is fixed by the system's own design
rather than chosen by the operator.

**C2 · Opt-out integrity `[S][D]`**
Is opt-out handled, and **does the handling work**? Does a follow-up loop re-check opt-out
state before *each* subsequent send, or only at entry? Entry-only = **CAUSES** for the
follow-ups. Test the matcher, not its presence: case, common variants (STOP / UNSUBSCRIBE /
END / QUIT / CANCEL), whitespace, punctuation, and false positives from substring matching.
Check whether the opt-out *confirmation* is gated on the opt-out write actually landing.

**C3 · Asserted-not-computed controls `[S]`**
Rules written as prose instructions to a language model — *"do not quote prices"*, *"always
be truthful"* — with nothing downstream enforcing them. Model output flowing to
customer-facing sends without validation. Any parameter the system accepts and demonstrably
ignores. **Grade the enforcement, not the intention.**

*Branch asymmetry ranks worse than uniform failure:* a setting honoured in one branch and
hardcoded in another grades **CAUSES**, because it defeats testing — the operator changes the
setting, sends a test through the branch that works, and ships. Compare configuration use
across **all** branches.

## D. Reporting truth — *do the numbers recompute from the system's own records?*

**D1 · Narrated vs. computed metrics `[S][D]`**
Trace every number in an output to its origin. Computed by code or aggregation over raw
records is the good path. Produced or *restated* by a language model is narrated by design
(**CAUSES**). Count the model hops between the computation and the reader; each one is a
restatement.

**D2 · Attribution and denominators `[S]`**
How are rates constructed? What is excluded from the denominator, and can the reader see the
exclusion? Silent exclusions = **PERMITS**. Include comparison-window construction — a
period-over-period delta built on mismatched weekdays or a timezone-shifted boundary is a
wrong number with a right formula. Include presentation: a correct figure rendered in a
numeric convention the reader does not share is a wrong number *to that reader*.

**D3 · Self-counting counters `[S]`**
Do counters increment on attempt or on verified outcome? "Sent" counted at dispatch with
delivery unchecked = **CAUSES** for any "delivered" or "reached" claim. Counters that
increment *before* the attempt grade CAUSES outright — they record work that provably did not
happen.

**D4 · Multi-artifact divergence `[S]`**
Does one run produce two or more audience-facing artifacts through different transformations,
without reconciliation — email plus chat message, dashboard plus digest, PDF plus in-app?
**CAUSES** where the branches pass through different models, model versions, or formatting
logic. The harm is specific: two people reading "the weekly report" hold different numbers,
and nothing in the design could ever detect it.

## E. Blast radius and change safety — *what does one edit or one race break?*

**E1 · Shared-state coupling `[S]`**
Fields written from multiple branches or parallel paths; last-write-wins races on the same
record; ordering assumptions between parallel sends. Watch counters read early and written
back late.

**E2 · Change control `[L]`**
Live check. Configuration analog: settings scattered as hardcoded literals across many steps,
inviting in-place edits with no single point of change. Count the repetitions, and note any
configuration embedded in a model prompt — where changing a client's settings means editing
prose.

**E3 · Abort reality `[S]`**
Long waits or sleeps in customer-bound sequences: if the campaign must stop, is there a gate
mid-sequence, or do queued sends fire regardless? Grade whether the gate actually halts
queued work, rather than being a flag nothing reads.

> **The pattern that passes.** In the audit behind this version, E3 was the only check to find
> no instance, and both passing systems shared a structure worth copying: no wait/sleep steps,
> and the send queue re-derived from external state on every tick — so changing the source
> stops the sends.

## F. Context integrity — *is the knowledge the system acts on current, sourced, and scoped?*

**F1 · Staleness `[S][D]`**
Grade **the semantics of the refresh mechanism, not its presence.** In order:

1. **Is there an update path at all?** None = **PERMITS**; **CAUSES** if the framing promises
   currency ("answers from your latest documents").
2. **What does it do on re-ingestion of a changed document — replace, upsert, or append?**
   This is the grading question, and it is usually a single option left at its default.
   *Append* = **CAUSES**: the superseded version is never removed, stays retrievable, and
   every diligent update makes the store worse. Cite the specific parameter and its default.
3. **Is there a deletion path?** Without one, a withdrawn document answers forever. Absence =
   **CAUSES** for any policy, pricing, or compliance corpus.
4. **What is the store's own lifetime?** In-memory and cache-backed stores lose everything on
   restart while the query path keeps serving.
5. **Does the query path depend on ingestion having succeeded?** If not, ingestion failure is
   invisible and the assistant answers from a silently partial corpus.

Probe: change the source document, wait for the refresh, ask again; then delete the source and
ask a third time.

**F2 · Provenance `[S][D]`**
Are answers traceable to retrieved passages — citations, source references in the output — or
free narration over retrieval results? No provenance = **PERMITS** for confidently-wrong
answers nobody can trace. **CAUSES** where the architecture *removes* traceability, e.g. a
retrieval tool with its own summarising model, so the answering agent never sees a passage and
structurally cannot cite one.

**F3 · Contradiction handling `[S][D]`**
When retrieval returns conflicting content — old policy and new policy both indexed — what
happens? Top-k merge with no conflict signal = **PERMITS**; **CAUSES** where F1 step 2 grades
*append*, since the design then manufactures the contradictions it cannot handle.
Probe: index two contradictory statements and ask the question.

**F4 · Scope flatness `[S]`**
One store, all queries hit everything: no per-user or per-client partitioning, no access
filtering at retrieval. For any multi-user framing = **PERMITS** at minimum for cross-context
leakage; **CAUSES** where the corpus routinely contains restricted material (HR, legal,
finance) and ingestion accepts all file types without review.

> **Check the platform layer too.** Partitioning that is correct at the workflow layer can be
> undone one layer down — a store key that the vendor documents as instance-global, outside
> the workflow's own access controls. Read the vendor's scoping documentation, not just the
> diagram.

---

## Scoring

Report **per applicable (check, system) pair** — pairs graded anything other than N/A —
rather than "checks that fired somewhere," which rises with the number of systems reviewed and
becomes near-impossible to fail. Report per-system rates too, so one pathological system
cannot carry a slate.

## Known limits of this version

- `[L]` checks are not gradable from configuration alone and are marked N/A rather than
  guessed.
- `B7` and several `[D]` checks are platform-specific in their answers; they must be known per
  platform or probed, not inferred.
- This version came from auditing public templates. Contact with real deployments will move
  it, and the checklist is versioned so those moves are visible.

### The recorder's ceiling

**Checks that read records inherit the completeness of the recorder. Where recording is in
doubt, audit the accounting identity (`B2`), not the rows.**

Nearly every check in this document has one shape: compare a claim against a record — a
confirmation against the calendar write, a dashboard figure against the raw logs, a reported
count against the items that entered. That shape has a floor it cannot see below. **If a
step declines to act and writes no row, there is no record to compare against, and a
row-reading audit sees a clean database and reports nothing.**

A run status of *completed* is a claim about the run. It is not a claim about coverage, and it
says nothing at all about what the run declined to do. The same distinction the rest of this
checklist draws between a request succeeding and the world changing applies one level lower:
**a green run is a claim about execution, not about what execution skipped.**

The defence that survives this is arithmetic rather than inspection, and it is why `B2` asks
for a terminal accounting — *processed + failed + skipped = entered* — rather than for a list
of processed items. **Make `skipped` a required term in that identity and a category that has
never once been written stops being invisible: it appears as unexplained residue, because the
arithmetic does not balance.** Rows can be absent without trace. A residue cannot hide.

**This puts the obligation on the engine, not on the workflow author, and that is the honest
statement of the limit: observability is a property of the writer, not the reader.** An audit
inherits the ceiling of what the system records. No amount of downstream rigour recovers a
skip the engine never wrote — the identity tells you a gap exists, and only the engine can
tell you what fell into it.

*Raised in the n8n community forum discussion of the audit that produced this version, by a
reader describing a workflow whose skipped branches wrote no step rows at all. Credited with
thanks; named here only with their consent.*

---

*Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt it,
build a product on it — attribution appreciated. If it finds something in your stack, we'd
genuinely like to hear what.*
