# UCSC BED format

The BED file here uses UCSC's 12-column ("BED12") format: one transcript per
line, tab-separated. Coordinates are **0-based, half-open**, so a feature with
`chromStart = 100` and `chromEnd = 210` covers 1-based positions 101–210.

| # | Field | Description |
|---|---|---|
| 1 | `chrom` | Chromosome or scaffold name (e.g. `chr1`) |
| 2 | `chromStart` | Start of the transcript (0-based, inclusive) |
| 3 | `chromEnd` | End of the transcript (exclusive) |
| 4 | `name` | Transcript name (e.g. `synthetic.1`, `ENST00000456328.2`) |
| 5 | `score` | Score from 0 to 1000; often `0` and unused |
| 6 | `strand` | `+` or `-` |
| 7 | `thickStart` | Start of the coding region (CDS) |
| 8 | `thickEnd` | End of the coding region (CDS). If `thickStart == thickEnd`, the transcript is non-coding |
| 9 | `itemRgb` | Display colour as `R,G,B` (e.g. `0,0,0`) |
| 10 | `blockCount` | Number of blocks (exons) |
| 11 | `blockSizes` | Comma-separated exon lengths |
| 12 | `blockStarts` | Comma-separated exon starts, **relative to `chromStart`** |

Notes:

- `thickStart`/`thickEnd` span the whole CDS from the first coding base to the
  last, so they include any introns in between. Use the blocks (fields 10–12) to
  find the exons.
- The start and end coordinates are always given on the + strand, even for
  minus-strand transcripts.
- Exon *i* covers `chromStart + blockStarts[i]` to
  `chromStart + blockStarts[i] + blockSizes[i]`.

## Example: `synthetic.bed`

```
chr1  100  210  synthetic.1  0  +  100  210  0,0,0  1  110,  0,
```

`synthetic.1` is a single-exon, plus-strand transcript on `chr1`. Its CDS covers
the whole transcript: 1-based positions 101–210, or `chr1:101-210` as a
samtools region.

Reference: [UCSC BED format specification](https://genome.ucsc.edu/FAQ/FAQformat.html#format1)
