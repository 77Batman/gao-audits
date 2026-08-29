# gao-audits

**Public evidence from our audits of automated business systems.**

---

## What this repo is

In August 2026 we audited five widely-installed public automation templates — the workflows
businesses deploy as their lead bots, client chatbots, outreach engines, reporting pipelines
and knowledge assistants — against a written checklist of **silent failure modes**: failures
that produce no error anyone sees.

All five graded *causes* on the same check. Every one expresses its safety-relevant rules —
don't promise what you can't verify, report accurately, behave correctly — as prompt
instructions to a language model, with nothing downstream enforcing, checking, or measuring
compliance. A booking confirmed to a customer that was never recorded. A campaign that emails
nobody and errors where nobody looks. A client report whose percentages are generated rather
than calculated. A knowledge base that appends every edit and never forgets what you replaced.
This repository contains the article, the checklist it used, the exhibits, and an evidence
file you can verify yourself.

**Our own errors are part of the record.** The audit kept a written log of every mistake made
during it, by any layer, caught before publication — 14 entries across four layers, including
the drafting of the article. Thirteen of the fourteen ran toward the more publishable claim.
That log is summarised in the article and the method notes, because an audit business whose
product is catching unverified claims has no standing to keep its own private.

## Contents

| Path | What it is |
|---|---|
| [`template-audit-2026-08.md`](template-audit-2026-08.md) | **The article.** The findings, the corrections, the null result, the error record |
| [`checklist/`](checklist/) | **The checklist.** Category and check definitions, grading semantics, applicability flags. **CC BY 4.0 — use it** |
| [`METHODS.md`](METHODS.md) | Selection procedure, two-pass structure, deviations, evidence rules, and what the method does *not* establish |
| [`exhibits/`](exhibits/) | Five screenshots from the executed probes, referenced from the article |
| [`evidence/`](evidence/) | A self-verifying evidence file for the knowledge-base finding |

## Verify the headline finding yourself

`evidence/f1-vector-evidence.json` contains the query embedding and both stored vectors in
full. The scratch index and every API key used in the audit were revoked before publication —
**and the evidence still checks out, because it needs no live service:**

```python
import json, math
ev = json.load(open("evidence/f1-vector-evidence.json"))
q  = ev["query_embedding"]["values"]
dot = lambda a, b: sum(x*y for x, y in zip(a, b))

for v in ev["stored_vectors"]:
    print(round(dot(q, v["values"]), 4), v["metadata"]["text"])
```

Two vectors, where **one** document was ingested. Both retrievable for the same question,
0.003 apart in relevance — the superseded policy and the current one, with nothing in the
design preferring either.

## About the templates

**Identified by function and approximate view count, never by name, id, or author.** The
findings are about the ecosystem's defaults — about what a demo-shaped system does when it
reaches production — not about individual builders, who built these to demonstrate a tool and
largely succeeded at that.

## Licence

- **`checklist/`** — [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt
  it, build on it commercially. Attribution appreciated.
- **Article, exhibits, evidence, and method notes** — standard copyright, all rights reserved.
  **Quoting with a link is welcomed** and needs no permission; wholesale reproduction is not.

## Contact

We audit exactly this — the same checklist, the same evidence rules, the same error record,
turned on a real stack instead of the ecosystem's defaults.

If you run the checklist yourself and it finds something, we would genuinely like to hear what.
If you think a finding here is wrong, tell us — the correction record is the part of this work
we are most willing to add to.
