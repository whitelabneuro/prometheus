# Software stack on Prometheus

## Overview

Prometheus uses a layered software strategy designed to keep the operating system clean, keep workflow execution reproducible, and align local development with CREATE HPC as closely as practical.

The core principles are:

- use system packages for core OS-level tooling
- use a lightweight environment manager for user-space packages
- use Nextflow as the workflow engine
- use Singularity CE as the primary container runtime for pipeline execution
- keep heavy caches off the OS disk
- retain Docker as a secondary development/runtime tool where useful

---

## Software layers

### 1. System packages via APT

APT is used for:

- core Linux administration tools
- monitoring and hardware utilities
- Java runtime
- CIFS mount support
- Singularity CE
- Docker
- basic developer tooling

This keeps core infrastructure under normal Ubuntu package management.

Examples include:

- `build-essential`
- `git`
- `curl`
- `wget`
- `rsync`
- `tmux`
- `htop`
- `tree`
- `jq`
- `nvme-cli`
- `smartmontools`
- `lm-sensors`
- `sysstat`
- `iotop`
- `iftop`
- `ethtool`
- `pigz`
- `fio`
- `stress-ng`
- `default-jre`
- `python3-pip`
- `cifs-utils`
- `singularity-container`

---

### 2. Micromamba for user-space environments

Micromamba is used for user-space package management rather than installing large numbers of bioinformatics tools directly into the OS.

This keeps the system cleaner and gives a reproducible, flexible way to manage analysis environments.

Prometheus uses:

- micromamba binary in `~/.local/bin`
- environment root at `/projects/micromamba`
- package cache at `/scratch2/micromamba/pkgs`

This split is intentional:

- environments persist under `/projects`
- package cache uses fast scratch-backed storage under `/scratch2`

---

### 3. Nextflow as workflow engine

Nextflow is installed directly as a user-local executable rather than inside conda or micromamba.

This avoids unnecessary coupling between workflow orchestration and package environments.

Prometheus uses:

- `nextflow` in `~/.local/bin`
- Nextflow home/cache at `/scratch2/nextflow_cache`

This keeps framework downloads, pipeline assets, plugins, and runtime cache off the OS disk.

---

### 4. Singularity CE as primary container runtime

Prometheus uses Singularity CE as the primary container runtime for pipeline execution.

Rationale:

- closer alignment with CREATE HPC practices
- suitable for nf-core and Nextflow workflows
- avoids reliance on Docker daemon for the main pipeline execution path
- supports portable workflow behaviour between workstation and HPC

Prometheus uses a centralized Singularity cache under:

- `/scratch2/container_cache/singularity`

This is controlled via:

- `NXF_SINGULARITY_CACHEDIR`
- `SINGULARITY_CACHEDIR`

---

### 5. Docker as secondary tool

Docker is installed and functional on Prometheus, but it is treated as secondary rather than primary for pipeline execution.

Docker remains useful for:

- image inspection
- container development
- compatibility with documentation that assumes Docker
- ad hoc runtime testing where appropriate

However, the default documented execution path for Prometheus should remain:

- Nextflow + Singularity CE

---

## Installed core components

### Java
Prometheus uses the Ubuntu default Java runtime installed via APT.

This supports Nextflow execution cleanly without requiring a separate environment manager.

### Nextflow
Installed as a user-local executable in:

```text
~/.local/bin/nextflow
```

### Micromamba
Installed as a user-local executable in:

```text
~/.local/bin/micromamba
```

### Singularity CE
Installed from Ubuntu packages and used as the primary workflow container runtime.

### Docker
Preinstalled and active on Prometheus, retained as a secondary runtime/development tool.

---

## Key environment configuration

Prometheus uses the following shell-level environment variables:

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

These are typically placed in `~/.bashrc`.

---

## Why the split locations matter

### Persistent and user-facing
- `/projects/micromamba`
- `/projects/test_runs`
- `/projects/pipelines`
- `/projects/refdata`

### Fast scratch/cache
- `/scratch1/nextflow_work`
- `/scratch2/nextflow_cache`
- `/scratch2/container_cache/singularity`
- `/scratch2/micromamba/pkgs`
- `/scratch2/tmp`

This keeps:

- the OS disk clean
- high-write workflow churn on NVMe scratch
- retained outputs organised under `/projects`
- cache-heavy content separated from active work dirs

---

## Monitoring and maintenance tools

Prometheus includes a practical set of system monitoring tools to support workflow debugging and workstation health checks.

Examples include:

- `smartmontools` for SMART health
- `nvme-cli` for NVMe inspection
- `lm-sensors` for thermal monitoring
- `sysstat` for performance monitoring
- `iotop` for disk I/O inspection
- `iftop` for network activity
- `htop` and `btop` for process and resource monitoring
- `fio` for storage benchmarking
- `stress-ng` for stress testing

These tools are particularly useful when validating storage and container behaviour during pipeline runs.

---

## Software strategy relative to CREATE HPC

Prometheus is intended to align with CREATE where practical, while remaining convenient as a local workstation.

### Shared principles with CREATE
- Nextflow-based pipeline execution
- Singularity-family container execution
- clear separation of work directories and retained outputs
- central shared storage destination via `/rds/prj/bcn_whitema_rbp`

### Differences from CREATE
- executor is local rather than scheduler-backed
- no SLURM submission layer
- user-local shell configuration controls cache and runtime behaviour
- faster iteration for development and debugging due to local NVMe and absence of queue delays

---

## Recommended operational policy

### Use APT for
- core OS tools
- Java
- Singularity CE
- CIFS support
- monitoring tools
- Docker

### Use micromamba for
- bioinformatics tool environments
- ad hoc user-space utilities
- project-specific package environments

### Use Nextflow directly for
- workflow orchestration
- nf-core execution
- local pipeline development
- reproducible testing

### Use Singularity CE by default for
- routine pipeline execution
- nf-core workflows
- CREATE-aligned workflow behaviour

### Use Docker when needed for
- development
- inspection
- compatibility cases
- non-primary runtime experiments

---

## Practical conclusion

Prometheus is deliberately configured so that the software stack supports:

- clean local development
- reproducible workflow execution
- predictable cache behaviour
- minimal pressure on the OS disk
- close operational alignment with CREATE HPC
