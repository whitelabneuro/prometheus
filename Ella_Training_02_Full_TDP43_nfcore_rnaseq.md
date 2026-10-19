# White Lab | Ella training 2 — Full-depth TDP-43 nf-core/rnaseq

**Audience:** Ella, working on Prometheus as Linux user `ella`  
**Exercise:** Repeat the validated White Lab full-depth TDP-43 workflow independently using all 16 biological samples, **Trim Galore**, STAR + Salmon and GRCh38/GENCODE v49.  
**Prerequisite:** Complete the two-sample `fastp` pilot training (`Ella_Prometheus_nfcore_rnaseq_training.md`).  
**Status:** Training instructions; **not** evidence that Ella's full run has started or finished.  
**Next exercise:** Training 3, evaluating outputs and safe storage management.

> **Safety rule:** Everything created or altered must stay under Ella's own `/projects/users/ella` and `/scratch*/users/ella` directories. `/projects/active/TDP43_LOF`, `/archive` and RDS sequencing originals are **read-only** reference sources. No `sudo`. A full human RNA-seq run is large: agree server load and space with Matt before launching.

## A. What is changing from the pilot, and why?

| Component | Fastp pilot | This production-depth exercise |
|---|---|---|
| Biological libraries | 2 (batch 1, DMSO) | **16: KD/SCR × DMSO/11J × four batches** |
| Paired FASTQs | 4 | **32** |
| FASTQ sampling | First 5,000 reads per library | **No subsampling** |
| Trimming | `fastp` | **`trimgalore` (nf-core default; specified explicitly)** |
| FASTQ FastQC | Skipped to keep pilot short | **Enabled (default)** |
| FASTQ linting | Skipped | **Enabled (default)** |
| DESeq2 exploratory QC | Skipped | **Enabled (default)** |
| Minimum trimmed reads | Pilot override of 100 | **Standard nf-core default (10,000)** |
| Align/quantify | STAR + Salmon | **STAR + Salmon** |
| References | GRCh38 / GENCODE v49 | **Same** |
| Software | nf-core/rnaseq 3.23.0, Nextflow 25.10.4 | **Same, pinned** |
| Published outputs | `results/nfcore_rnaseq_fastp_pilot` | **New, separately named full-depth output** |

The **reference full-run result** from the completed study is `/projects/active/TDP43_LOF/results/nfcore_rnaseq_out_20260828_115639`. Its recorded processing used Trim Galore, STAR+Salmon and GENCODE v49; it completed 606 tasks. This is a *methodological comparator*, not a folder to write into or copy into Ella's results. Exact Nextflow task counts are not a universal success criterion: the final trace must instead show every scheduled task completed.

**Learning goal:** Be able to describe why full-depth data are suitable for expression/splicing analysis whereas the tiny `fastp` pilot is useful only to test the workflow.

## B. Experimental design and directories

The samples have the naming pattern `TDP-{SCR,KD}-{DMSO,11J}-{1,2,3,4}`. Suffixes 1–4 denote **four independently differentiated neuronal batches**, with all four experimental conditions represented in each batch. These are not paired before/after observations. The 11J samples represent a **nominal NMD-inhibitor treatment**; do not claim functional inhibition without evidence.

Use the **existing training project** without overwriting the pilot's launcher, samplesheet, results or runstate:

| Type | Path |
|---|---|
| Persistent project | `/projects/users/ella/TDP43_LOF_training` |
| Original FASTQ copies | `/projects/users/ella/TDP43_LOF_training/inputs/raw` |
| New full samplesheet | `metadata/samplesheet_full.csv` |
| Existing verified local config | `nextflow.config` |
| New full launcher | `run_nfcore_rnaseq_full.sh` |
| New full heavy work | `/scratch1/users/ella/nextflow_work/TDP43_LOF_training_full` |
| New full runstate | `/scratch2/users/ella/runstate/TDP43_LOF_training/rnaseq_full` |
| Private Nextflow cache | `/scratch2/users/ella/nextflow_home` |
| Published results | a unique `results/nfcore_rnaseq_full_out_<timestamp>` |
| Output pointer | `docs/full_rnaseq_last_outdir.txt` |
| Source data | `/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw` |

### B1. Enter Prometheus and check permissions

From a KCL-connected Mac terminal: `ssh ella@10.131.34.252`. When the shell shows `ella@prometheus`, paste:

```bash
whoami
id
PROJECT=/projects/users/ella/TDP43_LOF_training
RDS=/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw
[[ "$(whoami)" == ella ]] || { echo 'STOP: not logged in as ella'; exit 1; }
[[ -d "$PROJECT" && -f "$PROJECT/nextflow.config" ]] || { echo 'STOP: complete pilot first'; exit 1; }
ls -ld "$PROJECT" "$PROJECT/inputs/raw" "$RDS"
df -h /projects /scratch1 /scratch2
free -h
```

**PASS:** correct Unix identity, pilot project exists, source RDS directory readable. **STOP:** missing source, account/group error, unexplained storage shortage or ongoing large competing server jobs. Arrange the full-data compute exercise with Matt first. The original TDP-43 study consumed substantial storage; allow ample free scratch capacity (typically **>1.5 TB** for this full-depth exercise) and more than the FASTQ size plus expected BAM outputs on `/projects`.

### B2. Prepare separate full-run locations

```bash
umask 077
PROJECT=/projects/users/ella/TDP43_LOF_training
mkdir -p "$PROJECT"/{scripts,metadata,results,logs,docs,inputs/raw}
mkdir -p /scratch1/users/ella/nextflow_work/TDP43_LOF_training_full
mkdir -p /scratch2/users/ella/runstate/TDP43_LOF_training/rnaseq_full
mkdir -p /scratch2/users/ella/{nextflow_home,container_cache/singularity,container_cache/runtime,tmp}
```

These directories are all Ella-owned. Never make a broad `chmod`, and never write into `/projects/active/TDP43_LOF`.

## C. Import all 16 sample pairs from RDS

### C1. Validate the source inventory *before* copying

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
RDS=/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw
python3 - <<'PY'
from pathlib import Path
rds = Path('/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw')
expected = [f'TDP-{p}-{t}-{i}_R{mate}_001.fastq.gz'
            for p in ('SCR','KD') for t in ('DMSO','11J')
            for i in range(1,5) for mate in (1,2)]
missing = [x for x in expected if not (rds/x).is_file() or not (rds/(x+'.md5')).is_file()]
print('Expected FASTQs:',len(expected),'Expected sidecars:',len(expected))
print('Source FASTQ volume (GB):',round(sum((rds/x).stat().st_size for x in expected if (rds/x).exists())/1e9,1))
if missing: raise SystemExit('STOP: missing FASTQ or MD5 sidecar: '+', '.join(missing))
print('PASS: all 32 original paired FASTQs and their MD5 sidecars visible')
PY
```

The original study contains 16 biological samples × 2 read directions = **32 FASTQ files**, **not** 32 biological samples. Read directions R1 and R2 must always refer to the same sample.

### C2. Copy into *your* raw folder (safe to repeat after interrupted transfer)

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
RDS=/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw
set -euo pipefail
for p in SCR KD; do
  for t in DMSO 11J; do
    for i in 1 2 3 4; do
      sample="TDP-${p}-${t}-${i}"
      for mate in 1 2; do
        file="${sample}_R${mate}_001.fastq.gz"
        rsync -av --no-perms --no-owner --no-group --partial --info=progress2 \
          "$RDS/$file" "$PROJECT/inputs/raw/"
        rsync -av --no-perms --no-owner --no-group --partial \
          "$RDS/$file.md5" "$PROJECT/inputs/raw/"
      done
    done
  done
done
```

This reads RDS originals, never changes them, and does not touch the previously completed research project. Full-depth copying can be substantial; network throughput and quotas vary. If it is interrupted, repeat the command and verify the MD5 checks *after* copying finishes.

### C3. Verify all 32 copies against supplier hashes

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
cd "$PROJECT/inputs/raw" || exit 1
set -euo pipefail
n=0
for file in TDP-{SCR,KD}-{DMSO,11J}-{1,2,3,4}_R{1,2}_001.fastq.gz; do
  [[ -s "$file" && -s "$file.md5" ]] || { echo "FAIL missing: $file"; exit 1; }
  expected=$(grep -Eom1 '[0-9A-Fa-f]{32}' "$file.md5" | tr 'A-F' 'a-f')
  actual=$(md5sum "$file" | awk '{print $1}')
  [[ "$expected" == "$actual" ]] || { echo "FAIL MD5: $file"; exit 1; }
  echo "PASS MD5: $file"
  n=$((n+1))
done
[[ "$n" == 32 ]] || { echo "STOP expected 32, found $n"; exit 1; }
echo 'PASS: all 32 full-depth FASTQs match supplier MD5'
```

**Stop on any failure.** Do not change the MD5 sidecar to make a mismatch disappear. MD5 tests transfer integrity; it does not show a FASTQ is biologically suitable.

## D. Generate and check a 16-sample full-depth samplesheet

We will programmatically generate the CSV so none of the 32 paths needs manual typing. This creates a **different** samplesheet from the pilot.

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
python3 - <<'PY'
from pathlib import Path
import csv
project=Path('/projects/users/ella/TDP43_LOF_training')
path=project/'metadata/samplesheet_full.csv'
if path.exists(): raise SystemExit('STOP: full samplesheet already exists; inspect rather than overwrite')
rows=[]
for p in ('SCR','KD'):
    for treatment in ('DMSO','11J'):
        for batch in range(1,5):
            sample=f'TDP-{p}-{treatment}-{batch}'
            left=project/'inputs/raw'/f'{sample}_R1_001.fastq.gz'
            right=project/'inputs/raw'/f'{sample}_R2_001.fastq.gz'
            if not left.is_file() or not right.is_file(): raise SystemExit(f'Missing reads: {sample}')
            rows.append([sample,str(left),str(right),'auto'])
with path.open('x',newline='') as fh:
    writer=csv.writer(fh,lineterminator='\n')
    writer.writerow(['sample','fastq_1','fastq_2','strandedness'])
    writer.writerows(rows)
print('PASS: generated',path,'with',len(rows),'unique biological samples')
PY
head -n 5 "$PROJECT/metadata/samplesheet_full.csv"
wc -l "$PROJECT/metadata/samplesheet_full.csv"
```

**PASS:** 17 lines including header; every sample appears once; 4 scramble + 4 KD under each of DMSO and 11J. `auto` asks Salmon to infer strandedness. We will inspect its agreement with RSeQC later in MultiQC.

## E. Preserve the pilot config; create a full-depth launcher

Your existing `nextflow.config` already handles local execution, `cache='lenient'`, retry behaviour, Singularity's shared **read-only** library and your **private writable** container cache. Reuse that config **without** changing it for this exercise.

The launcher below uses the **same scientific workflow as the production full-depth analysis** with isolated training output/work/runstate. It does *not* carry over the pilot's fastp or skip switches. A runstate output pointer ensures `-resume` targets the original run rather than silently creating a new results directory.

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
LAUNCHER="$PROJECT/run_nfcore_rnaseq_full.sh"
[[ ! -e "$LAUNCHER" ]] || { echo 'STOP: full launcher already exists'; exit 1; }
cat > "$LAUNCHER" <<'BASH'
#!/usr/bin/env bash
set -euo pipefail
umask 077
PROJECT=/projects/users/ella/TDP43_LOF_training
WORKDIR=/scratch1/users/ella/nextflow_work/TDP43_LOF_training_full
RUNSTATE=/scratch2/users/ella/runstate/TDP43_LOF_training/rnaseq_full
POINTER="$PROJECT/docs/full_rnaseq_last_outdir.txt"
SAMPLESHEET="$PROJECT/metadata/samplesheet_full.csv"
CONFIG="$PROJECT/nextflow.config"
FASTA=/projects/refdata/Homo_sapiens/GRCh38/Gencode/GRCh38.primary_assembly.genome.fa.gz
GTF=/projects/refdata/Homo_sapiens/GRCh38/Gencode/gencode.v49.primary_assembly.annotation.gtf.gz

export NXF_VER=25.10.4
export NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
export NXF_SINGULARITY_LIBRARYDIR=/scratch2/container_cache/singularity
export NXF_SINGULARITY_CACHEDIR=/scratch2/users/ella/container_cache/singularity
export SINGULARITY_CACHEDIR=/scratch2/users/ella/container_cache/runtime
export SINGULARITY_TMPDIR=/scratch2/users/ella/tmp
export TMPDIR=/scratch2/users/ella/tmp
export NXF_ANSI_LOG=false

mkdir -p "$WORKDIR" "$RUNSTATE" "$PROJECT/logs" "$PROJECT/docs" \
  "$PROJECT/results" "$NXF_HOME" "$NXF_SINGULARITY_CACHEDIR" \
  "$SINGULARITY_CACHEDIR" "$TMPDIR"
for file in "$SAMPLESHEET" "$CONFIG" "$FASTA" "$GTF"; do
  [[ -r "$file" ]] || { echo "STOP unreadable input: $file"; exit 1; }
done

resume=()
if [[ "${RESUME:-0}" == 1 ]]; then
  [[ -s "$POINTER" ]] || { echo 'STOP: no prior full-run output pointer'; exit 1; }
  OUTDIR=$(cat "$POINTER")
  [[ -d "$OUTDIR" ]] || { echo 'STOP: previous output missing'; exit 1; }
  resume=(-resume)
else
  [[ ! -e "$POINTER" ]] || { echo 'STOP: prior full run exists; use RESUME=1'; exit 1; }
  OUTDIR="$PROJECT/results/nfcore_rnaseq_full_out_$(date +%Y%m%d_%H%M%S)"
  mkdir "$OUTDIR"
  printf '%s\n' "$OUTDIR" > "$POINTER"
fi
LOGFILE="$PROJECT/logs/$(basename "$OUTDIR").nextflow.log"
printf 'launch_time=%s\noutput=%s\nwork=%s\nrunstate=%s\n' \
  "$(date -Is)" "$OUTDIR" "$WORKDIR" "$RUNSTATE" \
  >> "$PROJECT/docs/full_rnaseq_run_history.txt"

cd "$RUNSTATE"
/usr/local/bin/nextflow run nf-core/rnaseq \
  -r 3.23.0 \
  -profile singularity \
  -c "$CONFIG" \
  -work-dir "$WORKDIR" \
  "${resume[@]}" \
  --input "$SAMPLESHEET" \
  --outdir "$OUTDIR" \
  --aligner star_salmon \
  --trimmer trimgalore \
  --fasta "$FASTA" \
  --gtf "$GTF" \
  --gencode \
  2>&1 | tee -a "$LOGFILE"
BASH
chmod 700 "$LAUNCHER"
bash -n "$LAUNCHER" && echo 'PASS: launcher syntax'
```

**Scientific settings:** `trimgalore` is nf-core/rnaseq 3.23.0's default, explicitly selected here to avoid confusion. We do **not** specify `--extra_fastp_args`, `--skip_fastqc`, `--skip_linting`, `--skip_deseq2_qc`, `--min_trimmed_reads 100` or `--save_reference`. The latter preserves the validated production choice not to publish unnecessary large reference bundles.

### E1. Independently verify pilot-only options were removed

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
python3 - <<'PY'
from pathlib import Path
s=Path('/projects/users/ella/TDP43_LOF_training/run_nfcore_rnaseq_full.sh').read_text()
required=['-r 3.23.0','--aligner star_salmon','--trimmer trimgalore',
          '--input "$SAMPLESHEET"','--gencode','-profile singularity']
for x in required:
    assert x in s, f'Missing full-depth option: {x}'
for x in ['--reads_to_process','--extra_fastp_args','--skip_fastqc',
          '--skip_linting','--skip_deseq2_qc','--min_trimmed_reads 100']:
    assert x not in s, f'Pilot-only option remains: {x}'
print('PASS: full-depth Trim Galore command; pilot-only shortcuts absent')
PY
```

## F. Preflight before launching

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
export NXF_VER=25.10.4 NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
/usr/local/bin/nextflow -version
singularity --version
bash -n "$PROJECT/run_nfcore_rnaseq_full.sh"
df -h /projects /scratch1 /scratch2
free -h
# Check there is no other active Nextflow run of yours before starting.
pgrep -u ella -af '[n]extflow' || true
# Check completed Fastq copying and MD5 before continuing.
ls "$PROJECT/inputs/raw"/*.fastq.gz | wc -l
```

**PASS requires:** 32 MD5 checks, exactly 16 metadata rows, 25.10.4, valid config/launcher, and explicit supervisor agreement on storage/load. If the Nextflow pipeline cache is not accessible from Ella's account, do not copy or modify the established `/projects/active` study; ask Matt to provide approved access.

## G. Run within tmux and monitor

Create a **new** session (the previous pilot tmux session is not the full-depth session):

```bash
tmux new -s ella_tdp43_full_rnaseq
```

Within this session:

```bash
cd /projects/users/ella/TDP43_LOF_training || exit 1
./run_nfcore_rnaseq_full.sh
```

When tasks begin, detach with **Ctrl+B, D**. Reconnect using `tmux attach -t ella_tdp43_full_rnaseq`. The tmux session is not the pipeline itself: Nextflow's final summary and trace establish completion.

Read-only progress checks from another SSH shell:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
echo "Result directory: $OUT"
tail -n 35 "$PROJECT/logs/$(basename "$OUT").nextflow.log"
du -sh /scratch1/users/ella/nextflow_work/TDP43_LOF_training_full "$OUT" 2>/dev/null || true
df -h /projects /scratch1 /scratch2
tmux ls
```

Look for input checks, FastQC, **Trim Galore**, STAR, Salmon, Picard/mark-duplicate, MultiQC and completed processes. A temporary `0 of n` state means tasks are queued, not failed. If you see `FAILED` or `ABORTED`, retain the logs and work directory.

### Resuming a failed or interrupted analysis

After confirming the previous Nextflow process is no longer active and the work directory still exists, attach to the tmux session and run:

```bash
cd /projects/users/ella/TDP43_LOF_training || exit 1
RESUME=1 ./run_nfcore_rnaseq_full.sh
```

`-resume` reuses completed work *only while the original Nextflow history and work files are available*. Never run two copies concurrently. The output pointer ensures the same published results directory is reused.

## H. Completion handoff and reproducibility record

After Nextflow finishes, save the important identifiers for the next exercise:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
OUT=$(cat "$PROJECT/docs/full_rnaseq_last_outdir.txt")
{
  date -Is
  echo "OUTDIR=$OUT"
  /usr/local/bin/nextflow -version
  echo '===== FINAL EXECUTION LOG ====='
  tail -n 40 "$PROJECT/logs/$(basename "$OUT").nextflow.log"
  echo '===== PUBLISHED OUTPUT STRUCTURE ====='
  find "$OUT" -mindepth 1 -maxdepth 2 -type d | sort
  echo '===== KEY DGE MATRICES ====='
  ls -lh "$OUT/star_salmon/salmon.merged.gene_counts.tsv" \
         "$OUT/star_salmon/salmon.merged.gene_lengths.tsv"
} | tee "$PROJECT/docs/full_rnaseq_initial_completion.txt"
```

**Do not delete FASTQs, BAMs or Nextflow work yet.** Proceed to Training 3 for technical certification, MultiQC interpretation and the controlled storage workflow; then to Training 4 for DESeq2.

### Training checklist

- [ ] I know why a full-depth run differs from the 5,000-read pilot.
- [ ] I imported 32 FASTQs, kept the RDS sources unchanged and verified all 32 MD5 hashes.
- [ ] I generated an exact 16-sample full-depth samplesheet.
- [ ] I reused the pilot deployment config but created isolated full-run work, runstate and output locations.
- [ ] I can identify **Trim Galore** and all enabled full-data QC steps in the launcher.
- [ ] I ran in tmux, saved the output pointer and know how to resume safely.
- [ ] I can locate Nextflow logs, the trace, the MultiQC report, BAMs and the paired Salmon gene count/length matrices.

**Discuss with Matt:** Which computational stages take the most disk space and CPU? Why are the count and length matrices kept as separate, paired inputs for DESeq2? Why does a successful Nextflow exit not automatically mean the samples are biologically suitable for analysis?

**References:** White Lab pilot training document (15 October 2026); read-only `/projects/active/TDP43_LOF/README.md`; [nf-core/rnaseq 3.23.0 usage](https://nf-co.re/rnaseq/3.23.0/docs/usage); [nf-core/rnaseq 3.23.0 parameters](https://nf-co.re/rnaseq/3.23.0/parameters).
