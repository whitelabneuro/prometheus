# Storage layout on Prometheus

## Overview

Prometheus uses a split-storage model so that operating system files, active workflow execution, caches, retained project outputs, and shared institutional storage are clearly separated.

This improves:

- performance
- maintainability
- reproducibility
- ease of troubleshooting
- alignment with CREATE HPC usage patterns

---

## Mount points and roles

### `/`
Ubuntu operating system, installed software, system configuration, and user home directories.

This should **not** be used for heavy pipeline work directories, container caches, or large transient workflow outputs.

### `/scratch1`
Primary high-speed local execution space.

Intended uses:

- Nextflow `work` directories
- hashed task execution directories
- staged inputs during active runs
- task-local temporary outputs

### `/scratch2`
Cache-heavy and temporary infrastructure space.

Intended uses:

- Singularity container cache
- Nextflow framework and assets cache
- micromamba package cache
- temporary files via `TMPDIR`

### `/projects`
Structured local project space for retained and user-facing material.

Intended uses:

- project folders
- active analyses
- references
- pipeline repos
- retained test runs
- final local outputs

### `/archive`
Slower local holding and overflow space.

Intended uses:

- temporary local retention
- overflow storage
- intermediate staging that does not require NVMe performance

### `/rds/prj/bcn_whitema_rbp`
Persistent shared KCL Research Data Storage mounted over the network.

Intended uses:

- canonical shared lab storage
- shared project handoff
- institutional storage destination
- alignment with CREATE HPC paths

---

## Final model

| Mount point | Function | Performance class |
|---|---|---|
| `/` | OS, software, home | system |
| `/scratch1` | active Nextflow work dirs | fastest local execution |
| `/scratch2` | caches and temp | fast local cache |
| `/projects` | organised local project outputs | fast retained local storage |
| `/archive` | overflow / holding | slower local |
| `/rds/prj/bcn_whitema_rbp` | shared institutional storage | network storage |

---

## Practical workflow rules

### Use `/scratch1` for
- `-work-dir /scratch1/nextflow_work/<run_name>`

### Use `/scratch2` for
- `NXF_HOME`
- `NXF_SINGULARITY_CACHEDIR`
- `TMPDIR`
- micromamba package cache

### Use `/projects` for
- pipeline launch directories
- final results
- local test runs
- organised active project folders

### Use `/rds/prj/bcn_whitema_rbp` for
- canonical shared copies
- project handoff
- datasets shared across machines or users
- CREATE-aligned storage targets

---

## Standard directory skeleton

Example top-level structure:

```text
/scratch1/
  nextflow_work/

/scratch2/
  container_cache/
  micromamba/
  nextflow_cache/
  tmp/

/projects/
  active/
  micromamba/
  pipelines/
  refdata/
  test_runs/

/archive/
  staging/
  completed/

/rds/prj/bcn_whitema_rbp/
  ...
```

---

## Standard Nextflow launch pattern

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

This keeps:

- results under `/projects`
- heavy execution under `/scratch1`
- container and framework cache under `/scratch2`

---

## Relationship to CREATE HPC

Prometheus is intended to complement CREATE rather than replace it.

### Prometheus is best for
- local pipeline development
- debugging
- dry runs
- moderate-scale production runs
- rapid reruns using local NVMe
- validating storage and output behaviour before scaling up

### CREATE is best for
- large cohort processing
- heavy parallel production workloads
- long-running batch execution
- jobs that benefit from scheduler-managed shared compute

### Shared path consistency

A key design goal is that shared institutional storage appears at the same path on both systems:

- CREATE: `/rds/prj/bcn_whitema_rbp`
- Prometheus: `/rds/prj/bcn_whitema_rbp`

This reduces friction when moving between systems and simplifies documentation.

---

## Example storage behaviour from validation runs

The validated test workflows on Prometheus behaved as intended:

- Nextflow work directories were written to `/scratch1`
- Singularity cache and Nextflow framework assets were written to `/scratch2`
- retained pipeline outputs were written to `/projects`
- shared storage was available at `/rds/prj/bcn_whitema_rbp` for later sync or handoff

This confirms that the split-storage model is functioning correctly in practice rather than only in theory.

---

## Storage principle

Keep active execution on local NVMe, retain structured local outputs under `/projects`, and use `/rds/prj/bcn_whitema_rbp` as the canonical shared storage destination aligned with CREATE HPC.
