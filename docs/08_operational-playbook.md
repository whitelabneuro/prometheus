# Operational playbook for Prometheus

## Overview

This document provides the practical day-to-day operating model for Prometheus as the White Lab local bioinformatics workstation.

It is intended as a concise operational reference for:

- running workflows locally
- keeping storage usage disciplined
- using RDS appropriately
- handling routine maintenance
- troubleshooting common issues
- keeping Prometheus aligned with CREATE HPC usage patterns

This is not a hardware build document. It is the practical playbook for routine use after setup and validation.

---

## Role of Prometheus

Prometheus should be treated as:

- a local Linux bioinformatics workstation
- a development and debugging platform
- a moderate-scale workflow execution machine
- a pre-production staging layer before CREATE HPC
- a fast local NVMe execution environment with shared RDS access

Prometheus should **not** be treated as:

- the lab’s long-term archive
- a substitute for CREATE on very large parallel jobs
- a reason to keep everything permanently on local scratch

---

## Core operating principles

### 1. Keep active execution on local NVMe

Use `/scratch1` for heavy workflow execution and task work directories.

### 2. Keep caches separate from work

Use `/scratch2` for:

- Singularity cache
- Nextflow cache
- micromamba package cache
- temporary files

### 3. Keep retained outputs organised

Use `/projects` for:

- launch directories
- active project organisation
- retained results
- pipeline outputs worth browsing or sharing locally

### 4. Use RDS as the shared institutional destination

Use `/rds/prj/bcn_whitema_rbp` for:

- shared copies
- project handoff
- CREATE-aligned storage targets
- important outputs that should not remain local-only

### 5. Avoid using `/` for workflow churn

Do not allow large workflow work directories or container caches to accumulate on the OS disk.

---

## Standard run pattern

The standard local run pattern on Prometheus is:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

For nf-core test runs:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile test,singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

This should be treated as the default documented pattern for local workflow execution.

---

## Standard storage usage

### `/scratch1`

Use for:
- live workflow work directories
- heavy task execution
- temporary staged data during active runs

Do not rely on `/scratch1` as the only long-term location of important outputs.

### `/scratch2`

Use for:
- Singularity cache
- Nextflow framework/assets/plugins
- micromamba package cache
- `TMPDIR`

This is infrastructure/cache space rather than a user-facing results area.

### `/projects`

Use for:
- local project organisation
- launch directories
- retained results
- references
- pipeline repos
- active analysis folders

### `/archive`

Use for:
- slower local overflow
- temporary holding
- non-performance-critical staging

### `/rds/prj/bcn_whitema_rbp`

Use for:
- canonical shared storage
- lab-visible datasets and outputs
- handoff to CREATE-aligned storage
- outputs that need to persist beyond the local workstation context

---

## Standard shell environment

Prometheus depends on the following environment configuration being present in the user shell:

```bash
export PATH="$HOME/.local/bin:$PATH"

export MAMBA_ROOT_PREFIX=/projects/micromamba
export CONDA_PKGS_DIRS=/scratch2/micromamba/pkgs

export NXF_HOME=/scratch2/nextflow_cache
export TMPDIR=/scratch2/tmp

export NXF_SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity
export SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity
export APPTAINER_CACHEDIR=/scratch2/container_cache/apptainer

eval "$(micromamba shell hook --shell bash)"
```

These are typically maintained in `~/.bashrc`.

If shell behaviour looks wrong, this is one of the first places to inspect.

---

## Routine checks before major runs

Before starting a real workflow run, check the following:

### Storage availability

```bash
df -h / /scratch1 /scratch2 /projects /archive /rds/prj/bcn_whitema_rbp
```

### Key environment variables

```bash
echo "$NXF_HOME"
echo "$TMPDIR"
echo "$NXF_SINGULARITY_CACHEDIR"
echo "$MAMBA_ROOT_PREFIX"
```

### Nextflow version

```bash
nextflow -version
```

### Singularity version

```bash
singularity --version
```

### RDS visibility

```bash
ls /rds/prj/bcn_whitema_rbp | head
```

If any of these fail or look unexpected, fix that before launching a larger workflow.

---

## Monitoring during workflow runs

Prometheus includes several useful monitoring tools for active workflow supervision.

### Disk usage across storage tiers

```bash
watch -n 5 'df -h / /scratch1 /scratch2 /projects /rds/prj/bcn_whitema_rbp'
```

### Growth of work and cache locations

```bash
watch -n 5 'du -sh /scratch1/nextflow_work/<run_name> /scratch2/container_cache/singularity /scratch2/nextflow_cache 2>/dev/null'
```

### Workflow log

```bash
tail -f .nextflow.log
```

### CPU and memory

```bash
htop
```

### Disk I/O

```bash
sudo iotop
```

### Network activity

```bash
sudo iftop -i <interface>
```

These tools are especially useful during first runs of new workflows, first-time container pulls, or when a run appears slow or stalled.

---

## Resume behaviour

Prometheus has been validated for strong Nextflow resume behaviour.

Best practice:

- rerun from the same launch directory
- use the same `-work-dir`
- include `-resume`

Example:

```bash
nextflow run <pipeline> \
  -profile singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name> \
  -resume
```

This is especially useful during:

- debugging
- parameter adjustment
- interrupted runs
- iterative workflow development

---

## Promoting results to RDS

Prometheus should not be treated as the only home of important results.

When outputs are worth retaining or sharing, promote them to RDS.

Example:

```bash
rsync -avh --progress /projects/test_runs/<run_name>/results/ \
  /rds/prj/bcn_whitema_rbp/<destination_path>/
```

Recommended use cases include:

- validated test outputs worth retaining
- project deliverables
- shared data for downstream CREATE use
- lab-visible handoff material

---

## RDS maintenance note

The KCL RDS mount depends on a root-only SMB credentials file stored at:

```text
/root/.smb/rds_bcn_whitema_rbp.cred
```

If the KCL password changes, this file must be updated manually:

```bash
sudo nano /root/.smb/rds_bcn_whitema_rbp.cred
```

Then edit the `password=` line.

If this is not updated, the persistent RDS mount will fail.

This is one of the most important maintenance items on Prometheus.

---

## Routine cleanup guidance

### Safe to clean periodically

- old run directories in `/projects/test_runs`
- obsolete work directories in `/scratch1/nextflow_work`
- stale temporary files in `/scratch2/tmp`
- unnecessary local copies after results have been promoted to RDS

### Be careful with

- `/scratch2/container_cache/singularity`
- `/scratch2/nextflow_cache`

Removing these is not inherently wrong, but it will force future re-download of:

- container images
- pipeline assets
- plugins
- framework artefacts

That may be acceptable occasionally, but should be done deliberately rather than casually.

### Do not casually edit

- `/etc/fstab`
- `/root/.smb/rds_bcn_whitema_rbp.cred`
- `~/.nextflow/config`
- `~/.bashrc`

Those are core configuration points.

---

## Common problems and first checks

### Problem: workflow pulls containers into the work directory

First check:

```bash
echo "$NXF_SINGULARITY_CACHEDIR"
```

Expected:

```text
/scratch2/container_cache/singularity
```

If this is blank or wrong, reload shell config.

### Problem: RDS mount disappears or fails

Check:

```bash
mount | grep bcn_whitema_rbp
```

If not mounted:
- inspect `/etc/fstab`
- confirm credentials file exists
- confirm KCL password is current
- run `sudo mount -a`

### Problem: run is slow on first execution

Common reasons:
- first-time container pulls
- Wi-Fi bottleneck
- first-time pipeline asset download
- remote test data staging
- cold cache

This is often normal. Compare with a `-resume` rerun before assuming the workstation is underperforming.

### Problem: OS disk starts filling

Check whether:
- cache variables are set correctly
- someone launched a workflow without explicit `-work-dir`
- results or temporary data were written under home or root unintentionally

### Problem: unexpected local `work/` directory appears under a project

This usually means a workflow was launched without the intended explicit `-work-dir`.

---

## When to use Prometheus vs CREATE

### Prefer Prometheus for
- early development
- pipeline debugging
- testing parameters
- moderate-scale local runs
- checking storage and output behaviour
- validating end-to-end workflow structure before scaling up

### Prefer CREATE for
- large cohorts
- many-sample production runs
- long-running heavily parallel jobs
- jobs that benefit from queue-managed cluster execution

---

## Recommended operational rhythm

A sensible routine for Prometheus is:

1. prepare or sync input data
2. launch locally with explicit `-work-dir`
3. monitor the first run
4. rerun with `-resume` if needed
5. inspect results under `/projects`
6. promote retained outputs to RDS
7. clean up stale local artefacts when appropriate

This keeps the workstation fast, organised, and aligned with its intended role.

---

## Practical conclusion

Prometheus should be operated as a fast, disciplined local execution layer with clear storage separation and strong alignment to CREATE HPC conventions.

The most important habits are:

- use explicit `-work-dir`
- keep caches on `/scratch2`
- keep retained outputs under `/projects`
- promote important results to RDS
- update the RDS credentials file whenever the KCL password changes
