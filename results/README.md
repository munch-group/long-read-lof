# Primate LoF / gene-status datasets

Generated with `intersect_primate_lof.py` (see ../CLAUDE.md). Each dataset is the
standardised output of the tool run over one data source. Generated 2026-06-14.
**How every input was produced and combined: see `../METHODS.md`.**

All matrices are gene × lineage and 0/1 (`loss_matrix` / `matrix`) drop straight
into the group's `geneset.py` UpSet. Lineage = species (TOGA, collapsed most-severe
across assemblies) or the source's own lineage label (Mao/Yoo). Each matrix begins
with leading **metadata columns** (`gene`, `chrom`, and for Mao also `gene_id` +
`gene_type`) before the lineage 0/1 columns — skip them when slicing lineages.

**Output format.** The genome-wide TOGA datasets (`toga_primates_species.*`) are
pd-lfs multi-file **parquet directories** (`<name>.parquet/`); all other (smaller)
datasets are plain `.tsv`. Read parquet with `pd_lfs.parquet.read_parquet`.

**Output layout.** Files derived from a curated query gene list are grouped in a
**per-query folder** `results/<query-stem>/`, named by variant inside:

```
results/
  recombination_genes/        cuts of recombination_genes.txt
      toga.*  toga_strictL.*  toga_chrX.*  toga_chrX_strictL.*  mao.*  mao_chrX.*
      human.*  merged.*       (human SV-LoF + cross-source merge)
  mao2024/                    full Mao 2024 source reformats
      gene_disrupted.*  gene_disrupted_chrX.*  recurrent_disrupted.*
  yoo2025/                    full Yoo 2025 source reformat
      lineage_specific.*
  human_1kgp_sv/              human gene LoF from 1000G long-read SVs
      lof.*
  merged/                     unified cross-source matrix (TOGA+Mao+Human)
      loss_matrix.tsv  loss_matrix.coverage.tsv
  toga_primates_species.*           genome-wide walk (flat at results/ root)
  toga_primates_species_chrX.*      chrX cut of the genome-wide walk
```

The tool nests automatically: a **bare** `--out` label (no `/`) is written to
`results/<query-stem>/<label>.*`; an `--out` value **containing `/`** is written
verbatim, flat (used for the genome-wide walk and for the source-named folders
above). See `../METHODS.md`.

## Inputs

| Source | What | Mode |
|---|---|---|
| TOGA hg38 reference, Primates (169 assemblies, 130 species) | genome-wide intact/lost/missing/paralog per gene × species | `--toga-dir` |
| Mao et al. 2024 *Cell* Data S2 (`../mao2024_suppl.xlsx`) | fixed-SV gene disruptions in 8 NHPs | `--table` |
| Yoo et al. 2025 *Nature* Suppl. (`../data/yoo2025/…MOESM4…xlsx`, sheet 40) | lineage-specific **novel/gained** genes in 6 apes | `--table` |
| 1000 Genomes long-read SV catalog (1KG_ONT_VIENNA; Schloissnig 2025) | **human** gene LoF from polymorphic SVs, 967 samples / 5 superpops | `--table` (via `annotate_sv_lof.py`) |

Mao 2024 Data S2 and Yoo 2025 are **presence** tables (no TOGA-style status) →
0/1 `matrix.tsv`. TOGA and the human SV source are **status-aware** →
`status_matrix` + `loss_matrix`. **Humans appear as a LOSS source only here** —
TOGA/Mao use human as the *reference*, so they have no human column.

## Annotation (`../data/hg38_toga_genes.tsv`)

TOGA `loss_summ_data.tsv` GENE rows are bare `ENSG…` with no symbol/chromosome.
Built `id,gene,chrom` from Ensembl GRCh38 release-110 GTF (`gene` features).
Coverage of TOGA's 19,464 genes: **99.9% chrom, 99.5% symbol** (843 on chrX).
91/19,457 genes have no symbol and remain as `ENSG…` in the outputs.

## Datasets

### TOGA — genome-wide, species-collapsed  `toga_primates_species.*`  (results/ root)

Written as **pd-lfs multi-file parquet directories** (`<name>.parquet/` of
part-*.parquet + `_manifest.json`) so they fit on GitHub. Read back with
`pd_lfs.parquet.read_parquet("…/<name>.parquet")` (local path or https URL; dtypes
restored from the manifest). Regenerate via the wrapper (see `../METHODS.md`):
```
./run_toga_walk.sh data/all_toga_genes.txt results/toga_primates_species
```
(the second arg contains `/`, so it is written verbatim at the `results/` root.)
- `toga_long.parquet/` — **full tidy table** (3,249,966 rows): `gene, chrom,
  lineage(species), status, assembly(dir), raw_id`. The reusable audit trail; re-run
  any cut from it. **23 MB** (was 293 MB as .tsv).
- `status_matrix.parquet/` — 19,457 genes × 130 species → status code (I/PI/UL/L/M/PM/PG).
- `loss_matrix.parquet/` — 19,457 × 130 → 0/1 (1 = UL/PG/PM/L/M; I/PI not loss). UpSet-ready.
- `intersections.parquet/` — one row per (gene, species): chrom, most-severe status,
  all_statuses (2,490,753 rows; 1.3 MB, was 79 MB as .tsv).

> The equivalent `.tsv` versions are still on disk (verified identical, 0 differing
> cells) but the two large ones (`toga_primates_species.toga_long.tsv`,
> `…intersections.tsv`) are **gitignored** — commit the `.parquet/` datasets. The
> small `…status_matrix.tsv` / `…loss_matrix.tsv` remain committable for `geneset.py`.

### TOGA — chrX only  `toga_primates_species_chrX.*`  (results/ root)
Derived from the genome-wide table via `--table` reuse, 843 X genes × 130 species.
Kept flat at the root alongside the genome-wide walk it is a cut of. The reuse reads
the parquet directly (the `--out` contains `/`, so it stays flat):
```
python intersect_primate_lof.py --query data/all_query_genes.txt \
  --table results/toga_primates_species.toga_long.parquet \
  --gene-col gene --lineage-col lineage --status-col status --chrom-col chrom \
  --chrom chrX --out results/toga_primates_species_chrX
```

### Mao 2024 — SV gene-disruption  `mao2024/gene_disrupted.*`  (+ `_chrX`)
4,868 genes × 8 NHPs (chrX cut: 247 genes). Presence (0/1). Coords are hg38.
These are full-source reformats (catch-all query), so they are grouped by source in
`results/mao2024/` via an explicit `/`-path `--out`:
```
python intersect_primate_lof.py --query data/all_query_genes.txt \
  --table data/mao_gene_disrupted.tsv \
  --gene-col GENE --lineage-col LINEAGE --chrom-col CHR [--chrom chrX] \
  --meta-cols GENE_ID:gene_id,GENE_TYPE:gene_type \
  --out results/mao2024/gene_disrupted[_chrX]
```

**Leading columns: `gene, chrom, gene_id, gene_type, <8 lineages…>`.** Mao's
disrupted-gene list spans *all* biotypes, not just protein-coding (only ~1,334 of
the 4,868 are `protein_coding`; the rest are lncRNA ~1,960, pseudogenes ~1,150,
snRNA/miRNA/…). The symbol-less ones keep their GENCODE clone names (`AC007993`,
1,792 of them, ~99.7% non-coding) — these are **not errors**, just loci HGNC never
named. `--meta-cols` carries the stable Ensembl id (`gene_id`, 100% populated, the
reliable join key) and `gene_type` (biotype, whitespace canonicalised to
underscore so `protein_coding` is one value) so you can join/filter downstream,
e.g. `df[df.gene_type == "protein_coding"]`. The 0/1 lineage cells are unchanged.

### Mao 2024 — recurrently-disrupted genes  `mao2024/recurrent_disrupted.*`
139 genes × 8 NHPs (sheet XXXIV); same leading `gene_id`/`gene_type` columns
(joined in from the gene-disrupted source — all 139 are present there). Also
exported as a flat gene list: `../data/mao_recurrent_genes.txt` (candidate curated
query set).

### Yoo 2025 — lineage-specific novel genes  `yoo2025/lineage_specific.*`
1,075 genes × 9 lineages (6 apes + Pan/Pongo clades + HSA). **Gene gains, not LoF.**
`chrom` here is the *query-assembly* contig (e.g. `CM055…`), not hg38 — no chrX filter.

### Human — 1000 Genomes long-read SV LoF  `human_1kgp_sv/lof.*`
4,544 genes × 8 human "lineages": three LoF tiers — `Human_anyLoF` (≥1 carrier),
`Human_commonLoF` (global AF ≥ 1%), `Human_homozygousLoF` (≥1 sample with both
copies hit) — plus five superpopulations `AFR/AMR/EAS/EUR/SAS_LoF`. Status-aware:
`HC_LOF` (deletion removing a canonical CDS exon / whole-gene deletion; or a complex
event with ≥50 bp net coding loss) vs `LC_LOF` (insertion-in-CDS, duplication-over-CDS,
complex). `loss_matrix` uses **`--loss-status HC_LOF`** (1 = high-confidence). Built
from the giggles biallelic SV callset by `../annotate_sv_lof.py` → `../data/human_1kgp_sv_lof.tsv`
→ engine. **1,733 genes** carry a homozygous HC-LoF genome-wide — consistent with
gnomAD's ~1,800 genes with homozygous pLoF.

> ⚠ **Polymorphic, not fixed.** Unlike TOGA/Mao (fixed inter-species differences),
> these are SVs *segregating within humans*. A 1 means the LoF allele exists in the
> 1000G cohort, not that the gene is lost in the species.
>
> ⚠ **Data-coverage caveat.** This is a **biallelic** graph-genotyped callset, so it
> under-ascertains segmental-duplication–mediated common gene deletions: famous human
> null CNVs (GSTM1, GSTT1, UGT2B17, CYP2D6, AMY1A) are absent or appear only as rare
> small events — verified by direct VCF inspection, not a pipeline error. GSTT1/LILRA3
> are also outside the TOGA gene universe. The aggregate signal is sound (essential
> genes RAD51/TP53/BRCA2 show no homozygous LoF; the ~1,733 homozygous-LoF gene count
> matches gnomAD).

### Merged — unified cross-source matrix  `merged/loss_matrix.tsv`
23,095 genes × 146 lineage columns, **source-prefixed** (`toga:` 130 species,
`mao:` 8 NHPs, `human:` 8 tiers/superpops) + leading `gene, chrom, gene_id`. Built by
`../merge_loss_matrices.py` (outer-join on `gene`; Yoo excluded — gains, non-hg38).
Companion `merged/loss_matrix.coverage.tsv` is a **tested-mask** (`<src>:_tested`)
distinguishing "tested & not lost" (0 in matrix, 1 in mask) from "gene not in that
source" (0 in both). For the human source the tested universe is **all coding genes**
(passed via `--universe human=…hg38_gene_cds_span.tsv`), so a 0 there means
scanned-and-clean, not unscreened.

## Curated cut — recombination genes  `recombination_genes/*`

Query: `../recombination_genes.txt` (104 lines). 95 unique genes resolved in TOGA.
- 5 stray non-gene tokens ignored: `DSB`, `formation`, `invasion`, `processing`, `Strand`.
- 13 alias/typo fixes applied via `../data/recombination_alias.tsv` (`--alias`):
  CTIP→RBBP8, FANCJ→BRIP1, FIRRM→C1orf112, HEI10→CCNB1IP1, HOP2→PSMC3IP, NBS1→NBN,
  RAD21L→RAD21L1, RAD54→RAD54L, SIX6OS1→C14orf39, SWS1→SWSAP1, TO6BL→C11orf80(TOP6BL),
  XPF→ERCC4, PPCH2→TRIP13(PCH2).
- 3 unresolved — **need your attention** (left out): `BLMM1A`, `SPO16` (distinct from
  SHOC1, already in list), `REDIC1`.

Files (built by `--table` reuse of `toga_primates_species.toga_long.parquet`,
prefiltered to the 95 genes in `../data/recomb_toga_long.tsv`), all under
`results/recombination_genes/`:
| Output | Loss def. | Scope |
|---|---|---|
| `toga.*` | default (UL/PG/PM/L/M) | 95 genes × 130 species |
| `toga_strictL.*` | **L only** (true loss) | 95 × 130 — *recommended* |
| `toga_chrX[_strictL].*` | both | only TEX11 is X-linked here |
| `mao[_chrX].*` | SV-disruption presence | 6 genes hit (HELLS, MLH1, RAD51, RAD51D, SPIDR, ZCWPW2); none on chrX |
| `human.*` | human SV-LoF (HC) | 24/95 genes carry a human LoF SV; **6 with homozygous HC-LoF**: IHO1, MEI4, SYCP1, FANCM, MEI1, MSH5 (all meiosis genes) |
| `merged.*` | cross-source union | 95 genes × 143 cols (`toga:`+`mao:`+`human:`); IHO1/MEI4/SYCP1 are homozygous-LoF in humans **and** lost in a TOGA primate |

### ⚠ Loss definition matters a lot here
For this essential-gene set the TOGA status mix is 79.7% I, 10.6% PI, 5.0% UL,
0.9% **L**, 0.5% M. The **default** loss set (UL+PG+PM+L+M) flags *all 95* genes as
"lost somewhere" and is dominated by `M` (assembly gaps) / `UL` in low-quality
assemblies (top hitters Rhinopithecus/Nasalis are the high-total-loss genomes).
Use **`recombination_genes/toga_strictL`** (`--loss-status L`) for real LoF: 51/95
genes, much cleaner. Even then, true-`L` calls for essential genes (RAD51, BRCA1…)
deserve manual checking — they can still be paralog/assembly artefacts.
`status_matrix.tsv` keeps the raw codes so you can choose any threshold.

To re-run with your own list / per-assembly resolution — a **bare** `--out` label
nests under `results/<query-stem>/` (so `recombination_genes.txt` → this folder):
```
python intersect_primate_lof.py --query recombination_genes.txt --alias data/recombination_alias.tsv \
  --table results/toga_primates_species.toga_long.parquet \
  --gene-col gene --lineage-col lineage --status-col status --chrom-col chrom \
  --loss-status L [--chrom chrX] [--lineage-col assembly] --out toga_strictL
```
(the `assembly` column in `toga_long.parquet` gives per-assembly resolution.)

Human SV-LoF cut + cross-source merge for the recombination set (see `../METHODS.md`
for the genome-wide build and the 1KG download):
```
# human LoF restricted to the recombination query (reuses the genome-wide annotation)
python intersect_primate_lof.py --query recombination_genes.txt --alias data/recombination_alias.tsv \
  --table data/human_1kgp_sv_lof.tsv --gene-col GENE --lineage-col LINEAGE \
  --status-col STATUS --chrom-col CHR --loss-status HC_LOF \
  --meta-cols GENE_ID:gene_id,SVTYPE:svtype,CONSEQUENCE:consequence,AF:af,N_HOM_ALT:n_hom_alt \
  --out human
# merge TOGA(strict L) + Mao + Human into one matrix + coverage mask
python merge_loss_matrices.py \
  --matrix toga=results/recombination_genes/toga_strictL.loss_matrix.tsv \
  --matrix mao=results/recombination_genes/mao.matrix.tsv \
  --matrix human=results/recombination_genes/human.loss_matrix.tsv \
  --universe human=data/hg38_gene_cds_span.tsv \
  --out results/recombination_genes/merged.loss_matrix.tsv
```

## Caveats
- TOGA species collapse = most-severe status across that species' assemblies
  (`--toga-label species`); per-assembly spread is in `toga_long.assembly`.
- Mao lineage cells were exploded to one (gene, lineage) per row; 3 malformed
  footer rows and a `OWL_monkey`→`Owl_monkey` case variant were cleaned.
- Yoo and Mao symbols are GENCODE/RefSeq; minor symbol drift vs TOGA is possible
  (no alias harmonisation applied — supply `--alias` if needed).
