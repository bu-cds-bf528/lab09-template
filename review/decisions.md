# Review Log

Every block below needs a **Decision** and **Evidence** from you.

**Decision** is one of:

- `ACCEPT` — the choice is correct as it stands, and you are signing off on the
  choice
- `PATCH` — a small fix you make yourself. The spec does not need to say it, and
  it is not worth regenerating multiple files to fix.
- `SPECIFY` — an important enough decision that it should be in the specifications.
   Write the sentence into `specifications.md`, then have the agent update the code
   where the specs have changed.

**Evidence** is what convinced you. Acceptable evidence:

- a link to the specific section of the tool's documentation (not just the homepage)
- a command you ran and the relevant line(s) of its output
- a file you inspected (a FASTA header, a BED line, a tool's log) and what it showed

"Looks right", and "the agent chose this", are not evidence.


## How to work through this file

1. **Triage.** Read every block once. For each, ask: *if this choice were wrong,
   would the reported guides change for any input, not just this one?* If no,
   move the block to `## Dismissed` at the bottom and fill in the `Decision` and
   `Evidence`.
2. **Group what's left.** Add a `## Group: <name>` header for each set of related
   decisions and move their blocks under it. Name the group after the thing being
   decided (`## Group: quality control thresholds`), not after the
   process.
3. **Resolve each group as a unit.** Write all of its spec sentences, have the
   agent make all of its code changes at once, fill in the evidence, delete the
   `REVIEW` markers from the code, and commit the whole group as one commit.

---

## Required checkpoints

### R01 — Pipeline only uses one genome, and one transcript at a time

- **Applies to:** the workflow structure
- **Question:** Does the pipeline work only on a single gene and transcript at a time?
<!-- student -->
- **What breaks if wrong:**
- **Decision:** ACCEPT | PATCH | SPECIFY
- **Evidence:**

---

## Agent decisions

AGENT: append one block per decision below this template. Fill in only the fields
above the `<!-- student -->` line.

```markdown
### <UNIT>.<slug> — <short title>

- **Step:** <step name from the Pipeline Steps table, or workflow / config>
- **Unit:** <process name, or workflow / config>
- **Location:** `<file>:<line>`
- **Choice:** <exact value/flag/operator chosen>
- **Alternatives:** <what else could have been chosen>
- **Why not in spec:** <what the spec left unstated>
- **If wrong:** <concrete consequence>
<!-- student -->
- **Decision:** ACCEPT | PATCH | SPECIFY
- **Evidence:**
```
