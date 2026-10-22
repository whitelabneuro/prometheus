# White Lab | Ella training 4 — TDP-43 differential gene expression and QC

**Audience:** Ella, working in her own Prometheus training project  
**Exercise:** Run **nf-core/differentialabundance 1.5.0 / DESeq2** on all 16 full-depth TDP-43 samples, reproduce the four White Lab contrasts with differentiation-batch adjustment, and inspect the initial differential-expression outputs.  
**Prerequisites:** Training 2 (full nf-core/rnaseq) and Training 3 (successful RNA-seq technical audit plus MultiQC interpretation).  

> **Scientific objective:** Distinguish TDP-43 depletion-associated gene-expression changes from responses to nominal 11J treatment while accounting for independent neuronal differentiation batches. Use **the raw Salmon gene-count matrix together with its matching Salmon gene-length matrix**, not normalised TPMs and not the count matrix alone.

The read-only production methodological reference is `/projects/active/TDP43_LOF`. The analogous successfully reproduced hnRNP K analysis is `/projects/active/HNRNPK_LOF`, but your training analysis must remain under **`/projects/users/ella/TDP43_LOF_training`**. No `sudo` and no editing either completed project.

## A. Before running: understand DESeq2 and the experimental design

nf-core/rnaseq aligns and quantifies reads. nf-core/differentialabundance consumes the published gene-level matrices and fits differential-expression models. DESeq2 estimates how much a gene's abundance differs between conditions while accounting for technical variability and biological replicate variation.

The four experimental groups are:

| Group | Samples | Independent differentiations |
|---|---|---|
| Scramble + DMSO | `TDP-SCR-DMSO-{1..4}` | 4 |
| TDP-43 KD + DMSO | `TDP-KD-DMSO-{1..4}` | 4 |
| Scramble + nominal 11J | `TDP-SCR-11J-{1..4}` | 4 |
| TDP-43 KD + nominal 11J | `TDP-KD-11J-{1..4}` | 4 |

Differentiation batch is `batch_1`, `batch_2`, `batch_3` or `batch_4`, as specified by the numeric sample suffix. The `differentiation_batch` column is a **model blocking variable** in all four DESeq2 contrasts. This helps avoid attributing batch-to-batch changes to knockdown or treatment.

### Contrast direction (very important)

| Contrast ID | Target / numerator | Reference / denominator | Meaning of positive log2FC |
|---|---|---|---|
| `kd_vs_scramble_dmso` | KD DMSO | Scramble DMSO | Higher in KD (DMSO) |
| `kd_vs_scramble_nmdi` | KD 11J | Scramble 11J | Higher in KD (11J) |
| `nmdi_vs_dmso_scramble` | Scramble 11J | Scramble DMSO | Higher in 11J controls |
| `nmdi_vs_dmso_kd` | KD 11J | KD DMSO | Higher in 11J KD |

The `nmdi` string is a **computational naming convention** inherited from the completed White Lab analysis. It represents *nominal* 11J treatment, not independently demonstrated NMD inhibition.

**How to think about statistics:** `log2FoldChange=-1` means roughly half the abundance of reference; `+1` means roughly twice. `pvalue` is the unadjusted evidence against the null; `padj` is multiple-testing-adjusted (BH) evidence. `FDR<0.05` is a statistical threshold, **not** proof of direct TDP-43 binding or a disease mechanism. Low counts, biological heterogeneity and unusual samples also matter.

## B. Find the exact full-depth matrices and check readiness

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
[[ "$(whoami)" == ella ]] || { echo 'STOP: wrong Linux account'; exit 1; }
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
[[ -d "$OUT" ]] || { echo 'STOP: full RNAseq output missing'; exit 1; }
ls "$PROJECT"/docs/full_rnaseq_technical_QC_*.txt
ls "$PROJECT/docs/full_rnaseq_MultiQC_interpretation.md"
COUNTS="$OUT/star_salmon/salmon.merged.gene_counts.tsv"
LENGTHS="$OUT/star_salmon/salmon.merged.gene_lengths.tsv"
ls -lh "$COUNTS" "$LENGTHS"
df -h /projects /scratch1 /scratch2
```

The technical QC and a completed MultiQC interpretation should already be available. **Stop if the full-depth run is not technically certified or sample quality requires investigation.** Never use the pilot's `nfcore_rnaseq_fastp_pilot` data for biological differential expression.

Why two matrices? Salmon counts depend partly on transcript lengths. The gene-length matrix provides sample-specific information used by DESeq2 to account for length bias. This is the validated nf-core workflow and differs from passing scaled TPM as if it were raw counts.

## C. Prepare an isolated DGE workspace and copy the validated design

The completed TDP-43 study already has validated sample metadata, contrasts and DESeq2 settings. For training, we will **copy those small reference files** into your Ella project but redirect the input and matrix paths to your own full-depth RNA-seq result. This is more reliable than inventing new factor labels or changing statistical parameters.

| Item | Ella location |
|---|---|
| New DGE metadata | `metadata/dge_full/` |
| DESeq2 params | `metadata/dge_full/params.yaml` |
| Sample metadata and contrasts | `metadata/dge_full/sample_metadata.csv`, `contrasts.csv` |
| Existing execution config | `nextflow.config` (Ella's Singularity-safe config) |
| DGE launcher | `run_nfcore_differentialabundance_full.sh` |
| Work | `/scratch1/users/ella/nextflow_work/TDP43_LOF_training_dge_full` |
| Runstate | `/scratch2/users/ella/runstate/TDP43_LOF_training/dge_full` |
| Published result | `results/nfcore_differentialabundance_full_out_<timestamp>` |
| Output pointer | `docs/dge_full_last_outdir.txt` |

### C1. Generate source-matched DGE inputs with independent checks

The preparation below refuses to overwrite existing DGE metadata. It confirms that both matrices contain the same 16 samples and that the 16 rows of metadata represent the complete factorial design.

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
mkdir -p "$PROJECT/scripts" "$PROJECT/docs"
SCRIPT="$PROJECT/scripts/prepare_dge_full.py"
[[ ! -e "$SCRIPT" ]] || { echo 'STOP: preparation script already exists'; exit 1; }
cat > "$SCRIPT" <<'PY'
#!/usr/bin/env python3
from pathlib import Path
from collections import Counter
import csv,hashlib,shutil

base=Path('/projects/users/ella/TDP43_LOF_training')
ref=Path('/projects/active/TDP43_LOF/metadata/dge')
meta=base/'metadata/dge_full'
out=Path((base/'docs/full_rnaseq_last_outdir.txt').read_text().strip()).resolve()
if out.parent != (base/'results').resolve() or 'nfcore_rnaseq_full_out_' not in out.name:
    raise SystemExit('STOP: not an Ella full-run RNAseq output')
counts=out/'star_salmon/salmon.merged.gene_counts.tsv'
lengths=out/'star_salmon/salmon.merged.gene_lengths.tsv'
for file in (counts,lengths):
    if not file.is_file(): raise SystemExit('STOP: missing matrix '+str(file))
with counts.open(newline='') as f: h1=next(csv.reader(f,delimiter='\t'))
with lengths.open(newline='') as f: h2=next(csv.reader(f,delimiter='\t'))
expected={f'TDP-{p}-{t}-{i}' for p in ('SCR','KD') for t in ('DMSO','11J') for i in range(1,5)}
if h1!=h2 or h1[:2]!=['gene_id','gene_name'] or len(h1)!=18 or set(h1[2:])!=expected:
    raise SystemExit('FAIL: matrix sample/schema mismatch')
# Also require identical gene order, not merely identical column headers.
with counts.open(newline='') as a, lengths.open(newline='') as b:
    ca=csv.reader(a,delimiter='\t'); cb=csv.reader(b,delimiter='\t')
    next(ca);next(cb)
    n=0
    for x,y in __import__('itertools').zip_longest(ca,cb):
        if x is None or y is None or x[:2]!=y[:2]:
            raise SystemExit('FAIL: gene identity/order mismatch in matrices')
        n+=1
if n<10000: raise SystemExit('FAIL: unexpected gene universe')
print('PASS: counts and gene lengths have identical',n,'gene rows and 16 sample columns')

names=['sample_metadata.csv','contrasts.csv','params.yaml']
for name in names:
    if not (ref/name).is_file(): raise SystemExit('STOP: missing canonical reference '+name)
    if (meta/name).exists(): raise SystemExit('STOP: local DGE file already exists '+name)
with (ref/'sample_metadata.csv').open(newline='') as f:
    sr=list(csv.DictReader(f))
if len(sr)!=16 or {x['sample'] for x in sr}!=expected:
    raise SystemExit('FAIL: reference metadata sample IDs')
conditions=Counter(x['condition'] for x in sr)
if len(conditions)!=4 or set(conditions.values())!={4}:
    raise SystemExit('FAIL: 4x4 condition design')
for row in sr:
    suffix=row['sample'].rsplit('-',1)[1]
    if row['differentiation_batch']!='batch_'+suffix or row['biological_replicate']!=suffix:
        raise SystemExit('FAIL: differentiation batch mismatch '+row['sample'])
    if ('-11J-' in row['sample']) != (row['treatment']=='nominal_11j'):
        raise SystemExit('FAIL: nominal treatment label mismatch')
with (ref/'contrasts.csv').open(newline='') as f:
    cr=list(csv.DictReader(f))
pairs={
 'kd_vs_scramble_dmso':('kolf_scramble_dmso','kolf_tdp43_shrna_dmso'),
 'kd_vs_scramble_nmdi':('kolf_scramble_11j','kolf_tdp43_shrna_11j'),
 'nmdi_vs_dmso_scramble':('kolf_scramble_dmso','kolf_scramble_11j'),
 'nmdi_vs_dmso_kd':('kolf_tdp43_shrna_dmso','kolf_tdp43_shrna_11j')}
if len(cr)!=4 or {x['id'] for x in cr}!=set(pairs): raise SystemExit('FAIL: four contrasts')
for row in cr:
    if (row['reference'],row['target'])!=pairs[row['id']] or row['blocking']!='differentiation_batch':
        raise SystemExit('FAIL: contrast direction/blocking '+row['id'])
print('PASS: four contrasts and 4x4 batch-blocked design')

original=(ref/'params.yaml').read_text()
root='/projects/active/TDP43_LOF/metadata/dge/'
oldrna='/projects/active/TDP43_LOF/results/nfcore_rnaseq_out_20260828_115639/'
if original.count(root)!=2 or original.count(oldrna)!=2 or original.count('study_name: "tdp43_lof_in"')!=1:
    raise SystemExit('STOP: validated reference parameters differ from expected; ask Matt')
updated=original.replace(root,str(meta)+'/').replace(oldrna,str(out)+'/')
updated=updated.replace('study_name: "tdp43_lof_in"','study_name: "tdp43_lof_ella_full_training"')
for item in (str(meta/'sample_metadata.csv'),str(meta/'contrasts.csv'),str(counts),str(lengths)):
    if item not in updated: raise SystemExit('FAIL: adapted path '+item)
for item in ('deseq2_test: "Wald"','deseq2_shrink_lfc: true',
             'differential_max_qval: 0.05','gsea_run: false','gprofiler2_run: false'):
    if item not in updated: raise SystemExit('FAIL: scientific parameter '+item)
meta.mkdir(parents=True,exist_ok=True)
for name in ('sample_metadata.csv','contrasts.csv'): shutil.copy2(ref/name,meta/name)
(meta/'params.yaml').write_text(updated)
print('PASS: created Ella-local source-matched DGE files at',meta)
for path in sorted(meta.iterdir()):
    print(hashlib.sha256(path.read_bytes()).hexdigest(),path)
PY
chmod 700 "$SCRIPT"
python3 -m py_compile "$SCRIPT" || exit 1
python3 "$SCRIPT" | tee "$PROJECT/docs/dge_full_preparation_audit.txt"
```

**PASS:** 16 matched input sample columns, identical gene ordering, four groups with four biological replicates, correct contrast targets and batch-blocking, and source-matched params preserving DESeq2 method. The training output differs from production in *file paths and study identifier*, not the scientific tests.

The reference `prometheus.config` in `/projects/active/TDP43_LOF/metadata/dge` uses a shared cache location that you must not write into. Instead, **use your pre-existing `nextflow.config`** with its private writable Singularity cache. This is an infrastructure adaptation, not a change to DESeq2 statistics.

## D. Create a dedicated DESeq2 run launcher

The launcher reuses the central Nextflow 25.10.4 installation, saves output/runstate independently of the pilot and RNA-seq, and can resume an interrupted run. **It must run only after `prepare_dge_full.py` passes.**

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
LAUNCHER="$PROJECT/run_nfcore_differentialabundance_full.sh"
[[ ! -e "$LAUNCHER" ]] || { echo 'STOP: DGE launcher already exists'; exit 1; }
cat > "$LAUNCHER" <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
umask 077
PROJECT=/projects/users/ella/TDP43_LOF_training
META="$PROJECT/metadata/dge_full"
WORKDIR=/scratch1/users/ella/nextflow_work/TDP43_LOF_training_dge_full
RUNSTATE=/scratch2/users/ella/runstate/TDP43_LOF_training/dge_full
POINTER="$PROJECT/docs/dge_full_last_outdir.txt"
PARAMS="$META/params.yaml"
CONFIG="$PROJECT/nextflow.config"

export NXF_VER=25.10.4
export NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
export NXF_SINGULARITY_LIBRARYDIR=/scratch2/container_cache/singularity
export NXF_SINGULARITY_CACHEDIR=/scratch2/users/ella/container_cache/singularity
export SINGULARITY_CACHEDIR=/scratch2/users/ella/container_cache/runtime
export SINGULARITY_TMPDIR=/scratch2/users/ella/tmp
export TMPDIR=/scratch2/users/ella/tmp
export NXF_ANSI_LOG=false

mkdir -p "$WORKDIR" "$RUNSTATE" "$PROJECT/results" "$PROJECT/logs" \
  "$PROJECT/docs" "$NXF_HOME" "$NXF_SINGULARITY_CACHEDIR" \
  "$SINGULARITY_CACHEDIR" "$TMPDIR"
for f in "$PARAMS" "$CONFIG" "$META/sample_metadata.csv" "$META/contrasts.csv"; do
  [[ -r "$f" ]] || { echo "STOP: DGE input missing: $f"; exit 1; }
done

resume=()
if [[ "${RESUME:-0}" == 1 ]]; then
  [[ -s "$POINTER" ]] || { echo 'STOP: no DGE resume pointer'; exit 1; }
  OUTDIR=$(cat "$POINTER")
  [[ -d "$OUTDIR" ]] || { echo 'STOP: old DGE output missing'; exit 1; }
  resume=(-resume)
else
  [[ ! -e "$POINTER" ]] || { echo 'STOP: previous DGE run recorded; resume instead'; exit 1; }
  OUTDIR="$PROJECT/results/nfcore_differentialabundance_full_out_$(date +%Y%m%d_%H%M%S)"
  mkdir "$OUTDIR"
  printf '%s\n' "$OUTDIR" > "$POINTER"
fi
LOGFILE="$PROJECT/logs/$(basename "$OUTDIR").nextflow.log"
printf 'date=%s\nout=%s\nparams=%s\n' "$(date -Is)" "$OUTDIR" "$PARAMS" \
  >> "$PROJECT/docs/dge_full_run_history.txt"

cd "$RUNSTATE"
/usr/local/bin/nextflow run nf-core/differentialabundance \
  -r 1.5.0 \
  -profile singularity \
  -c "$CONFIG" \
  -work-dir "$WORKDIR" \
  -params-file "$PARAMS" \
  "${resume[@]}" \
  --outdir "$OUTDIR" \
  2>&1 | tee -a "$LOGFILE"
BASH
chmod 700 "$LAUNCHER"
bash -n "$LAUNCHER" && echo 'PASS: DESeq2 launcher syntax'
```

**Important:** `-c` supplies compute/Singularity settings. `-params-file` supplies the DGE experiment, counts, lengths and statistical settings. They are different Nextflow mechanisms.

## E. Preflight and launch within tmux

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
export NXF_VER=25.10.4 NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
/usr/local/bin/nextflow -version
bash -n "$PROJECT/run_nfcore_differentialabundance_full.sh"
cat "$PROJECT/metadata/dge_full/contrasts.csv"
grep -E '^(study_name|input|contrasts|matrix|transcript_length_matrix|gtf|deseq2_test|deseq2_shrink_lfc|gsea_run|gprofiler2_run):' \
  "$PROJECT/metadata/dge_full/params.yaml"
df -h /projects /scratch1 /scratch2
pgrep -u ella -af '[n]extflow' || true
```

Check the four contrast directions before committing the run. If a previous run is active, do not launch a second concurrent copy.

Start a new tmux session:

```bash
tmux new -s ella_tdp43_full_dge
```

Inside tmux:

```bash
cd /projects/users/ella/TDP43_LOF_training || exit 1
./run_nfcore_differentialabundance_full.sh
```

Detach using **Ctrl+B, D**; reconnect with `tmux attach -t ella_tdp43_full_dge`.

To monitor from a separate terminal:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/dge_full_last_outdir.txt")
echo "DGE output: $OUT"
tail -n 35 "$PROJECT/logs/$(basename "$OUT").nextflow.log"
tmux ls
```

The validated production TDP-43 and new HNRNPK DGE analyses each completed **21 tasks** within a few minutes. Your runtime may differ. The stages should include input validation, matrix filtering, four DESeq2 comparisons, differential plots, exploratory plots and a report. **Do not delete any work if an error appears.** To resume an interrupted run after confirming the previous process is no longer active:

```bash
cd /projects/users/ella/TDP43_LOF_training || exit 1
RESUME=1 ./run_nfcore_differentialabundance_full.sh
```

## F. Locate and interpret the published outputs

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/dge_full_last_outdir.txt")
find "$OUT" -mindepth 1 -maxdepth 2 -type d | sort
find "$OUT/tables/differential" -maxdepth 1 -type f | sort
find "$OUT/plots" -type f -name '*.png' | sort | head -n 35
ls -lh "$OUT/tables/processed_abundance/"*.tsv
find "$OUT/report" -type f -name '*.html'
```

Expected central outputs:

| Output | How to interpret |
|---|---|
| `tables/differential/kd_vs_scramble_dmso.deseq2.results.tsv` | **Full** KD-minus-scramble DMSO gene-test universe |
| `tables/differential/kd_vs_scramble_nmdi.deseq2.results.tsv` | **Full** KD-minus-scramble nominal 11J universe |
| `tables/differential/nmdi_vs_dmso_scramble.deseq2.results.tsv` | Treatment response in controls |
| `tables/differential/nmdi_vs_dmso_kd.deseq2.results.tsv` | Treatment response in TDP-43 KD |
| `*.deseq2.results_filtered.tsv` | Pipeline-filtered genes after FDR and fold-change criteria |
| `tables/processed_abundance/all.normalised_counts.tsv` | Expression normalised for library/composition differences; **not** raw-count input for this completed DGE run |
| `tables/processed_abundance/all.vst.tsv` | Variance-stabilised expression for plots/correlations |
| `plots/exploratory/condition/png/pca2d.png` | Principal-component analysis: global sample structure/outliers |
| `plots/differential/*/png/volcano.png` | Effect size versus statistical significance |
| `tables/annotation/gencode.anno.tsv` | Reference annotation data |
| `pipeline_info/` | Reproducibility, exact task trace and software versions |

The complete DESeq2 tables commonly report `gene_id`, `baseMean`, `log2FoldChange`, `lfcSE`, `pvalue` and `padj`. They may **omit the `gene_name` column**, as seen in both production TDP-43 and HNRNPK analyses. This does not invalidate the statistics: recover gene names by exact gene-ID matching to the Salmon count matrix, making a *separate annotated copy*, never rewriting the nf-core output.

### F1. First-pass certification and DESeq2 numerical QC

This script checks trace completion, four result tables, FDR/fold-change filtering, target-gene depletion and sample-level VST correlation. It saves a permanent text report; it **does not** alter the published results or silently call the study biologically validated.

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
SCRIPT="$PROJECT/scripts/audit_full_dge.py"
[[ ! -e "$SCRIPT" ]] || { echo 'STOP: DGE audit script exists'; exit 1; }
cat > "$SCRIPT" <<'PY'
#!/usr/bin/env python3
from pathlib import Path
from collections import Counter
from datetime import datetime
import csv,math,statistics

base=Path('/projects/users/ella/TDP43_LOF_training')
out=Path((base/'docs/dge_full_last_outdir.txt').read_text().strip()).resolve()
assert out.parent==(base/'results').resolve() and out.name.startswith('nfcore_differentialabundance_full_out_')
log=base/'logs'/(out.name+'.nextflow.log')
text=log.read_text(errors='replace')
if 'Succeeded' not in text or 'Completed at:' not in text:
    raise SystemExit('FAIL: Nextflow did not publish completion summary')
traces=list((out/'pipeline_info').glob('execution_trace_*.txt'))
if len(traces)!=1: raise SystemExit('FAIL: execution trace missing/ambiguous')
with traces[0].open(newline='') as f: tasks=list(csv.DictReader(f,delimiter='\t'))
statuses=Counter((r.get('status') or '').strip().upper() for r in tasks)
if not tasks or set(statuses)!={'COMPLETED'}: raise SystemExit('FAIL: '+str(statuses))

names={}
counts=Path((base/'docs/full_rnaseq_last_outdir.txt').read_text().strip())/'star_salmon/salmon.merged.gene_counts.tsv'
with counts.open(newline='') as f:
    for r in csv.DictReader(f,delimiter='\t'):
        if r['gene_id'] in names: raise SystemExit('FAIL: duplicate gene ID')
        names[r['gene_id']]=r['gene_name']
target=[k for k,v in names.items() if v.upper()=='TARDBP']
if len(target)!=1: raise SystemExit('STOP: TARDBP gene annotation needs review')

def num(x):
    try:
        x=float(x)
        return x if math.isfinite(x) else None
    except (TypeError,ValueError): return None

comparisons=('kd_vs_scramble_dmso','kd_vs_scramble_nmdi',
             'nmdi_vs_dmso_scramble','nmdi_vs_dmso_kd')
output=['DGE INITIAL QC', 'Date: '+datetime.now().astimezone().isoformat(),
        'Result directory: '+str(out),'Trace tasks: '+str(len(tasks))+' '+str(dict(statuses))]
for contrast in comparisons:
    path=out/'tables/differential'/f'{contrast}.deseq2.results.tsv'
    filtered=out/'tables/differential'/f'{contrast}.deseq2.results_filtered.tsv'
    with path.open(newline='') as f:
        reader=csv.DictReader(f,delimiter='\t'); cols=reader.fieldnames; rows=list(reader)
    with filtered.open(newline='') as f: fr=list(csv.DictReader(f,delimiter='\t'))
    required={'gene_id','baseMean','log2FoldChange','lfcSE','pvalue','padj'}
    if not required.issubset(cols or []): raise SystemExit('FAIL: DESeq2 schema '+contrast)
    ids=[r['gene_id'] for r in rows]
    if len(ids)!=len(set(ids)) or not all(x in names for x in ids):
        raise SystemExit('FAIL: duplicated/unmapped gene IDs '+contrast)
    byid={r['gene_id']:r for r in rows}
    sig=[r for r in rows if num(r['padj']) is not None and num(r['padj'])<0.05
         and num(r['log2FoldChange']) is not None]
    twofold={r['gene_id'] for r in sig if abs(float(r['log2FoldChange']))>=1}
    actual={r['gene_id'] for r in fr}
    if twofold!=actual or len(actual)!=len(fr):
        raise SystemExit('FAIL: pipeline filtering differs from FDR05 / twofold '+contrast)
    h=byid.get(target[0])
    if h is None: raise SystemExit('FAIL: TARDBP missing '+contrast)
    line=(f'{contrast}: full={len(rows)} FDR05={len(sig)} '
          f'FDR05+twofold={len(twofold)} TARDBP_log2FC={h["log2FoldChange"]} '
          f'TARDBP_padj={h["padj"]}')
    print(line);output.append(line)

vst=out/'tables/processed_abundance/all.vst.tsv'
with vst.open(newline='') as f:
    reader=csv.DictReader(f,delimiter='\t'); header=reader.fieldnames or []
    observed={f'TDP-{p}-{t}-{i}' for p in ('SCR','KD') for t in ('DMSO','11J') for i in range(1,5)}
    if not observed.issubset(header): raise SystemExit('FAIL: VST samples missing')
    # No need to load the large full matrix to certify its schema here.
    n=sum(1 for _ in reader)
if n<10000: raise SystemExit('FAIL: unexpected VST gene count')
output.append('VST matrix: 16 samples, '+str(n)+' rows')
pca=out/'plots/exploratory/condition/png/pca2d.png'
if not pca.is_file(): raise SystemExit('FAIL: exploratory PCA missing')
output.append('PCA exists: '+str(pca))
output.append('STATUS: TECHNICAL INITIAL QC PASS; inspect PCA/volcano/replicates with Matt')
report=base/'docs'/('dge_full_initial_QC_'+datetime.now().strftime('%Y%m%d_%H%M%S')+'.txt')
report.write_text('\n'.join(output)+'\n')
print('\n'.join(output[-3:])); print('Saved:',report)
PY
chmod 700 "$SCRIPT"
python3 -m py_compile "$SCRIPT" || exit 1
python3 "$SCRIPT"
```

**How to interpret the expected TARDBP result:** The original validated production TDP-43 run found approximately **log2FC −1.770 in DMSO**, and **−1.916 under nominal 11J**, corresponding to roughly 71% and 73.5% reductions in TARDBP RNA abundance. Your independent full-depth run *may* reproduce those results closely if inputs and settings match, but do not impose these numbers as a rigid PASS criterion. Investigate large differences in sample identity, library design, strandedness, pipeline versions, missing covariates or count/length inputs. This is an RNA-level estimate, not protein validation.

The production study recorded approximately 16,209 (DMSO) and 16,254 (nominal 11J) genes at FDR < 0.05 in the knockdown comparisons; the additional FDR+twofold counts were 6,491 and 6,492. The historical numbers are **contextual comparators**, not targets to tune toward or substitute for observed output.

### F2. Plot interpretation — PCA, dispersion and volcano plots

**PCA:** Each point is a biological sample. Ask whether samples cluster by knockdown/treatment, whether a differentiation batch appears influential, and whether a sample sits unexpectedly far from its group. PCA is descriptive and cannot substitute for statistical adjustment for batch.

**Dispersion plot:** DESeq2 models variability between biological replicates as a function of expression abundance. A severe departure from the fitted trend or one unusual sample deserves investigation.

**Volcano plot:** Horizontal axis is estimated log2FC, vertical is statistical significance. A dramatic point at low RNA abundance can be less persuasive than a consistently expressed, replicate-supported result. Significance counts depend on the testable universe and thresholds.

**Full versus filtered:** The full table retains insignificant and `NA`-adjusted-P rows, which are necessary for rigorous downstream comparison. The pipeline-filtered table is expected to represent FDR<0.05 and |log2FC|≥1 with the inherited configuration. Never analyse only a filtered list if the question requires comparing overlapping testable gene universes.

**Sample-level expression:** `all.normalised_counts.tsv` and `all.vst.tsv` support descriptive replicate/QC analyses. Check TARDBP values in all 16 samples, not just its final differential fold change. Large sample outliers require biological evaluation before any exclusion decision.

## G. Initial conclusions

Create `docs/dge_full_biological_review.md` containing the following *observed* information:

```markdown
# Ella full TDP-43 differential expression — review

- Source full RNA-seq result:
- Count/length input SHA256:
- DGE output and execution status:
- All four contrast IDs, target and reference directions:
- Differentiation-batch design check:
- TARDBP log2FC and FDR in DMSO:
- TARDBP log2FC and FDR in nominal 11J:
- Four-group normalised TARDBP expression values:
- Number of testable genes and FDR-significant genes by contrast:
- Number of genes meeting FDR < 0.05 and |log2FC| >= 1:
- PCA observation / sample outliers:
- How does 11J affect scramble and KD samples?
- Deviations from original production results, if any:
- QC verdict: PASS / INVESTIGATE / FAIL, with rationale:
- Reviewed with Matt (date):
```

**Interpretive caution:** A significant 11J treatment effect is not itself proof of successful inhibition of nonsense-mediated RNA decay. DGE does not establish direct TDP-43 RNA binding or identify cryptic exons. Both require separate analytical and biological evidence.

### Training checklist

- [ ] I used **the full 16-sample** RNA-seq matrices, not the fastp pilot.
- [ ] I verified identical count/length genes and sample columns.
- [ ] I copied and verified the canonical four-contrast, batch-blocked design.
- [ ] I understand the sign of each DESeq2 contrast.
- [ ] I used scientific `params.yaml` plus the Ella safe local compute `nextflow.config`.
- [ ] I launched in tmux and can resume with retained work/runstate.
- [ ] All four full and filtered DESeq2 tables, normalised counts, VST and PCA exist.
- [ ] I extracted TARDBP depletion rather than assuming it from the shRNA label.
- [ ] I can explain adjusted P-values, log2FC, the filtering rules and why `NA` values occur.
- [ ] I reviewed biological QC with Matt before proposing follow-up or cleanup.

**Next task:** After review, extend to rMATS splicing or mechanistic integration. That requires a **separate** validated launch procedure. Preserve the 16 canonical mark-duplicate BAMs and their indices.

**References:** Read-only `/projects/active/TDP43_LOF/README.md` and `metadata/dge/`; [nf-core/differentialabundance 1.5.0 usage](https://nf-co.re/differentialabundance/1.5.0/docs/usage); [nf-core/differentialabundance 1.5.0 parameters](https://nf-co.re/differentialabundance/1.5.0/parameters); [nf-core/rnaseq 3.23.0 outputs](https://nf-co.re/rnaseq/3.23.0/docs/output).
