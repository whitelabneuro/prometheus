# White Lab | Prometheus onboarding and nf-core/rnaseq training

**Trainee:** Ella  
**System:** Prometheus (Ubuntu, SSH, local Nextflow + Singularity)  
**Exercise:** TDP-43 loss-of-function RNA-seq pilot with a deliberately reduced read count  
**Prepared:** 15 October 2026  
**Status:** Updated after Prometheus access checks on 15 October 2026. PI setup is complete; Ella to create and run her own training project.

## Learning outcomes

By the end of this exercise you should be able to (1) explain where datasets and processing files live on Prometheus; (2) create a new research project without touching existing analyses; (3) transfer and verify a small number of sequencing files from the group RDS; (4) construct a valid nf-core/rnaseq samplesheet and a local Nextflow configuration; (5) execute, inspect and resume an independent Nextflow run.

**Important:** This is a *technical pipeline test*. A few thousand reads per sample are insufficient for meaningful gene expression, splice-event, or library QC interpretation. The validated production TDP-43 workflow uses Trim Galore on full-depth data, not fastp read subsampling.

## Part A | How Prometheus is organised

| Location | What lives here | Your access / rule |
|---|---|---|
| `/home/ella` | Your login settings and small scripts | Private; avoid large analysis files |
| `/projects/active/TDP43_LOF` | Completed White Lab TDP-43 project, inputs, metadata, scripts and results | Read existing files; **do not edit** |
| `/projects/refdata` | Reference FASTA, GTF and other reusable reference files | Read only |
| `/projects/users/ella` | Your projects and persistent, final outputs | **Create this yourself** after PI preflight |
| `/scratch1/users/ella` | Large Nextflow task/work files (intermediates) | **Create this yourself** |
| `/scratch2/users/ella` | Nextflow home, run history, private container cache and temporary files | **Create this yourself** |
| `/scratch2/container_cache/singularity` | Previously downloaded laboratory Singularity images | Read-only shared image library |
| `/archive` | Long-term sequencing archive | Read only; do not use as work storage |
| `/rds/prj/bcn_whitema_rbp` | Group KCL RDS: sequencing sources and approved export destinations | Read sequencing originals; write only to agreed upload/export areas |

**What belongs where?** `/projects` holds results we need to keep. `/scratch1` runs compute-heavy tasks. `/scratch2` holds regenerable software state, downloaded images, run metadata and temporary files. RDS is the laboratory’s network-backed source/transfer store; `/archive` is local long-term archival storage. Each member has a separate Unix account and no administrator (`sudo`) privileges. The `prometheusresearchers` group permits independent project creation and locally controlled RDS access; it does not grant write access to completed `/projects/active` analyses.

The production TDP-43 design keeps the launcher, samplesheet and outputs in `/projects`; task directories in `/scratch1`; and Nextflow runstate and caches in `/scratch2`. This exercise follows **that division of roles**, while using your own user area rather than the production project.

## Part B | Administrator preparation: already completed (PI reference)

The PI completed the following on 15 October 2026. **Ella does not repeat these commands, install software or need sudo.**

| Item | Established configuration / observed result |
|---|---|
| Linux identity | `ella` (UID 1003), no sudo; SSH functioning |
| Researcher group | `prometheusresearchers` (GID 1004); Ella is a member |
| Shared Nextflow | `/usr/local/bin/nextflow`, framework JAR in `/opt/nextflow/framework/25.10.4/` |
| Runtime validation | Nextflow **25.10.4 build 11173** runs as Ella without sudo |
| Researcher parent directories | `/projects/users`, `/scratch1/users`, `/scratch2/users` owned by `root:prometheusresearchers` with mode **3770** |
| Existing TDP-43 inputs and references | Readable by Ella, not writable |
| Group RDS access | FASTQ source read test passed; directory write permission test passed |

Mode `3770` sets both the setgid bit (new directories inherit the researcher group) and the sticky bit (one researcher cannot ordinarily delete another person's top-level entry). Personal project directories will have private permissions (`umask 077`). A new SSH session is required after a group-membership change.

**RDS caveat:** `test -w` verifies locally presented directory permissions but is not proof that the remote SMB server will accept a file creation. A small, authorised upload test should be performed in an agreed RDS export location before relying on two-way transfer. This mount uses a shared KCL SMB connection identity, so actions are not separately authenticated to RDS as Ella; follow group data-handling rules. Original sequencing FASTQs remain immutable by laboratory convention even though the mount permits writes.

For an administrator to recheck the prerequisites, without altering files:

```bash
id ella
ls -ld /projects/users /scratch1/users /scratch2/users
sudo -H -u ella env NXF_VER=25.10.4 \
  NXF_DIST=/opt/nextflow/framework NXF_HOME=/home/ella/.nextflow \
  /usr/local/bin/nextflow -version
sudo -l -U ella
```

**Do not** recreate the parent directories, change their group, chmod the whole RDS, or alter the completed TDP-43 project. Any server-side access problem should go back to the PI.

## Part C | Log in and orient yourself (Ella)

From **Terminal on your Mac**, while connected to the approved KCL network/VPN:

```bash
ssh ella@10.131.34.252
```

If your SSH key has a non-default filename, connect using `ssh -i ~/.ssh/YOUR_PRIVATE_KEY ella@10.131.34.252`. Never share your private key. **After the terminal prompt changes to `ella@prometheus`**, run:

```bash
whoami
hostname
id  # Confirm membership in prometheusresearchers
pwd
```

Once logged in, explore (read-only):

```bash
ls /projects/active/TDP43_LOF
ls /projects/refdata/Homo_sapiens/GRCh38/Gencode
ls /rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw | head
df -h /projects /scratch1 /scratch2 /archive
```

## Part D | Create your own directories (Ella)

You create the following yourself; your PI has established only the three shared `/users` parents:

```bash
umask 077
mkdir -p /projects/users/ella/TDP43_LOF_training/{inputs/raw,metadata,results,logs,docs}
mkdir -p /scratch1/users/ella/nextflow_work/TDP43_LOF_training
mkdir -p /scratch2/users/ella/{nextflow_home,runstate/TDP43_LOF_training,container_cache/singularity,container_cache/runtime,tmp}

PROJECT=/projects/users/ella/TDP43_LOF_training
ls -ld "$PROJECT" /scratch1/users/ella /scratch2/users/ella
```

All three directory roots should be owned by you. Keep the project name consistent throughout the exercise. Do not write under `/projects/active/TDP43_LOF`.

### Test the shared Nextflow installation

```bash
export NXF_VER=25.10.4
export NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
/usr/local/bin/nextflow -version
singularity --version
java -version
```

Expect **Nextflow 25.10.4**. `NXF_DIST` directs the launcher to the centrally installed read-only JAR, whereas `NXF_HOME` is *your* writable pipeline/cache home. You do not need sudo.

Next, fetch the **pinned pipeline code** into your personal Nextflow home (this may require external network access):

```bash
/usr/local/bin/nextflow pull nf-core/rnaseq -r 3.23.0
```

This is not the same as installing Nextflow itself. If the pull fails due to network restrictions, stop and ask the PI to supply a vetted copy from the existing laboratory pipeline cache. Do not work around the restriction by editing the established project.

## Part E | Understand the TDP-43 dataset

The White Lab TDP-43 LOF study uses KOLF2.1J NGN2-induced cortical neurons, two perturbations (TDP-43 knockdown versus scramble), and two treatments (DMSO versus 11J), with four independent differentiation batches. This gives 16 biological libraries and 32 paired-end FASTQs.

For this initial exercise, use only **two biological libraries from batch 1**:

| Sample | Interpretation | Files |
|---|---|---|
| `TDP-SCR-DMSO-1` | Scramble, DMSO, batch 1 | R1 + R2 |
| `TDP-KD-DMSO-1` | TDP-43 knockdown, DMSO, batch 1 | R1 + R2 |

The same batch number indicates a shared differentiation batch; it does **not** make these paired before/after observations. For future full analyses, the biological batches are handled in downstream statistical designs. The 11J treatment must not be labelled a validated NMD inhibition condition until supported by QC.

## Part F | Onboard the FASTQs from group RDS (Ella)

First, find and inspect the source without modifying it:

```bash
RDS=/rds/prj/bcn_whitema_rbp/RNA_seq_studies/input_sequencing_files/TDP43_LOF/inputs/raw
PROJECT=/projects/users/ella/TDP43_LOF_training
ls -lh "$RDS"/TDP-{SCR,KD}-DMSO-1_R{1,2}_001.fastq.gz
ls -lh "$RDS"/TDP-{SCR,KD}-DMSO-1_R{1,2}_001.fastq.gz.md5
```

Now copy **only four FASTQs and their four MD5 sidecars** into your project's `inputs/raw`, using `rsync` so that an interrupted transfer can be repeated. This imports real research data without modifying the RDS copies:

```bash
for sample in TDP-SCR-DMSO-1 TDP-KD-DMSO-1; do
  for read in R1 R2; do
    filename="${sample}_${read}_001.fastq.gz"
    rsync -av --no-perms --partial --info=progress2 \
      "$RDS/$filename" "$PROJECT/inputs/raw/"
    rsync -av --no-perms --partial \
      "$RDS/$filename.md5" "$PROJECT/inputs/raw/"
  done
done
```

**Why MD5?** An MD5 hash checks that the copied file's bytes match the supplier's recorded hash. It is an integrity check, not a security mechanism. The sidecars may use different formatting, so this check extracts the first 32-digit hexadecimal hash from each sidecar:

```bash
cd "$PROJECT/inputs/raw"
for fastq in *.fastq.gz; do
  expected=$(grep -Eom1 '[0-9A-Fa-f]{32}' "$fastq.md5" | tr 'A-F' 'a-f')
  actual=$(md5sum "$fastq" | awk '{print $1}')
  if [[ -n "$expected" && "$expected" == "$actual" ]]; then
    echo "PASS: $fastq"
  else
    echo "FAIL: $fastq"; exit 1
  fi
done
```

All four must show `PASS`. If any fails, stop: do not run the pipeline on an unverified copy. Reconcile with the PI. Do not delete the source FASTQs or sidecar checksums.

**Storage tip:** These are full-sized FASTQs. fastp limits the reads handed to later stages, but does not magically make the original files small. Copy only the two selected samples for this exercise, and do not duplicate all 16 libraries unnecessarily.

## Part G | Create the nf-core samplesheet (Ella)

Create `metadata/samplesheet.csv` in your project using `nano` (save with Ctrl+O, Enter; exit Ctrl+X):

```bash
nano /projects/users/ella/TDP43_LOF_training/metadata/samplesheet.csv
```

Enter these **three CSV lines exactly**:

```csv
sample,fastq_1,fastq_2,strandedness
TDP-SCR-DMSO-1,/projects/users/ella/TDP43_LOF_training/inputs/raw/TDP-SCR-DMSO-1_R1_001.fastq.gz,/projects/users/ella/TDP43_LOF_training/inputs/raw/TDP-SCR-DMSO-1_R2_001.fastq.gz,auto
TDP-KD-DMSO-1,/projects/users/ella/TDP43_LOF_training/inputs/raw/TDP-KD-DMSO-1_R1_001.fastq.gz,/projects/users/ella/TDP43_LOF_training/inputs/raw/TDP-KD-DMSO-1_R2_001.fastq.gz,auto
```

Verify the sheet and FASTQ paths:

```bash
cat /projects/users/ella/TDP43_LOF_training/metadata/samplesheet.csv
ls -lh /projects/users/ella/TDP43_LOF_training/inputs/raw/*.fastq.gz
```

The required columns are `sample,fastq_1,fastq_2,strandedness`. `auto` allows the pipeline to infer library strandedness; the reduced read sample may make this inference unstable.

Check each FASTQ path directly against the samplesheet (this is more reliable than checking filenames by eye):

```bash
python3 - <<'PY'
import csv
from pathlib import Path
sheet = Path('/projects/users/ella/TDP43_LOF_training/metadata/samplesheet.csv')
with sheet.open(newline='') as f:
    reader = csv.DictReader(f)
    assert reader.fieldnames == ['sample', 'fastq_1', 'fastq_2', 'strandedness']
    rows = list(reader)
assert len(rows) == 2, f'Expected two samples, found {len(rows)}'
for row in rows:
    for mate in ('fastq_1', 'fastq_2'):
        path = Path(row[mate])
        assert path.is_file(), f'Missing {path}'
        print('PASS:', row['sample'], mate, path.name)
print('Samplesheet preflight: PASS')
PY
```


## Part H | Create your `nextflow.config` (Ella)

Open the config in your project:

```bash
nano /projects/users/ella/TDP43_LOF_training/nextflow.config
```

Enter:

```groovy
// Based on /projects/active/TDP43_LOF/nextflow.config
// Local execution and restart behaviour follow the validated project.
process {
  executor = 'local'
  cache = 'lenient'
  maxRetries = 2
  errorStrategy = 'retry'
}

singularity {
  enabled = true
  autoMounts = true
  // Existing laboratory image store is used read-only.
  libraryDir = '/scratch2/container_cache/singularity'
  // Your own cache is writable; no group-wide write privileges needed.
  cacheDir = '/scratch2/users/ella/container_cache/singularity'
}

docker {
  enabled = false
}

cleanup = false
trace.enabled = true
timeline.enabled = true
report.enabled = true
dag.enabled = true
```

The production configuration pointed `singularity.cacheDir` directly at the shared store. You cannot write there, so your version separates a **shared read-only image library** from your **private writable cache**. This is a deployment difference, not a biological pipeline parameter change.

## Part I | Create your run launcher (Ella)

Open the launcher:

```bash
nano /projects/users/ella/TDP43_LOF_training/run_nfcore_rnaseq.sh
```

Enter the following Bash script:

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT=/projects/users/ella/TDP43_LOF_training
WORKDIR=/scratch1/users/ella/nextflow_work/TDP43_LOF_training
RUNSTATE=/scratch2/users/ella/runstate/TDP43_LOF_training
OUTDIR="$PROJECT/results/nfcore_rnaseq_fastp_pilot"
LOGFILE="$PROJECT/logs/nfcore_rnaseq_fastp_pilot.log"
SAMPLESHEET="$PROJECT/metadata/samplesheet.csv"
CONFIG="$PROJECT/nextflow.config"
FASTA=/projects/refdata/Homo_sapiens/GRCh38/Gencode/GRCh38.primary_assembly.genome.fa.gz
GTF=/projects/refdata/Homo_sapiens/GRCh38/Gencode/gencode.v49.primary_assembly.annotation.gtf.gz

# One Nextflow executable/JAR shared by the lab; personal writable state.
export NXF_VER=25.10.4
export NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
export NXF_SINGULARITY_LIBRARYDIR=/scratch2/container_cache/singularity
export NXF_SINGULARITY_CACHEDIR=/scratch2/users/ella/container_cache/singularity
export SINGULARITY_CACHEDIR=/scratch2/users/ella/container_cache/runtime
export SINGULARITY_TMPDIR=/scratch2/users/ella/tmp
export TMPDIR=/scratch2/users/ella/tmp
export NXF_ANSI_LOG=false

mkdir -p "$WORKDIR" "$RUNSTATE" "$OUTDIR" "$(dirname "$LOGFILE")" "$TMPDIR"
for f in "$SAMPLESHEET" "$CONFIG" "$FASTA" "$GTF"; do
  test -r "$f" || { echo "Cannot read: $f"; exit 1; }
done

# Nextflow .nextflow task cache + history are kept in the RUNSTATE folder.
cd "$RUNSTATE"

# First use: ./run_nfcore_rnaseq.sh
# After interruption/failure: RESUME=1 ./run_nfcore_rnaseq.sh
resume=()
if [[ "${RESUME:-0}" == 1 ]]; then resume=(-resume); fi

/usr/local/bin/nextflow run nf-core/rnaseq \
  -r 3.23.0 \
  -profile singularity \
  -c "$CONFIG" \
  -work-dir "$WORKDIR" \
  "${resume[@]}" \
  --input "$SAMPLESHEET" \
  --outdir "$OUTDIR" \
  --aligner star_salmon \
  --trimmer fastp \
  --extra_fastp_args '--reads_to_process 5000' \
  --min_trimmed_reads 100 \
  --skip_fastqc \
  --skip_linting \
  --skip_deseq2_qc \
  --fasta "$FASTA" \
  --gtf "$GTF" \
  --gencode \
  2>&1 | tee -a "$LOGFILE"
```

Make it executable and examine it:

```bash
chmod 700 /projects/users/ella/TDP43_LOF_training/run_nfcore_rnaseq.sh
bash -n /projects/users/ella/TDP43_LOF_training/run_nfcore_rnaseq.sh
```

**About the pilot switches:** `--trimmer fastp` and `--extra_fastp_args '--reads_to_process 5000'` take up to 5,000 paired reads through trimming per library. `--min_trimmed_reads 100` overrides the usual 10,000-read eligibility requirement. `--skip_fastqc` and `--skip_linting` prevent other modules from scanning the *entire original FASTQs* before the subsampling exercise; this is appropriate only because the supplier files have already been MD5-checked and this is a smoke test. `--skip_deseq2_qc` omits across-sample DESeq2 plots that are not useful for just two samples. STAR+Salmon processing and the remaining enabled QC/MultiQC stages still run. A very small input may be insufficient for some downstream QC stages; failure at 5,000 reads does not necessarily imply that the installation is broken.

## Part J | Preflight and launch (Ella)

Check capacity and the exact versions before launching:

```bash
PROJECT=/projects/users/ella/TDP43_LOF_training
df -h /projects /scratch1 /scratch2
free -h
export NXF_VER=25.10.4 NXF_DIST=/opt/nextflow/framework
export NXF_HOME=/scratch2/users/ella/nextflow_home
nextflow -version
singularity --version
bash -n "$PROJECT/run_nfcore_rnaseq.sh"
```

**Before running:** the source datasets may be large and even a 5,000-read test may build substantial human STAR/Salmon reference indices. Read subsampling reduces per-sample processing, **not reference-index construction**. Confirm available resources with the PI; a verified reusable index can be configured separately if one exists.

To execute in a persistent terminal session, first create a tmux session:

```bash
tmux new -s ella_rnaseq_pilot
```

**Once you are inside tmux** (the terminal has switched to the new session), execute:

```bash
cd /projects/users/ella/TDP43_LOF_training
./run_nfcore_rnaseq.sh
```

Detach from tmux with **Ctrl+B, then D**; later reconnect using `tmux attach -t ella_rnaseq_pilot`. Do not run a second copy of the same job concurrently. Check the Nextflow report, task state, STAR/Salmon outputs and MultiQC when the run finishes.

If the run stops and the original work directory remains intact, **resume rather than restarting**. Run this from inside the existing tmux session (or a new one after the previous run is no longer active):

```bash
cd /projects/users/ella/TDP43_LOF_training
RESUME=1 ./run_nfcore_rnaseq.sh
```

Do not delete `/scratch1/users/ella/nextflow_work/TDP43_LOF_training` or `/scratch2/users/ella/runstate/TDP43_LOF_training` before attempting resume. If any failures occur, record the failing process name, last 30 lines of the log, and the error in `.command.err` rather than deleting the work files.

### Optional supervised RDS off-ramp exercise

Ask the PI to identify or approve a dedicated **RDS export folder** first. Do not export into the sequencing `inputs/raw` directory, overwrite an original, or copy an entire large `results/` tree during this smoke test. Once the destination exists and is confirmed, replace the placeholder path below with the approved one, then upload a small text file and read it back:

```bash
# Example layout ONLY: confirm RDS_EXPORT with the PI before running.
RDS_EXPORT=/rds/prj/bcn_whitema_rbp/RNA_seq_studies/APPROVED_EXPORT_FOLDER/ella
PROJECT=/projects/users/ella/TDP43_LOF_training
printf 'Prometheus training transfer test\n' > "$PROJECT/docs/rds_transfer_test.txt"
# The PI-approved RDS_EXPORT destination must already exist.
rsync -rv --no-perms --no-owner --no-group \
  "$PROJECT/docs/rds_transfer_test.txt" "$RDS_EXPORT/"
md5sum "$PROJECT/docs/rds_transfer_test.txt" "$RDS_EXPORT/rds_transfer_test.txt"
```

Matching hashes verify the off-ramp. In normal use, export agreed final products, manifests and QC summaries to the appropriate RDS study directory; preserve original names, checksums and scientific provenance. Ask the PI before deleting a local copy.


## Part K | Training checklist and interpretation

- [ ] I can SSH to Prometheus under the `ella` account and have no sudo privileges.
- [ ] I understand the difference between `/projects`, `/scratch1`, `/scratch2`, `/archive`, and the group RDS.
- [ ] I created my own persistent project and scratch directories.
- [ ] I checked that centrally installed Nextflow is **25.10.4**.
- [ ] I copied the two selected RDS sample pairs and verified all four MD5 checks.
- [ ] I created and checked a valid paired-end samplesheet.
- [ ] I wrote and syntax-checked my own Nextflow configuration and launcher.
- [ ] I understand why only 5,000 reads are processed, and why this is not valid RNA-seq data analysis.
- [ ] I know how to start, detach, monitor, resume and review a Nextflow run.
- [ ] I can describe the approved RDS off-ramp and why the raw sequencing delivery must not be modified.
- [ ] I can identify the results, execution work folders, logs, reports and MultiQC summary.

**Discuss with the PI:** What did Nextflow reuse from the shared installation? Which steps downloaded containers? Did Fastp, STAR and Salmon execute? Which checks were disabled for the pilot, and what changes are required before processing all 16 samples at full read depth? What would change for a new project with different samples and references?

## References and source of truth

1. White Lab original project documentation: `/projects/active/TDP43_LOF/README.md`, `run_nfcore_rnaseq.sh`, and `nextflow.config` (supplied and reviewed 15 October 2026).
2. nf-core/rnaseq **3.23.0** usage and FASTQ sampling: https://nf-co.re/rnaseq/3.23.0/docs/usage/
3. nf-core/rnaseq **3.23.0** parameter reference: https://nf-co.re/rnaseq/3.23.0/parameters/
4. Nextflow Singularity cache versus library: https://docs.seqera.io/nextflow/container/singularity
5. Nextflow environment variables: https://docs.seqera.io/nextflow/reference/env-vars

**Lab policy:** treat existing project inputs and validated results as read-only; use your own workspace; avoid broad permission changes; keep a reproducible sample manifest and all software versions; ask before importing large volumes of sequencing data or deleting Nextflow work directories.
