> **Note:** Sections marked **Instructions:** are scaffolding to guide you. Delete every
> instruction block (and any Example Table) once you've filled in the section, so the final
> document contains only your specifications.


# Objective

> **Instructions:** Write a short paragraph describing the workflow, as you would to a
> wet-lab colleague who is going to use it. No tool names, no file formats, no code.
>
> *Delete this instruction block once you've completed this section.*

# Inputs

> **Instructions:** Describe the starting data, including:
>
> 1. What files does the pipeline need?
> 2. How is the gene specified?
> 3. What fields from the various files do you need?
>
> *Delete this instruction block once you've completed this section.*

# Outputs

> **Instructions:** Write what should be created per region specified:
>
> *Delete this instruction block once you've completed this section.*

Per region:





# Pipeline Steps

> **Instructions:** Fill in one row per process in the pipeline. Derive the step list from
> the Objective and Outputs sections above. "Depends on" should name the upstream step(s)
> whose output this step consumes (used for wiring channels). "Existing tool?" names the
> tool you are using, or says that you are writing the step yourself.
>
> The Example Table below is a simple worked example (not from this pipeline) showing the
> expected format: a process that downloads a genome, followed by a process that runs a
> script on it. **Delete the Example Table and this instruction block before you submit.**

## Example Table

| Step | Input type | Depends on | Existing tool? |
|---|---|---|---|
| Download genome | Accession ID | - | `wget` |
| Run script on genome | Genome FASTA | Download genome | written by us |

## Pipeline Table

| Step | Input type | Depends on | Existing tool? |
|---|---|---|---|
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |

# Design Choices

> **Instructions:** Leave this section empty for your first generation. During review,
> each `SPECIFY` decision becomes a sentence here.
>
> *This is the one instruction block you may leave in place before generating. Delete it
> once you add your first design choice.*

# Development Workflow

Every process defines a `stub:` run that produces placeholder output files.

# Environment and Reproducibility

> **Instructions:** Decide how you want to handle reproducibility for this simple
> pipeline. Fill in one row per tool in the Pipeline Table, with the exact version and
> how it is provided (a container image or a conda package). Never use `latest`.
>
> *Delete this instruction block once you've completed this section.*

| Tool | Version | Container / conda | Used by step |
|---|---|---|---|
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |

# Resource Requirements

> **Instructions:** This is a bit trivial now but you can think about how you might
> adjust this for the next lab
>
> *Delete this instruction block once you've completed this section.*

| CPUs | RAM | Label | Rationale |
|---|---|---|---|
|  |  |  |  |
|  |  |  |  |

# Success Criteria

The stub run correctly produces named placeholder outputs for every process.

Every line in Design Choices is checkable against the output.

> **Instructions:** Add any further criteria your group agrees on.
>
> *Delete this instruction block once you've completed this section.*

# Out of Scope

> **Instructions:** I've given you a few out of scope to start just to
> limit things to a reasonable degree
>
> *Delete this instruction block once you've reviewed this section.*

- Pipeline only needs to consider one genome and one gene at a time
- Pipeline only needs to return location information - GC content,
  self-complementarity, and other metrics are out of scope
- Only consider NGG canonical PAM sites
