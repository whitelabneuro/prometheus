# Nextflow and container setup on Prometheus

## Overview

Prometheus is configured to run Nextflow workflows locally while keeping heavy workflow execution, framework cache, and container cache off the OS disk.

The main goals are:

- predictable local execution
- reproducible nf-core and Nextflow workflow behaviour
- fast work directory performance on local NVMe
- clean separation between work directories, caches, and retained outputs
- close alignment with CREATE HPC usage patterns

The standard runtime model on Prometheus is:

- Nextflow as workflow engine
- local executor
- Singularity CE as primary container runtime
- explicit work directories on `/scratch1`
- framework and container caches on `/scratch2`
- retained outputs under `/projects`

---

## Core installation locations

### Nextflow executable

```text
~/.local/bin/nextflow
```

### Nextflow home/cache

```text
/scratch2/nextflow_cache
```

### Singularity cache

```text
/scratch2/container_cache/singularity
```

### Temporary files

```text
/scratch2/tmp
```

### Standard work directory root

```text
/scratch1/nextflow_work
```

---

## Shell environment

Prometheus uses the following environment variables for Nextflow and container execution:

```bash
export NXF_HOME=/scratch2/nextflow_cache
export TMPDIR=/scratch2/tmp

export NXF_SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity
export SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity
export APPTAINER_CACHEDIR=/scratch2/container_cache/apptainer
```

These are placed in `~/.bashrc` so that workflow execution uses the intended cache layout consistently.

---

## Why `NXF_SINGULARITY_CACHEDIR` matters

During early validation runs, Singularity images were initially written into the run-local work directory rather than the intended centralized cache location.

The key fix was to ensure:

```bash
export NXF_SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity
```

Without this, Nextflow may fall back to storing pulled images inside the work directory, which is less tidy and makes cache behaviour less predictable.

Using a centralized Singularity cache keeps large image downloads in one controlled location and avoids repeated clutter inside individual run work directories.

---

## Global Nextflow configuration

Prometheus uses a user-level Nextflow config at:

```text
~/.nextflow/config
```

The validated configuration is:

```groovy
process {
  executor = 'local'
  cache = 'lenient'
}

workDir = '/scratch1/nextflow_work'

singularity {
  enabled = true
  autoMounts = true
  cacheDir = '/scratch2/container_cache/singularity'
}

docker {
  enabled = false
}

cleanup = false

trace {
  enabled = true
}

timeline {
  enabled = true
}

report {
  enabled = true
}

dag {
  enabled = true
}
```

---

## Configuration rationale

### `executor = 'local'`
Prometheus is a workstation, so workflows run with the local executor rather than a scheduler such as SLURM.

### `cache = 'lenient'`
This supports practical local reruns and resume behaviour without being overly strict in common local development situations.

### `workDir = '/scratch1/nextflow_work'`
Provides a sensible default work root so active task execution lands on the intended NVMe scratch area rather than under arbitrary launch directories.

### `singularity.enabled = true`
Makes Singularity the default container runtime for workflow execution.

### `singularity.autoMounts = true`
Supports practical bind behaviour for typical workflow inputs and outputs.

### `singularity.cacheDir = '/scratch2/container_cache/singularity'`
Centralizes container image storage on the cache-heavy scratch tier.

### `docker.enabled = false`
Prevents Docker from becoming the default runtime in routine workflow execution.

### `cleanup = false`
Retains workflow artefacts during development and troubleshooting. This is useful while Prometheus is being used as a development and validation machine.

### trace / timeline / report / dag
These are enabled to provide workflow provenance and debugging outputs automatically.

---

## Standard execution pattern

The recommended run pattern on Prometheus is:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

For nf-core test profiles:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile test,singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

---

## Why explicit `-work-dir` is still recommended

Even though a global `workDir` is configured, using explicit `-work-dir` in documented production-style runs is still recommended.

Benefits include:

- clearer run reproducibility
- easier documentation in project repos
- less ambiguity when inspecting old run commands
- easier separation between run names and task execution locations
- clearer cleanup and troubleshooting

For Prometheus, explicit `-work-dir` is the preferred documented pattern.

---

## Validated runtime behaviour

Prometheus has now been validated to show the intended separation:

### `/scratch1`
Used for:
- hashed Nextflow work directories
- task-local execution
- staged inputs and temporary task artefacts

### `/scratch2`
Used for:
- Singularity container cache
- Nextflow framework and assets cache
- plugin downloads
- temp files

### `/projects`
Used for:
- run launch directories
- retained results
- pipeline information
- human-facing outputs

This is the core behaviour that the workstation was designed to achieve.

---

## Validated workflow examples

The following examples have been run successfully on Prometheus:

### Singularity pull / run test
A local Singularity pull and run of `hello-world` confirmed:

- network image pull
- container conversion
- local container execution
- basic cache behaviour

### `nextflow run hello`
Confirmed:
- Nextflow installation works
- local executor works
- simple workflow execution works

### `nf-core/demo`
Confirmed:
- nf-core pipeline pull works
- plugin resolution works
- Singularity execution works
- pipeline output structure is generated correctly
- resume behaviour is functional

### `nf-core/rnaseq`
Confirmed:
- realistic nf-core RNA-seq workflow execution works
- output structure matches expectations
- work/cache/output separation is correct
- resume behaviour is strong and highly reusable

---

## Resume behaviour

Resume behaviour has been explicitly validated on Prometheus.

For `nf-core/rnaseq`:

- initial run completed successfully
- resume run completed successfully
- high cache reuse was observed
- rerun duration was dramatically shorter than the initial run

This confirms that the current Nextflow/cache configuration is functioning as intended.

---

## Monitoring during runs

Useful monitoring commands during workflow execution include:

### Disk usage

```bash
watch -n 5 'df -h / /scratch1 /scratch2 /projects /rds/prj/bcn_whitema_rbp'
```

### Cache and work growth

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

---

## Practical conclusion

Prometheus is now configured to run Nextflow workflows in a way that is:

- fast locally
- reproducible
- storage-aware
- easy to debug
- aligned with CREATE HPC execution practices wherever practical
