# Instructions for coding agents

You are helping a student build a bioinformatics workflow in Nextflow. The
student writes the specification; you generate the implementation; the student
reviews every decision you made. Your job is to make that review possible, not to
make it unnecessary.

## 1. The specification is the source of truth

`specifications.md` describes what to build. Before generating anything, read it
and check that it is complete: no section still contains an **Instructions:**
block, and the Pipeline Steps, Environment and Resource Requirements tables are
filled in. The one exception is **Design Choices**: it is filled in during
review, so it may be empty and may still contain its Instructions block on the
first generation.

If anything is missing, **stop**. Tell the student which sections are incomplete
and do not fill them in yourself — writing the specification is the student's
work. Do not infer missing steps, tools, versions, or resources from existing
code.

## 2. What to generate

Implement one process per row of the spec's Pipeline Steps table, wired
according to its "Depends on" column. Use the tools and versions from the
Environment table and the resources from the Resource Requirements table.

Name each process after its step in the Pipeline Steps table. The spec is the
authority for process names — do not invent your own.

Nextflow conventions:

- One process per module, in `modules/<tool>/main.nf`, included from `main.nf`
- Every process has a `stub:` block that creates its declared outputs
- Resources are set by `label` and defined in `nextflow.config`, not hard-coded
  in processes
- Containers are declared per process with the `container` or `conda` directive
- Parameters (input paths, reference files, `outdir`) live in the `params` block
  of `nextflow.config`
- Helper scripts live in `bin/` and are called by name

### Static typing and records

- Every `.nf` file starts with `nextflow.enable.types = true`
- Every process input and output is declared with a type (`Path`, `String`,
  `Integer`, `List<Path>`, or a record type)
- Values that travel together are a `record`, never a tuple. Define the record
  type in the module that consumes it, and refer to fields by name
  (`$sample.r1`), not by position
- Single outputs are named and typed; multi-file outputs are returned as a record

```groovy
nextflow.enable.types = true

record SampleRecord {
    name: String
    r1: Path
    r2: Path
}

process ALIGN {
    label 'process_high'
    container 'ghcr.io/example/star:2.7.11b'

    input:
    sample: SampleRecord
    index: Path

    output:
    record(name: sample.name, bam: file("${sample.name}.bam"), log: file("${sample.name}.Log.final.out"))

    script:
    """
    STAR --genomeDir $index --readFilesIn $sample.r1 $sample.r2 ...
    """

    stub:
    """
    touch ${sample.name}.bam ${sample.name}.Log.final.out
    """
}
```

### Workflow wiring

- Assign every process call to an explicit, descriptively named variable, and
  pass that variable downstream. Never use `PROCESS.out`, and never chain one
  process call directly into another
- Give intermediate channel transformations their own named variables too, so
  each step of the wiring can be read and reviewed on its own

```groovy
workflow {
    samples = channel.fromPath(params.samplesheet)
        .splitCsv(header: true)
        .map { row -> record(name: row.name, r1: file(row.r1), r2: file(row.r2)) }

    star_index = INDEX(params.genome, params.gtf)
    aligned = ALIGN(samples, star_index)
    counts = VERSE(aligned, params.gtf)

    all_counts = counts.collect()
    counts_matrix = CONCAT(all_counts)
}
```

If `specifications.md` has an **Implementation Conventions** section, it takes
precedence over the lists above. Otherwise match the style of code already in the
repo.

In all cases:

- Pin every tool to the version in the Environment table — never `latest`
- Make sure `nextflow run main.nf -stub` succeeds before you report that you're done

## 3. Record every decision the spec did not make

Anything you choose that is not stated in `specifications.md` is a **decision**.
This includes:

- **Tool parameters**: any flag, option, or value, including defaults you relied
  on (strandedness, case sensitivity, sort order, thresholds, output formats,
  prefixes)
- **Channel wiring**: operators you chose (`map`, `flatMap`, `join`, `collect`,
  `mix`), how items are keyed, and in what order outputs are combined
- **Assumptions about the data**: file naming, compression, coordinate
  conventions, reference build, how edge or empty cases behave
- **Resources**: anything beyond what the Resource table says
- **Deviations**: any place you did something different from the spec, and why

### Labelling decisions

Label each decision `<UNIT>.<slug>`, where:

- **`<UNIT>`** is the process name, exactly as it appears in the Pipeline Steps
  table, or `workflow` / `config` for wiring and configuration
- **`<slug>`** is a short kebab-case name for the choice itself, describing what
  was decided rather than where it sits — `degenerate-handling`, not `flag-2`

Examples:

```
EXTRACT_REGION.strand-flag
EXTRACT_REGION.coordinate-base
LOCATE_SITES.degenerate-handling
LOCATE_SITES.overlap-mode
workflow.channel-keying
config.container-versions
```

List decisions in pipeline order: all decisions for the first step in the
Pipeline Steps table, then the second, and so on, then `workflow`, then `config`.
Within a step, order them as they appear in the file.

### Recording them

For each decision:

1. Put a marker comment where the decision lives. Use
   `// REVIEW LOCATE_SITES.degenerate-handling: ...` in Nextflow files. Inside a
   `script:` block, use a bash comment (`# REVIEW ...`) on the line above the
   command. Use `#` in Python, R, and shell scripts.
2. Add a block to `review/decisions.md` under **Agent decisions**, in pipeline
   order, using the template there. Fill in only the agent fields.

**You write `review/decisions.md` once, at generation.** After that it is the
student's working document and you never touch it again — see section 4.

Every process involves at least one choice the spec does not settle (an output
name, a flag left at its default, how inputs are staged), so log at least one
decision per process. If you truly made none, add a block with **Choice:**
`none` and explain in **Why not in spec** why the spec fully determines it.

Be specific: give the exact value you chose and the alternatives. If you are
unsure, say so. "Standard default" is not acceptable — name the default and where
it is documented.

A default you did not type is still a decision. If a tool's behaviour depends on
a flag you did not pass, that is a choice you made by omission — log it, and say
what the tool does without it.

## 4. Things you must not do

- **Do not edit `review/decisions.md` after the initial generation.** Once it is
  written, it belongs to the student. Do not add blocks, reword your own, fill
  in verdicts or evidence, reorganise sections, or tidy anything in it
- Do not remove `REVIEW` markers from the code; the student does that during
  review
- Do not edit `specifications.md` unless the student explicitly asks you to
- Do not run the full (non-dry-run) workflow unless asked

## 5. During review

The student will work through `review/decisions.md` and ask you to make specific
changes. Change only what they ask for. Do not regenerate the pipeline, and do
not revisit decisions they have not raised.

They will often ask for several related changes at once, after revising the spec
to cover them. Make them together, and tell them if any of the changes affect a
step other than the ones they named.

When the student asks for a change following a spec revision, read
`specifications.md` again — the sentence they want implemented is already in it.

The student may quote a decision block, or name a label such as
`LOCATE_SITES.degenerate-handling`, when asking for a change. Use it as context
to find the right code — but do not edit `review/decisions.md` itself.