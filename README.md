# Lab 09 - CRISPR-Cas9 Guide Selection with agentic coding

For today's lab, you will generate a nextflow pipeline that will return all the
possible CRISPR-Cas9 Guide RNA sites for a given sequence. Practically speaking,
your pipeline will use a genome, and a set of coordinates for a single location
of interest (gene / transcript) to extract the region and then scan for all
possible CRISPR-Cas9 sites that could generate a double stranded break within
the region. 

## Validating agent-generated workflows

Code an agent writes can look right and still be wrong. Here are three ways to
check it:

### 1. Tests

The first tool for checking any code is a test: give it a known input and check
the output against what you expect. For simple scripts that do well-defined
operations, ordinary unit tests work well. Does the script convert BED coordinates
to 1-based correctly? Does it fail on a missing transcript? We had a little
experience with this in a previous lab.

Pipelines are harder to unit test. Most of the work is done by external tools
(samtools, seqkit) chained together. The interesting bugs are usually in the
dependencies and in the flags chosen, not in any one function. Testing frameworks
like [nf-test](https://www.nf-test.com/) help, but you still have to know the right
answer to write the test. The core behaviour of established tools has also
already been tested and validated by their developers and users. You don't need
to re-test that `samtools faidx` extracts a region. You need to check that you
asked it for the *right* region where the definition of *right* region depends
entirely on what you plan to do with it.

### 2. Synthetic data you can check by eye

Make an input small enough that you can work out the answer by hand, then
compare it with the pipeline's output. That's what `data/synthetic.fa` and `data/synthetic.bed` are:
a 110 bp CDS where you can mark every `NGG` on both strands yourself (see
`data/synthetic_double_stranded.txt`). Synthetic data also lets you plant the edge cases you care
about on purpose and if the pipeline's count differs from yours, the difference
becomes a point for you to review.

### 3. Subset or spike-in real data

Synthetic data only tests the cases you thought to include. To get closer to a real
situation:

- **Subset real data.** Run on one real gene, or a small region of a real
  chromosome, and compare against an independent tool. For this pipeline that
  is ChopChop. Real data brings in edge cases you may not have originally
  accounted for.
- **Spike in known answers.** Insert sequences you know into real data (for
  example, a guide + PAM placed at a known position) and check that the
  pipeline finds them at the right coordinates. This gives you the realism of
  true data and still lets you know the right answer for those sites.

### How this looks in published bioinformatics

The same strategies show up in how new tools and methods are checked in the
literature. A typical methods paper uses more than one kind of ground truth,
because each one alone has blind spots:

- **Simulated (synthetic) data.** Reads or sequences are generated with a known
  true answer built in. Examples: simulating sequencing reads from a reference
  with tools like `wgsim` or ART, or simulating RNA-seq with known expression
  levels with Polyester. Because the true answer is known exactly, you can
  measure accuracy, sensitivity and false positives directly. The catch is that
  a simulation is only as realistic as its model of errors and biology.
- **Spike-ins and mock samples.** Known material is added to, or run alongside,
  real samples. Examples: ERCC RNA spike-ins at known concentrations in RNA-seq,
  or mock microbial communities with a known species mix in metagenomics. You
  get real lab and sequencing noise and still know the right answer for that
  part of the data.
- **Previously validated results.** The output is compared with reference
  datasets or with established tools. Examples: the Genome in a Bottle truth
  sets for variant calling, community benchmarks like CAMI for metagenomics,
  or results from a widely used existing tool. This shows the method agrees
  with what the field already trusts. It can't show the method is *better*
  where the reference itself is wrong.

This lab and the next follow the same pattern: synthetic data, then a real but
simple gene, then a check against ChopChop as the previously validated tool.

---

This is the first part of this lab where we will try another strategy for
generating workflow code with agents. You will write a `specifications.md` as you
have been doing in the rest of the class and use that as the context for an agent
to generate code for you.

The goal of this first part today is to generate a pipeline that will recover
all of the guide sites in the provided synthetic data. In the next lab, you will
extend your pipeline to work on a simple example of real data (a single gene),
and note where you may have under-specified how you want your pipeline to behave.

This repository has been setup to enforce a particular framework for how you might
want to develop a pipeline using agentic coding harnesses. This may seem tedious
but remember that the time saved by not writing the code / worrying about syntax
should now rightly be devoted to ensuring that your pipeline is operating how
you want.

Once the agent generates your pipeline, it will also generate a checklist for
you to validate and make decisions about your workflow in `review/decisions.md`.

## What's here

| File | Purpose |
|---|---|
| `AGENTS.md` | Agent instructions and Nextflow conventions (static typing, records, explicit workflow variables) |
| `CLAUDE.md` | Imports `AGENTS.md` for Claude Code |
| `specifications.md` | Spec template you fill in |
| `review/decisions.md` | Review log: your checkpoints, then the agent's decisions |
| `review/example_decisions.md` | An example based on a different pipeline |
| `review/comparison.md` | Your hand count of guides vs. what the pipeline reports, and your account of what's missing |
| `data/` | Synthetic FASTA, BED and double-stranded view of the gene; `data/README.md` describes the UCSC BED fields |
| `AGENT_SETUP.md` | Explanation of how the agent's instructions are loaded (`CLAUDE.md`, `@` imports, `/memory`, permissions) and what you shouldn't change |

## Read AGENTS.md (Focus on Sections 1 and 2)

You'll notice that I've encoded most of the conventions we've been following in
class in this AGENTS.md file that is being used as context to generate your
workflow. Remember that agents are *probabilistic* and are not guaranteed to
always and fully respect these guidelines: that's what makes manual review and
validation key when you are generating code with LLMs even with conventions like
AGENTS.md

## The loop

```
  1. SPECIFY          2. GENERATE          3. RUN               4. REVIEW              5. VERIFY
  specifications.md → agent writes code → hand count vs.    →  review/decisions.md → rerun, then
  (you write)         + logs decisions     pipeline output      triage from gaps      edge-cases.tsv
        ↑                                                             │
        └──────────────── SPECIFY: write it into the spec ────────────┘
```

### 1. Specify

Complete `specifications.md`. There are some light instructions but you should be
used to this by now. Assume you have been provided the FASTA (genome), the BED 
(gene coordinates), and you know the transcript name (synthetic.1). All of these
files are small enough that you can inspect them by eye. Leave **Design Choices**
empty for now; you will fill it in during review.

The goal of this workflow is to extract out the specific sequence of a gene from a
larger sequence and then identify potential CRISPR-Cas9 guides. Remember that a guide
is the 20 nt immediately 5' of the PAM, on the PAM-bearing strand, read 5'→3'.

Remember that your FASTA stores only the plus strand. A minus-strand
guide is already there; you just have to read it the other way, or simply reverse
complement the sequence.

### 2. Generate

Start your coding agent in this directory. It reads `AGENTS.md` (or `CLAUDE.md`)
automatically. For you, this command will be:

```bash
claude
```

Make a commit of your completed `specifications.md` to record your initial draft
of the document:

```bash
git add specifications.md
git commit -m "docs: initial specs"
```

Now have the agent attempt to generate the workflow by some variation of the command
in claude code:

```
Generate the workflow described in @specifications.md
```

Along with the code, the agent will add:

- a `REVIEW <UNIT>.<slug>` comment at every place it made a choice your spec didn't make
- a matching entry for each one under **Agent decisions** in `review/decisions.md`

Commit again at this point to record the initial state of your pipeline.

```bash
git add <new files>
git commit -m "agent: initial generation"
```

### 3. Run it once

Before you look at any output, work out the answer yourself. There are 8
possible PAM motifs in `data/synthetic_double_stranded.txt`. Mark every one on
both strands, and for each write the guide you'd expect: strand, position, and
the 20 nt sequence. You can use Ctrl+F to help you visually, but be aware of
the structure of DNA and how the sequence is encoded. Record these in
`review/comparison.md` under **My hand count**.

Then confirm the stub run works:

```bash
nextflow run main.nf -stub
```

Then actually run the pipeline:

```bash
nextflow run main.nf
```

Add the guides the pipeline reported to `review/comparison.md`. Your pipeline
will likely report fewer than 8. 

### 4. Review

`review/decisions.md` has one entry per choice your agent made. These differ
between all of you.

Agent decisions are labelled `<UNIT>.<slug>`, where the unit is a process name
(or `workflow` / `config`) and the slug says what was decided — for example
`FIND_TARGET_SITES.degenerate-handling` or `config.transcript-default`. The label tells
you where the decision lives and what it is about, so you can find it in the code
without looking it up.

This might be intimidating at first glance. When I used this framework, the agent
generated 30 choices for my review and as a reminder, this would be considered a
very small and relatively simple pipeline.

#### Triage first

Before resolving anything, go through the whole list once and ask of each item:

> **If this choice were wrong, would the set of reported guides change?**

Start with the decisions that could explain a difference in `review/comparison.md`.

1. Make a new section at the bottom `## Dismissed` and move any decisions that
you believe won't actually affect the biological results. Resolve each one with
`ACCEPT` and a one-line reason. This can be as simple as "The agent chose to use
conda environments in YML files with one tool and the version pinned".

2. For any decisions under `## Dismissed`, look at the **Location:** field, open
that file and line, and remove the `REVIEW` comment left in the code.

#### Handling the rest

Everything you didn't dismiss gets one of three choices:

- **`ACCEPT`** — the choice is correct and the behavior is what you intended
- **`PATCH`** — a small fix you make yourself. The spec doesn't need to say it.
- **`SPECIFY`** — this choice was important enough that you should have specified
  it up front. Write the sentence into `specifications.md` into Design Choices.


#### Adjusting your specifications.md

Don't work item by item. Go through your decisions and **group the related ones**.
Then handle each group as a unit: write all of its spec sentences, have the agent
make all of its code changes at once, and commit the whole thing together.

Grouping is a judgement call, and making it is part of the exercise. Decisions
about coordinate base, which end a position refers to, and how minus-strand sites
are reported are not three independent problems: they are one decision about how
you report a site, and specifying them separately is how specs end up
contradicting themselves.

Make the grouping visible: add a `## Group: <name>` header in `review/decisions.md`
and move each group's blocks under it. Name the group after the thing being decided
— `## Group: how a site is reported`. 

You can look at the example provided `review/example_decisions.md` for what I mean
by the above. 

For each group:

1. **Write the spec sentences.** Translate each change into how you would have
   specified it up front. "Handle lowercase properly" is not a specification but
   "Soft-masked bases are treated as valid target sites and all reported sequences
    are upper-cased" is. Add these to your Design Choices in your specifications.

2. **Ask the agent to make all of the group's changes at once**, from the revised
   spec. Doing them one at a time is tedious and may end up overwriting the same
   pieces of logic. 

In your claude code session:

```
The @specifications.md has been updated. Update the code based on the changes
```

3. **Fill in the evidence** for every decision in the group, including the
   `ACCEPT` and `PATCH` ones that belong to it. A `PATCH` records what you
   changed yourself.

4. **Delete the `REVIEW` marker** from the code for each resolved item. Keep
   everything in the `review/decisions.md` as written; this is your log of what
   the agent chose and what you decided. Go through the code locations listed
   and remove the `REVIEW` comments once you have made a decision or an agent
   has on your behalf.

5. **Commit the group as one commit**: the spec sentences, the code changes, and
   the resolved blocks in `decisions.md` are all one commit. The commit is what
   ties the group together and how you can still take advantage of git if the
   code produced has a bug or doesn't work properly. 

   ```
   specify how a site is reported: coordinate base, position end, strand
   ```

   A commit that changes code without changing the spec means the spec is now
   wrong. A commit holding two unrelated changes means you'll have to do extra
   work later on to reconcile what was changed when and why when git already
   solves this problem.

### 5. Verify

After each group commit, rerun the pipeline and add a row to the **Pipeline
output** table in `review/comparison.md`:

```bash
nextflow run main.nf -stub
```

```bash
nextflow run main.nf
```

When you have resolved every decision, compare your final output with your
hand count once more. Then copy the "known" answers:

```bash
cp /projectnb/bf528/materials/labs/lab09/edge-cases.tsv review/
```

For each guide in `edge-cases.tsv` that is not in your output, record in
`review/comparison.md` under **After edge-cases.tsv** which of these it is:

- a choice you made in the spec
- a choice your agent made that you accepted
- a behavior of a tool you used

## Extending this to your own projects

The specific framework here including `REVIEW` markers, labelled decisions, a 
review log is just one way of organizing a project generated by agentic coding.
What matters is that something forces the agent's unstated choices into the open,
and that you have a process for dealing with them. 

### Put your conventions in a file the agent reads

Every agentic coding tool reads a project instructions file automatically:
`AGENTS.md` for most, `CLAUDE.md` for Claude Code, `.cursorrules` for Cursor,
`.github/copilot-instructions.md` for Copilot. Several of them now read
`AGENTS.md` as a common format.

Put in it the things you would otherwise find yourself saying in every session:

- the language, framework and versions you use
- directory layout and naming conventions
- how to run the tests, the linter, the build
- things that are easy to get wrong in your codebase
- what not to touch

The longer this becomes, the more likely it is that LLMs may not perfectly follow
every instruction in it. Context is finite and attention across it is uneven.
Keep the file to the things you would otherwise repeat in every session.

### Separate specifying from generating

Writing down what you want before you ask for it is most of the benefit, and it
does not need any tooling. A paragraph in a markdown file, an issue, a comment at
the top of the module are all ways of doing this. What matters is that it exists
before the code does, and that it survives afterwards so you can tell whether the
code still matches it.

### Make the agent tell you what it decided

This is the part worth stealing. You can get most of it from a single instruction
in your project file:

> When you make a choice that my request did not specify, leave a comment at that
> line saying what you chose and what you considered instead.

Then `grep` for your marker. You do not need a review log, labels, or a
template — the point is only that choices made on your behalf end up somewhere
you can find them outside of the many lines of generated code. 

The same question does the triage: *if this choice were wrong, would the result
change?* Most of the time it will not, and you can accept certain design choices
that don't change the results (what you care about). Sometimes those design
choices are important enough that you want them enforced in a certain way. 

### Keep the diff small enough to read

Everything here assumes you actually read what came back. Try to ask for one
coherent change at a time, and commit at a point you would be willing to return to
if what was generated isn't what you want or isn't working correctly. 

### Know which parts you cannot delegate

An agent can write code that runs. It cannot tell you whether the result is
biologically sensible, whether the comparison you set up answers the question you
have, or whether a number that looks reasonable is correct.

The time you saved by not writing the code now needs to go into validating that
it does what you want and if it's correct. 
