# White Lab | Ella training 3 — RNA-seq QC, MultiQC and safe data cleanup

**Audience:** Ella, on Prometheus  
**Exercise:** Inspect and certify your **full-depth, 16-sample** TDP-43 nf-core/rnaseq result, understand MultiQC and build a safe, evidence-based storage cleanup plan.  
**Prerequisite:** Training 2 completed.  
**Next exercise:** Training 4, differential expression with nf-core/differentialabundance.

> **Three independent questions:** (1) Did Nextflow finish? (2) Are all the required files complete and scientifically plausible? (3) Which files can eventually be removed without destroying reproducibility? A “Pipeline completed successfully” line answers **only question 1**.

This exercise is patterned on the White Lab's formal HNRNPK_LOF and TDP43_LOF RNA-seq certifications. Your own analysis remains isolated under `/projects/users/ella/TDP43_LOF_training`. **No deletion should be done during the initial audit.** An approved cleanup can be performed later using the explicit safeguards below. Retain the raw FASTQs and Nextflow task work until both Training 3 QC and Training 4 DGE are complete.

## A. What each output means

| Output or location | What it tells us | Retention decision |
|---|---|---|
| `logs/nfcore_rnaseq_full_out_*.nextflow.log` | Pipeline progress and errors | **Keep** |
| `pipeline_info/execution_trace_*.txt` | Each scheduled task and its final state | **Keep** |
| `multiqc_report.html` | Interactive overview of read and alignment quality | **Keep** |
| `multiqc_data/` | Underlying numerical QC data | **Keep** |
| `fastqc/`, Trim Galore reports | Sequencing and trimming quality | **Keep** if published |
| `star_salmon/*.markdup.sorted.bam` and indexes | Genome alignments used for later splicing analysis | **Keep**, even after DGE |
| `star_salmon/salmon.merged.gene_counts.tsv` | Gene-level Salmon counts | **Keep** |
| `star_salmon/salmon.merged.gene_lengths.tsv` | Matched gene-length offsets for DESeq2 | **Keep** |
| Other published BAMs | Could be unique, derivative, duplicated or symlinked | **Audit before any removal** |
| `/scratch1/users/ella/..._full` | Regenerable task work used by `-resume` | Delete **only after certification and explicit approval** |
| `inputs/raw/*.fastq.gz` | Working copies of supplier sequencing | Offload **only after verifying RDS/archive source integrity** |
| `/scratch2/users/ella/nextflow_home`, runstate | Cached pipelines and run history | Keep during training; do not wholesale delete |

**Key principle:** a BAM may be very large but still essential for a future rMATS analysis. A FASTQ may be redundant locally but remain the source material needed for a later rerun. “Large” never means “safe to delete.”

## B. Locate your exact published run (read-only)

Log on to Prometheus, then:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
POINTER="$PROJECT/docs/full_rnaseq_last_outdir.txt"
[[ -s "$POINTER" ]] || { echo 'STOP: no full-run pointer; complete Training 2'; exit 1; }
OUT=$(cat "$POINTER")
echo "Published results: $OUT"
[[ -d "$OUT" ]] || { echo 'STOP: published results missing'; exit 1; }
find "$OUT" -mindepth 1 -maxdepth 2 -type d | sort
find "$OUT" -type f -name 'multiqc_report.html'
find "$OUT/pipeline_info" -maxdepth 1 -type f | sort
ls -lh "$OUT/star_salmon/salmon.merged.gene_counts.tsv" \
       "$OUT/star_salmon/salmon.merged.gene_lengths.tsv"
```

**PASS:** output lives under Ella's own `results/`, the two Salmon matrices exist and MultiQC + `pipeline_info` were published. **STOP:** missing output or a pointer into the production `/projects/active` directory.

## C. Read the actual Nextflow execution record

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
LOG="$PROJECT/logs/$(basename "$OUT").nextflow.log"
tail -n 55 "$LOG"
find "$OUT/pipeline_info" -maxdepth 1 -name 'execution_trace_*.txt'
```

A successful run should finish with an explicit completion statement and a nonzero count of successful tasks. Check the trace as well: each task should be `COMPLETED`, not `FAILED` or `ABORTED`. The historical TDP-43 production run completed **606 tasks** in **14h 3m 35s** using 467.9 CPU hours. These are helpful scale comparisons, **not performance targets or mandatory task counts**: run paths, cache reuse, process configuration and versions can change execution details.

### C1. Create a repeatable Stage 1 certification and checksum report

This script reads the trace, verifies the 16 canonical BAMs and indexes, checks the merged count/length matrices, locates MultiQC and records the key file hashes. It does **not** assume any specific human QC result or overwrite any pipeline output.

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
mkdir -p "$PROJECT/scripts" "$PROJECT/docs"
SCRIPT="$PROJECT/scripts/audit_full_rnaseq.py"
[[ ! -e "$SCRIPT" ]] || { echo 'STOP: audit script exists; inspect it before rerunning'; exit 1; }
cat > "$SCRIPT" <<'PY'
#!/usr/bin/env python3
from pathlib import Path
from collections import Counter
from datetime import datetime
import csv, hashlib, subprocess

base=Path('/projects/users/ella/TDP43_LOF_training').resolve()
out=Path((base/'docs/full_rnaseq_last_outdir.txt').read_text().strip()).resolve()
if out.parent != base/'results' or not out.name.startswith('nfcore_rnaseq_full_out_'):
    raise SystemExit('FAIL: unexpected output location')
log=base/'logs'/(out.name+'.nextflow.log')
if not log.is_file() or 'Pipeline completed successfully' not in log.read_text(errors='replace'):
    raise SystemExit('FAIL: successful Nextflow completion not established')
traces=list((out/'pipeline_info').glob('execution_trace_*.txt'))
if len(traces)!=1: raise SystemExit('FAIL: expected one trace')
with traces[0].open(newline='') as f:
    tasks=list(csv.DictReader(f,delimiter='\t'))
statuses=Counter((r.get('status') or '').strip().upper() for r in tasks)
if not tasks or set(statuses)!={'COMPLETED'}:
    raise SystemExit('FAIL: incomplete task trace '+str(statuses))
expected={f'TDP-{p}-{t}-{i}' for p in ('SCR','KD') for t in ('DMSO','11J') for i in range(1,5)}
bams=sorted((out/'star_salmon').glob('*.markdup.sorted.bam'))
observed={p.name.removesuffix('.markdup.sorted.bam') for p in bams}
if len(bams)!=16 or observed!=expected: raise SystemExit('FAIL: canonical BAM sample set')
samtools='/projects/micromamba/envs/hts-tools/bin/samtools'
if not Path(samtools).is_file(): raise SystemExit('STOP: ask Matt for validated samtools access')
for bam in bams:
    indices=[Path(str(bam)+'.bai'),bam.with_suffix('.bai'),Path(str(bam)+'.csi')]
    if not bam.stat().st_size or not any(p.is_file() and p.stat().st_size>0 for p in indices):
        raise SystemExit('FAIL: BAM/index '+str(bam))
result=subprocess.run([samtools,'quickcheck','-v',*map(str,bams)],capture_output=True,text=True)
if result.returncode: raise SystemExit('FAIL: BAM quickcheck '+result.stderr)
merged=out/'star_salmon'
counts=merged/'salmon.merged.gene_counts.tsv'
lengths=merged/'salmon.merged.gene_lengths.tsv'
if not counts.is_file() or not lengths.is_file(): raise SystemExit('FAIL: Salmon matrix missing')
with counts.open(newline='') as f: hc=next(csv.reader(f,delimiter='\t'))
with lengths.open(newline='') as f: hl=next(csv.reader(f,delimiter='\t'))
if hc!=hl or hc[:2]!=['gene_id','gene_name'] or len(hc)!=18 or set(hc[2:])!=expected:
    raise SystemExit('FAIL: gene matrices do not match 16 samples')
qc=list(out.rglob('multiqc_report.html'))
if not qc: raise SystemExit('FAIL: MultiQC report missing')
def digest(p):
    h=hashlib.sha256()
    with p.open('rb') as f:
        for block in iter(lambda:f.read(4*1024*1024),b''): h.update(block)
    return h.hexdigest()
lines=[
    'Ella FULL RNAseq technical QC',
    'Checked: '+datetime.now().astimezone().isoformat(),
    'Output: '+str(out),
    'All Nextflow tasks: '+str(len(tasks))+' '+str(dict(statuses)),
    'Canonical BAMs and indexes: 16/16 PASS',
    'samtools quickcheck: PASS',
    'Counts: '+str(counts),
    'Counts SHA256: '+digest(counts),
    'Lengths: '+str(lengths),
    'Lengths SHA256: '+digest(lengths),
    'Matching 16-sample matrix columns: PASS',
    'MultiQC: '+str(qc[0]),
    'STATUS: TECHNICAL PASS; biological QC still requires MultiQC inspection',
]
report=base/'docs'/('full_rnaseq_technical_QC_'+datetime.now().strftime('%Y%m%d_%H%M%S')+'.txt')
report.write_text('\n'.join(lines)+'\n')
print('\n'.join(lines)); print('Saved:',report)
PY
chmod 700 "$SCRIPT"
python3 -m py_compile "$SCRIPT" || exit 1
python3 "$SCRIPT"
```

**Expected result:** all trace tasks `COMPLETED`, **16/16** mark-duplicate BAMs with indexes, **16/16** matrix sample columns and one or more MultiQC HTML files. A technical PASS is not yet a blanket biological QC PASS.

## D. Learn to read MultiQC scientifically

MultiQC combines tool-specific reports into one interactive HTML file. It is not itself the aligner, quantifier or statistical model. It can highlight failed or unusual samples, but must be interpreted in light of library preparation and experimental design.

Locate the HTML report:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
find "$OUT" -type f -name 'multiqc_report.html' -print
find "$OUT" -type f -name 'multiqc_general_stats.txt' -print
```

To inspect it on your Mac, download the **small HTML report** using the file path printed above. Run `scp` on your **Mac**, not inside the Prometheus SSH session, and replace the placeholder with that **actual** path:

```bash
scp 'ella@10.131.34.252:/ACTUAL/OUTPUT/PATH/multiqc_report.html' ~/Downloads/ella_tdp43_full_multiqc.html
```

Open the downloaded HTML with your browser. If the file includes links to resources, keep its accompanying assets; normal MultiQC HTML is generally self-contained. Do not export entire BAMs or raw FASTQs to the laptop.

### D1. A guided MultiQC review

| Section to look for | What to examine | Why it matters / when to investigate |
|---|---|---|
| **General Statistics** | Library size; % mapped; % uniquely mapped; duplication; other available columns | Rapidly identifies group-wide or single-sample anomalies; some metrics come from different read denominators |
| **FastQC, raw reads** | Quality by base, read length, adapter content, overrepresented sequences, GC distribution | Establishes whether sequencing or contamination problems existed *before* trimming |
| **Trim Galore / Cutadapt** | % bases/reads trimmed and read retention | Excessive trimming may reduce usable material or reflect unexpected adapters/read quality |
| **STAR** | Uniquely mapped %, multiple mapped %, unmapped categories, chimeric reads if reported | High alignment to GRCh38 is reassuring; unusually low unique mapping may reflect contamination, poor quality or repeats |
| **Salmon** | Library type, assignment/quantification summaries, evidence of strandedness | Tests quantification compatibility and library orientation |
| **Picard MarkDuplicates** | Duplicate fraction per library | PCR duplication reduces independent information; interpretation depends on RNA abundance and library complexity |
| **RSeQC and strand checks** | Gene-body coverage, read distribution, inferred strandedness | Identifies 3′/5′ biases, contamination, incorrect orientation or RNA-integrity issues |
| **Sample correlations / exploratory QC** | Whether biological samples cluster sensibly | Large outliers could reflect technical issues, swaps, batch effects or strong biology |
| **Module warnings/absence** | Which analyses were actually produced | Not every module runs or has a general-statistics column; absence ≠ a PASS or FAIL by itself |

**Interpret the magnitude comparatively, not mechanically.** In the completed HNRNPK full-depth study, sample QC medians were approximately **77.1 million input read pairs**, **98.1% STAR mapped**, **94.1% uniquely mapped** and **24.1% Picard duplication**. Those are helpful, not universal cutoffs and not a prediction of your TDP-43 results.

Key questions for each metric:

1. Are all **16** samples represented (and is each sample named correctly)?
2. Are technical metrics broadly consistent across DMSO, 11J, KD, scramble and differentiation batches?
3. Does an apparent outlier occur across **several independent metrics** or just one?
4. Can changes be explained by biology (e.g. transcript abundance) or an experimental batch?
5. Do you need to review the originating FastQC/STAR/Picard log rather than interpreting a plot in isolation?

### D2. Record your own QC assessment

Create a simple file: `docs/full_rnaseq_MultiQC_interpretation.md`. Use the following headings and fill in the *actual values* from your report:

```markdown
# Ella full TDP-43 RNA-seq — MultiQC interpretation

- Run directory:
- MultiQC HTML:
- Number of biological samples present:
- Raw reads / read pairs: range and median:
- Trim Galore read retention: range and median:
- STAR overall and unique mapping: range and median:
- Picard duplicate fraction: range and median:
- RSeQC/Salmon strandedness consistency:
- Gene-body coverage / other warnings:
- Most unusual sample(s), and evidence:
- Is any unusual sample restricted to a particular condition or batch?
- QC decision: PASS / REVIEW REQUIRED / FAIL (give reasons):
- Checked with Matt (date):
```

This evaluation is an intellectual part of the exercise: don't copy the PI's QC conclusion without reading your own data.

## E. Understand why the BAMs and Salmon matrices matter

A `*.markdup.sorted.bam` contains alignments of reads to the genome; its index (`.bai` or `.csi`) allows tools such as IGV and rMATS to access genomic regions efficiently. These files will be required by the subsequent **splicing** workflow, even though the DGE pipeline uses only gene-count and gene-length matrices.

The primary DGE inputs are:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
head -n 3 "$OUT/star_salmon/salmon.merged.gene_counts.tsv" | cut -c 1-200
head -n 3 "$OUT/star_salmon/salmon.merged.gene_lengths.tsv" | cut -c 1-200
sha256sum "$OUT/star_salmon/salmon.merged.gene_counts.tsv" \
          "$OUT/star_salmon/salmon.merged.gene_lengths.tsv" \
  | tee "$PROJECT/docs/full_rnaseq_DGE_input_SHA256.txt"
```

These matrices must have identical gene IDs, ordering and sample columns, but the numbers have different meanings. Raw gene counts alone should **not** be handed to DESeq2 without the matched gene-length information in this workflow; see Training 4.

## F. Plan space recovery; do not start by deleting

Measure storage, and identify exactly where bytes reside:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
df -h /projects /scratch1 /scratch2
du -sh "$PROJECT/inputs/raw" "$OUT" \
       /scratch1/users/ella/nextflow_work/TDP43_LOF_training_full 2>/dev/null
find "$OUT/star_salmon" -maxdepth 1 -type f -name '*.bam' \
  -printf '%s %p\n' | sort -nr | head -n 35
find "$OUT/star_salmon" -maxdepth 1 -type l -ls
```

There are three **different** forms of apparent duplication to distinguish:

- **Raw FASTQ copies:** RDS/archive originals and Ella's local staging copies. Local FASTQs may eventually be removed once the authoritative originals and hashes have been verified.
- **Published BAMs:** BAMs with similar names (`*.sorted.bam`, `*.markdup.sorted.bam`, other derivatives) are **not automatically identical**. Some contain different alignment records or duplicate markings. Keep canonical `*.markdup.sorted.bam` plus index for downstream rMATS.
- **Nextflow work files:** staged links, sorted BAMs and other task outputs may consume large space. They enable `-resume`, and are treated differently from published final files.

### F1. Audit extra BAMs without touching them

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
find "$OUT/star_salmon" -maxdepth 1 \( -type f -o -type l \) -name '*.bam' \
  -printf '%y %s %p -> %l\n' | sort > "$PROJECT/docs/full_rnaseq_BAM_inventory.txt"
cat "$PROJECT/docs/full_rnaseq_BAM_inventory.txt"
```

For any proposed redundant BAM, evaluate **one exact pair at a time**:

```bash
# EXAMPLE ONLY: substitute two actual files identified in the inventory.
# Do not execute until you have identified valid paths.
# stat -c '%d %i %h %s %n' /ACTUAL/FILE_1.bam /ACTUAL/FILE_2.bam
# sha256sum /ACTUAL/FILE_1.bam /ACTUAL/FILE_2.bam
```

For example, to inspect a **specific** sorted BAM against the canonical mark-duplicate BAM without deleting either file, select a genuine sample name and run:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
SAMPLE=TDP-SCR-DMSO-1
CANDIDATE="$OUT/star_salmon/${SAMPLE}.sorted.bam"
KEEP="$OUT/star_salmon/${SAMPLE}.markdup.sorted.bam"
if [[ -f "$CANDIDATE" && -f "$KEEP" ]]; then
  stat -c '%d %i %h %s %n' "$CANDIDATE" "$KEEP"
  sha256sum "$CANDIDATE" "$KEEP"
else
  echo 'No matching BAM pair; review actual inventory instead.'
fi
```

If **both SHA256 values are identical**, the candidate may be a byte-identical duplicate. If the checksums differ, do **not** label it redundant: duplicate marking or other processing may have changed the BAM. When Matt confirms a particular **verified identical and unused** candidate may be removed, `rm -i -- "$CANDIDATE"` offers an interactive last safeguard; first confirm the variable points to the approved candidate path, not the canonical `KEEP` path. Repeat the identity/usage audit independently for each candidate rather than inferring all BAM pairs are equivalent.

Equal *filenames* or similar sizes do not prove identity. Equal full-file SHA256 hashes demonstrate byte-for-byte identity. If a proposed file is a symlink, examine its destination (`readlink -f`) rather than treating the link as another physical copy. **Never remove the 16 canonical mark-duplicate BAMs or their indices** during this exercise. Even after two candidate files have matching hashes, seek PI approval and verify no result, manifest or downstream process still refers to the candidate path before unlinking it. If in doubt, retain the BAM.

## G. FASTQ offload — verification and supervised action

**Sequence of work:** technical QC PASS → MultiQC review with Matt → DGE completed/certified (Training 4) → authoritative RDS **and** archive locations verified → no active pipeline/rerun requirement → supervised local FASTQ offload. Offload means removing *Ella's unnecessary local duplicates*, never the authoritative originals.

### G1. Independently verify local and RDS hashes

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
python3 - <<'PY'
from pathlib import Path
import hashlib,re
project=Path('/projects/users/ella/TDP43_LOF_training')
local=project/'inputs/raw'
source=Path('/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw')
n=0
for p in ('SCR','KD'):
 for t in ('DMSO','11J'):
  for i in range(1,5):
   for read in (1,2):
    name=f'TDP-{p}-{t}-{i}_R{read}_001.fastq.gz'
    for folder in (local,source):
     f=folder/name; side=folder/(name+'.md5')
     if not f.is_file() or not side.is_file(): raise SystemExit('STOP missing: '+str(f))
     m=re.search(r'[a-fA-F0-9]{32}',side.read_text())
     if not m: raise SystemExit('STOP invalid sidecar: '+str(side))
     h=hashlib.md5()
     with f.open('rb') as stream:
      for chunk in iter(lambda:stream.read(4*1024*1024),b''): h.update(chunk)
     if h.hexdigest()!=m.group().lower(): raise SystemExit('FAIL integrity: '+str(f))
    n+=1
print('PASS:',n,'local FASTQs and RDS originals match their own MD5 sidecars')
print('NEXT: Matt must confirm the distinct archive copy and explicitly authorise any local offload')
PY
```

This takes time because it reads **64 full FASTQ files** (32 local, 32 RDS). Save the terminal output with `tee` if using it for the deletion decision. RDS may be a network mount, so allow for transfer time. Verify the separate archive location using the production TDP-43 README and Matt; **don't invent an archive path** or treat two filenames as independent archive copies.

### G2. Local FASTQ removal — optional, only with Matt's explicit approval

Do this **after** Training 4, only after the archive and RDS source validation has been signed off and no live Nextflow job is using these files. This command is intentionally confined to **Ella's own** expected FASTQ filenames. It leaves MD5 sidecars and an offload manifest in `docs/`. It requires typed confirmation; do not run it during routine QC.

```bash
[[ "$(whoami)" == ella ]] || { echo 'STOP: wrong Unix account'; exit 1; }
PROJECT=/projects/users/ella/TDP43_LOF_training
RAW="$PROJECT/inputs/raw"
[[ -d "$RAW" && ! -L "$RAW" ]] || { echo 'STOP: unsafe raw path'; exit 1; }
[[ "$(readlink -f "$RAW")" == "$RAW" ]] || { echo 'STOP: raw path resolves elsewhere'; exit 1; }
pgrep -u ella -af '[n]extflow' && { echo 'STOP: Nextflow is active'; exit 1; }
[[ -f "$PROJECT/docs/full_rnaseq_DGE_input_SHA256.txt" ]] || { echo 'STOP: QC checksum evidence missing'; exit 1; }
# This shell block DOES NOT reverify the archive, MD5 or DGE status.
# Only continue if the full sign-off checklist above has independently passed.
echo 'REQUIRES: Matt approved deletion after RDS + archive checks and DGE certification.'
read -r -p 'Type APPROVED_REMOVE_ELLA_FASTQ_COPIES to continue: ' CONFIRM
[[ "$CONFIRM" == APPROVED_REMOVE_ELLA_FASTQ_COPIES ]] || { echo 'Cancelled'; exit 1; }
manifest="$PROJECT/docs/full_fastq_local_offload_$(date +%Y%m%d_%H%M%S).tsv"
printf 'filename\tbytes\n' > "$manifest"
for f in "$RAW"/TDP-{SCR,KD}-{DMSO,11J}-{1,2,3,4}_R{1,2}_001.fastq.gz; do
  [[ -f "$f" && ! -L "$f" ]] || { echo "STOP missing/symlink: $f"; exit 1; }
  printf '%s\t%s\n' "$(basename "$f")" "$(stat -c %s "$f")" >> "$manifest"
done
[[ $(($(wc -l < "$manifest")-1)) -eq 32 ]] || { echo 'STOP unexpected FASTQ count'; exit 1; }
echo "Proposed manifest: $manifest"
# Delete only the 32 exact checked local copies; never RDS or archive files.
for f in "$RAW"/TDP-{SCR,KD}-{DMSO,11J}-{1,2,3,4}_R{1,2}_001.fastq.gz; do
  rm -- "$f" || exit 1
done
echo 'Local offload complete; supplier MD5 sidecars and manifest retained.'
echo 'Any future rerun must reimport the original FASTQs and reverify MD5.'
```

**Critical:** that removal block is an *action*, not an audit. It cannot itself prove the archive exists or that Matt has approved it. After local offload, the historical samplesheet still records the original input paths but those FASTQs will no longer be present; **Nextflow resume/reprocessing from that samplesheet will not work until the files are restored**. This is why offload is scheduled after DGE and sign-off.

## H. Heavy Nextflow work cleanup — independent, supervised action

The full-run scratch path is:

`/scratch1/users/ella/nextflow_work/TDP43_LOF_training_full`

It may contain large reference intermediates, BAMs and duplicate task outputs. Removing it will prevent reusing these tasks via `-resume`. **Do not remove work to fix a failing pipeline.** Only after results, checksum records, MultiQC review and follow-on work have been certified, obtain explicit PI approval.

First perform a **read-only** audit:

```bash
WORK=/scratch1/users/ella/nextflow_work/TDP43_LOF_training_full
[[ -d "$WORK" && ! -L "$WORK" ]] || { echo 'STOP missing or symlink workdir'; exit 1; }
[[ "$(readlink -f "$WORK")" == "$WORK" ]] || { echo 'STOP unexpected workdir target'; exit 1; }
du -sh "$WORK"
find "$WORK" -mindepth 1 -maxdepth 1 -type d | wc -l
pgrep -u ella -af '[n]extflow' || true
```

After explicit authorisation, if there are no active jobs and you have accepted loss of Nextflow resume capability, the **following narrowly scoped command removes only the confirmed full-run scratch directory**; it cannot be undone:

```bash
[[ "$(whoami)" == ella ]] || { echo 'STOP: wrong account'; exit 1; }
WORK=/scratch1/users/ella/nextflow_work/TDP43_LOF_training_full
[[ -d "$WORK" && ! -L "$WORK" ]] || { echo 'STOP: unexpected workdir'; exit 1; }
[[ "$(readlink -f "$WORK")" == "$WORK" ]] || { echo 'STOP: unexpected work path'; exit 1; }
pgrep -u ella -af '[n]extflow' && { echo 'STOP: active Nextflow process'; exit 1; }
read -r -p 'After Matt approval, type DELETE_ELLA_FULL_RNASEQ_WORK: ' APPROVAL
[[ "$APPROVAL" == DELETE_ELLA_FULL_RNASEQ_WORK ]] || { echo 'Cancelled'; exit 1; }
rm -rf -- "$WORK"
echo 'Full RNA-seq Nextflow work removed. Resume is no longer available.'
```

Do **not** generalise that path to `/scratch1/users/ella` or delete the project `results/`, `.nextflow` histories, shared container caches or anyone else's data. Avoid cleaning DGE/rMATS scratch work until each has separately finished and been certified.

## I. Final QC and retention decision

Before moving on, create a brief audit note in `docs/` with these answers:

1. **Execution:** Did all Nextflow tasks complete, and what did the execution trace show?
2. **Data products:** Were all 16 canonical BAMs and indexes present? Did the two Salmon matrices contain the identical 16 sample names and gene order?
3. **MultiQC:** Were there any outliers in read depth, Trim Galore retention, mapping, strandedness or duplication? What additional review did you perform?
4. **Retention:** Which outputs must remain for DGE and later rMATS? Which large files are regenerable?
5. **Offload:** Where are the two independently checked authoritative source copies, and what exact validation supports deleting the local FASTQ duplicates?
6. **Cleanup:** What is lost when the full-run Nextflow work folder is removed? Has Matt authorised that loss?

### Training checklist

- [ ] Technical certification script PASS; provenance and gene-count/length SHA256 recorded.
- [ ] Reviewed **actual** MultiQC plots for all 16 libraries; documented evidence and anomalies.
- [ ] Understand the relationship between FastQC, Trim Galore, STAR, Salmon, Picard and rMATS.
- [ ] Can explain why normalised counts are not the same as raw counts and why gene lengths matter for DGE.
- [ ] Located redundant-looking BAMs **without** removing essential canonical BAMs.
- [ ] Audited local FASTQs against MD5 sidecars and independently verified RDS/archive policy.
- [ ] Did **not** remove anything before downstream completion and explicit PI sign-off.

**Reference:** White Lab completed TDP-43 and hnRNP K QC/certification records; [nf-core/rnaseq 3.23.0 output descriptions](https://nf-co.re/rnaseq/3.23.0/docs/output); [MultiQC documentation](https://docs.seqera.io/multiqc/); [nf-core/rnaseq usage](https://nf-co.re/rnaseq/3.23.0/docs/usage).
