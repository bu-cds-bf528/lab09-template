# Review Log — worked example

This is an example from the RNA-seq pipeline you built earlier in the course, so
you can see the conventions on something familiar without seeing the answers to
your own lab. The pipeline is: FASTQC, build a STAR index, align with STAR,
quantify with VERSE, concatenate the counts, parse the GTF for gene names, and
aggregate with MultiQC.

Every block needs a **Decision** and **Evidence** from you.

**Decision** is one of:

- `ACCEPT` — the choice is correct and the behaviour is what you intended.
  Nothing changes
- `PATCH` — a small fix you make yourself. The spec doesn't need to say it
- `SPECIFY` — this choice was important enough that you should have specified it
  up front. Write the sentence into the **Design Decisions** section of
  `specifications.md`, then have the agent change the code to match

The difference between `PATCH` and `SPECIFY` is not how big the change is. It is
whether the choice would have to be made the same way again. If a rebuild of this
pipeline could reasonably come out differently, and that would matter, it's a
`SPECIFY`.

**Evidence** is what convinced you. Acceptable evidence:

- a link to the specific section of the tool's documentation (not just the homepage)
- a command you ran and the relevant line(s) of its output
- a file you inspected (a FASTQ header, a log, a report) and what it showed

"Looks right", "the agent said so", and "standard default" are not evidence.

One test: **if this choice were wrong, which line of your evidence would look
different?** If the answer is none, you have an assumption, not evidence.

## How to work through this file

1. **Triage.** Read every block once. For each, ask: *if this choice were wrong,
   would the counts matrix change — for any input, not just this one?* If no,
   move the block to `## Dismissed` at the bottom.
2. **Group what's left.** Add a `## Group: <name>` header for each set of related
   decisions and move their blocks under it. Name the group after the thing being
   decided (`## Group: how a read is assigned to a gene`), not after the process.
3. **Resolve each group as a unit.** Write all of its spec sentences, have the
   agent make all of its code changes at once, fill in the evidence, delete the
   `REVIEW` markers from the code, and commit the whole group as one commit.

Check what's left to review with:

```bash
grep -rn "REVIEW " modules/ bin/ main.nf
```

---

## Group: how a read is assigned to a gene

### VERSE.strandedness — unstranded counting

- **Unit:** VERSE
- **Location:** `modules/verse/main.nf:21`
- **Choice:** no `-s`, so VERSE's default unstranded mode applies: a read counts
  toward a gene regardless of which strand it came from
- **Alternatives:** `-s 1` (forward) or `-s 2` (reverse), matching the library
  protocol
- **Why not in spec:** the spec names the library kit in Inputs but says nothing
  about how strand information is used
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** ran one sample three ways. Assigned reads: unstranded 94.1%,
  `-s 1` 3.8%, `-s 2` 91.6%. The near-total loss under `-s 1` and the near-match
  under `-s 2` identify this as a reverse-stranded library, which agrees with the
  kit named in the spec. Unstranded still assigns slightly *more* reads than the
  correct setting, because it also counts antisense reads — so the wrong choice
  looks better by the one summary statistic you would check. Nothing in the
  output flags it. Spec sentence added: "Counting is reverse-stranded (`-s 2`),
  matching the library preparation."

### VERSE.feature-type — counting over exons, grouped by gene_id

- **Unit:** VERSE
- **Location:** `modules/verse/main.nf:22`
- **Choice:** `-t exon -g gene_id`, VERSE's defaults
- **Alternatives:** `-t CDS` to exclude UTRs; `-g gene_name` to group by symbol
- **Why not in spec:** spec says "count reads per gene" without saying what a
  gene is in the GTF
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** `awk '$3=="exon"' gencode.gtf | wc -l` and the same for `CDS`
  give 1,443,281 and 859,114 — different feature sets, so the choice is not
  cosmetic. Counting over exons includes UTR-mapping reads, which is what we want
  for expression. Grouping by `gene_id` rather than `gene_name` also matters:
  `cut -f9 gencode.gtf | grep -o 'gene_name "[^"]*"' | sort -u | wc -l` returns
  fewer unique values than the gene_id count, so symbols are not unique and
  grouping by them would silently merge genes. Spec sentence added: "Reads are
  counted over `exon` features and grouped by `gene_id`."

### STAR_ALIGN.multimappers — default multimapping limit, multimappers unassigned

- **Unit:** STAR_ALIGN
- **Location:** `modules/star_align/main.nf:24`
- **Choice:** no `--outFilterMultimapNmax`, so STAR's default of 10 applies;
  reads mapping to more than 10 loci are not output, and VERSE is not given
  `--multiMapMode`, so reads mapping to several genes are discarded rather than
  counted
- **Alternatives:** a different multimap limit; fractional assignment of
  multimappers in VERSE
- **Why not in spec:** neither the spec's Alignment nor its Quantification
  section mentions multimapping reads
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** `Log.final.out` for one sample reports 8.2% of reads as "mapped
  to multiple loci" and 0.4% as "mapped to too many loci". Those 8.2% are aligned
  and then dropped at counting — they appear in the alignment rate in MultiQC but
  not in the counts matrix, so the two numbers describe different read sets.
  Discarding them is the conventional choice for differential expression and we
  are keeping it, but a rebuild that enabled `--multiMapMode` would produce a
  different matrix with no error. Spec sentence added: "Reads mapping to more
  than one gene are not counted; STAR's default multimapping limit of 10 is
  used."

---

## Group: what the counts matrix contains

### CONCAT.join-behaviour — inner join across samples

- **Unit:** CONCAT
- **Location:** `bin/concat_counts.py:19`
- **Choice:** per-sample counts are merged on gene_id with an inner join, so only
  genes present in every sample's VERSE output reach the matrix
- **Alternatives:** an outer join filling missing genes with zero
- **Why not in spec:** spec says "combine the per-sample counts into a matrix"
  without saying what to do with genes that appear in some samples and not others
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** `wc -l` on each sample's VERSE output gives the same row count
  for all eight samples, so the join is currently lossless and the choice is
  invisible — `wc -l counts_matrix.tsv` matches. It stops being invisible the
  moment a sample fails partway or a different annotation is used, and an inner
  join would then silently shrink the matrix rather than erroring. Switching to
  an outer join with zero fill. Spec sentence added: "Per-sample counts are
  merged on gene_id with an outer join; genes absent from a sample are recorded
  as zero."

### CONCAT.row-order — rows in the first sample's order

- **Unit:** CONCAT
- **Location:** `bin/concat_counts.py:23`
- **Choice:** the output matrix keeps whatever gene order the first sample's
  VERSE output had
- **Alternatives:** sort by gene_id; sort by genomic position
- **Why not in spec:** ordering not mentioned
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** "the first sample" is whichever one the channel happened to emit
  first, which Nextflow does not guarantee between runs. Ran the pipeline twice
  on the same input: `md5sum counts_matrix.tsv` differed, while
  `sort -k1,1 counts_matrix.tsv | md5sum` matched. The numbers are identical and
  the file is not — which breaks any byte-level reproducibility check, and makes
  two runs look different when they are not. Spec sentence added: "Rows in the
  counts matrix are sorted by gene_id."

### PARSE_GTF.id-versioning — version suffix kept on gene IDs

- **Unit:** PARSE_GTF
- **Location:** `bin/parse_gtf.py:14`
- **Choice:** gene IDs are written as they appear in the GTF, including the
  version suffix (`ENSG00000139618.15`)
- **Alternatives:** strip the version so IDs join against databases that omit it
- **Why not in spec:** spec says the mapping file has gene IDs and gene names,
  without saying which form
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** `head -2 gene_map.tsv` shows versioned IDs, and so does
  `cut -f1 counts_matrix.tsv | head -2` — so the two files join correctly to each
  other, which is what the pipeline needs. They will not join against anything
  that drops versions, which is most downstream annotation sources. Keeping the
  versioned form because it must match the matrix, and stating it so the
  constraint is visible rather than discovered later. Spec sentence added: "Gene
  IDs are reported with their GENCODE version suffix in both the counts matrix
  and the gene name mapping."

---

## Group: how the index is built

### STAR_INDEX.sjdbOverhang — left at the default of 100

- **Unit:** STAR_INDEX
- **Location:** `modules/star_index/main.nf:18`
- **Choice:** no `--sjdbOverhang`, so STAR's default of 100 applies
- **Alternatives:** read length − 1, which the STAR manual gives as the ideal
- **Why not in spec:** spec says "build a STAR index from the genome and GTF" and
  gives no parameters
<!-- student -->
- **Decision:** SPECIFY
- **Evidence:** the STAR manual states the ideal value is read length − 1 and
  that 100 works well in most cases. Our reads are 100 bp
  (`head -2 sample1_R1.fastq | tail -1 | wc -c`), so the ideal is 99 and the
  default of 100 is effectively correct here. It would not be for 50 bp reads,
  and nothing in the pipeline reads the read length to check. Keeping 100 and
  recording why. Spec sentence added: "The STAR index is built with
  `--sjdbOverhang 100`, appropriate for the 100 bp reads named in Inputs."

---

## Group: input handling

### workflow.sample-keying — samples keyed on the samplesheet name column

- **Unit:** workflow
- **Location:** `main.nf:14`
- **Choice:** records are built as `record(name: row.name, r1: ..., r2: ...)` and
  joins are on `name`; nothing checks that names are unique
- **Alternatives:** validate uniqueness up front; key on a composite of name and
  run ID
- **Why not in spec:** spec gives the samplesheet columns but not what guarantees
  they carry
<!-- student -->
- **Decision:** PATCH
- **Evidence:** duplicated a samplesheet row with the same name and different
  FASTQs: the run completed and silently published one sample's BAM over the
  other's — `results/` held a single `sample1.bam` with no warning anywhere in
  `.nextflow.log`. Patched `main.nf:14` to collect the names and fail with a
  clear message if any repeat. Not a spec change: the spec already implies one
  row per sample, and this enforces what it says.

### STAR_ALIGN.paired-end-assumed — both FASTQs required

- **Unit:** STAR_ALIGN
- **Location:** `modules/star_align/main.nf:22`
- **Choice:** `--readFilesIn $sample.r1 $sample.r2`; a record missing `r2` fails
  at channel construction
- **Alternatives:** detect single-end input and pass one file
- **Why not in spec:** the Inputs section lists r1 and r2 columns but does not say
  whether r2 may be empty
<!-- student -->
- **Decision:** ACCEPT
- **Evidence:** blanked the `r2` value in one samplesheet row: the run fails in
  `main.nf` at record construction with a null path, before any compute is spent.
  Failing immediately on malformed input is what we want, and the samplesheet
  format in the spec's Inputs section already requires both columns.

---

## Dismissed

Decisions that cannot change the counts matrix, for any input.

### config.outdir — output directory

- **Unit:** config
- **Location:** `nextflow.config:6`
- **Choice:** `params.outdir = 'results'`, relative to the launch directory
- **Alternatives:** `${projectDir}/results`
- **Why not in spec:** spec gives no output location
<!-- student -->
- **Decision:** ACCEPT
- **Evidence:** changing where a file is written cannot change its contents.

### MULTIQC.report-title — report title left unset

- **Unit:** MULTIQC
- **Location:** `modules/multiqc/main.nf:14`
- **Choice:** no `--title`, so MultiQC uses its default heading
- **Alternatives:** pass the project name
- **Why not in spec:** not mentioned
<!-- student -->
- **Decision:** ACCEPT
- **Evidence:** the title is a string in the HTML header; the parsed metrics in
  `multiqc_data/multiqc_general_stats.txt` are identical either way.

### STAR_ALIGN.bam-sorting — coordinate-sorted BAM output

- **Unit:** STAR_ALIGN
- **Location:** `modules/star_align/main.nf:25`
- **Choice:** `--outSAMtype BAM SortedByCoordinate`
- **Alternatives:** unsorted BAM, then sort separately
- **Why not in spec:** output format not stated
<!-- student -->
- **Decision:** ACCEPT
- **Evidence:** VERSE reads either, and sorting reorders records without changing
  which alignments exist — `samtools view -c` matches on sorted and unsorted
  output of the same alignment. Worth noting this would *not* be dismissible if
  we were counting with a tool that required name-sorted input for paired reads.

### config.resource-labels — resource values per label

- **Unit:** config
- **Location:** `nextflow.config:14`
- **Choice:** `process_high` = 16 CPUs, 64 GB for STAR_INDEX and STAR_ALIGN;
  `process_single` = 1 CPU, 4 GB for everything else
- **Alternatives:** separate labels per process
- **Why not in spec:** Resource Requirements gives labels but no values
<!-- student -->
- **Decision:** ACCEPT
- **Evidence:** resources affect whether a job completes, not what it computes. A
  STAR run that exceeds memory is killed and reported as a failed task rather
  than silently producing a worse alignment.

  One thing worth recording even though this is dismissed: STAR's
  `--runThreadN` has to match the label's CPUs, or STAR will request threads the
  scheduler has not granted. `modules/star_align/main.nf:21` passes
  `--runThreadN $task.cpus`, so they track automatically.