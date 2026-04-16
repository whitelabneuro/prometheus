# Validation runs on Prometheus

## Overview

Prometheus was validated step by step as a local bioinformatics development and moderate-scale workflow execution workstation.

The validation strategy was designed to confirm:

- hardware detection and system health
- storage and mount behaviour
- local NVMe work directory performance model
- cache routing away from the OS disk
- container execution
- Nextflow installation and behaviour
- nf-core workflow execution
- resume/caching behaviour
- integration with shared KCL RDS storage

The goal was not only to install tools, but to demonstrate that the machine behaves correctly under realistic pipeline workloads.

---

## Validation phases

### Phase 1: hardware and system validation

Prometheus was first validated at the hardware and OS level.

Confirmed successfully:

- correct CPU detection
- correct RAM size and channel population
- ECC memory detection
- all NVMe devices present
- HDD present
- NVIDIA GPU present and functioning
- expected motherboard and firmware identification
- network interfaces present, including Intel X550 10GbE adapters

### Confirmed hardware summary

- AMD Ryzen Threadripper PRO 5975WX
- 32 cores / 64 threads
- ASUS Pro WS WRX80E-SAGE SE WIFI
- 256 GB ECC DDR4 RDIMM
- NVIDIA RTX 5060 Ti 16 GB
- 1 × 1 TB Kioxia NVMe
- 3 × 4 TB Samsung 990 Pro NVMe
- 1 × 8 TB WD Red Pro HDD

---

### Phase 2: storage validation and final mount layout

Prometheus storage was then validated and reorganised into the final workstation layout.

Final validated layout:

- `/` on Kioxia 1 TB NVMe
- `/scratch1` on Samsung 990 Pro 4 TB
- `/scratch2` on Samsung 990 Pro 4 TB
- `/projects` on Samsung 990 Pro 4 TB
- `/archive` on WD Red Pro 8 TB
- `/rds/prj/bcn_whitema_rbp` on KCL networked RDS

Filesystems used:

- `xfs` for `/scratch1`
- `xfs` for `/scratch2`
- `ext4` for `/projects`
- `ext4` for `/archive`

This matched the intended split between:

- OS/system storage
- active work directory storage
- cache-heavy infrastructure storage
- retained local project outputs
- slower local overflow
- shared institutional network storage

---

### Phase 3: system tooling and monitoring validation

Prometheus was provisioned with core Linux and monitoring tools including:

- `nvme-cli`
- `smartmontools`
- `lm-sensors`
- `sysstat`
- `iotop`
- `iftop`
- `ethtool`
- `tmux`
- `tree`
- `pigz`
- `fio`
- `stress-ng`

Sensor and SMART inspection confirmed that:

- NVMe devices passed SMART health checks
- the HDD passed SMART health checks
- CPU temperature reporting worked
- DIMM and motherboard sensor visibility was available
- the system showed no major thermal or storage-health concerns during setup

---

## Container and workflow validation

### Singularity pull and runtime test

The first container validation step used a simple Singularity pull and run test with `hello-world`.

This confirmed:

- network image retrieval worked
- Singularity image conversion worked
- local container execution worked
- the container runtime was functioning correctly on Prometheus

This was the first end-to-end proof that container execution was operational.

---

### Nextflow installation and basic test

Nextflow was installed as a local user executable and validated using the standard `hello` workflow.

This confirmed:

- Java runtime support was correct
- Nextflow launched cleanly
- workflow pulling from GitHub worked
- the local executor worked
- Nextflow could execute simple tasks successfully

---

## nf-core workflow validation

### `nf-core/demo`

The next stage used the lightweight `nf-core/demo` pipeline to validate more realistic nf-core workflow behaviour without jumping directly to a larger RNA-seq test.

This validated:

- nf-core pipeline retrieval
- plugin download and resolution
- remote test input retrieval
- Singularity image pulls during workflow execution
- task execution on the local executor
- MultiQC output generation
- standard nf-core result structure
- resume behaviour

The first run also revealed an important practical issue:

- Singularity image caching was initially occurring inside the work directory rather than in the intended centralized cache location

This led to the key fix:

```bash
export NXF_SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity
```

After this correction, cache behaviour aligned with the intended design.

This was a useful validation result because it proved not just that the workflow could run, but that cache behaviour needed to be tuned for Prometheus.

---

### `nf-core/rnaseq`

The main workflow validation target was `nf-core/rnaseq`, as this is directly relevant to the intended use of Prometheus.

The validated command pattern used:

```bash
cd /projects/test_runs/nfcore_rnaseq_test

nextflow run nf-core/rnaseq \
  -profile test,singularity \
  --outdir /projects/test_runs/nfcore_rnaseq_test/results \
  -work-dir /scratch1/nextflow_work/nfcore_rnaseq_test
```

This test completed successfully.

#### Initial run result

- pipeline completed successfully
- duration: 21 minutes 47 seconds
- succeeded tasks: 219

#### Resume run result

- pipeline completed successfully
- duration: 1 minute 14 seconds
- cached: 208 tasks
- succeeded fresh tasks: 11
- cache reuse: 85.9%

This is a strong validation of Prometheus as a real local workflow execution machine.

It confirmed:

- nf-core/rnaseq runs correctly
- Singularity execution works under realistic load
- work directories are correctly separated from retained outputs
- centralized cache routing works
- resume behaviour is strong and useful for iterative development

---

## Observed storage behaviour during validation

The validation runs demonstrated that Prometheus now behaves as designed.

### `/scratch1`

Used for:

- active Nextflow work directories
- hashed task execution folders
- staged workflow inputs
- task-local intermediate files

Example validated usage:

- `/scratch1/nextflow_work/nfcore_demo`
- `/scratch1/nextflow_work/nfcore_rnaseq_test`

### `/scratch2`

Used for:

- Singularity container cache
- Nextflow framework and assets cache
- plugin downloads
- temporary files
- micromamba package cache

Example validated usage:

- `/scratch2/container_cache/singularity`
- `/scratch2/nextflow_cache`
- `/scratch2/tmp`
- `/scratch2/micromamba/pkgs`

### `/projects`

Used for:

- launch directories
- retained run folders
- final pipeline outputs
- pipeline information and reports

Example validated usage:

- `/projects/test_runs/nfcore_demo/results`
- `/projects/test_runs/nfcore_rnaseq_test/results`

This confirmed that the system is using local NVMe scratch for execution and `/projects` for retained human-facing outputs, as intended.

---

## Example measured disk usage after validation

A validated `nf-core/rnaseq` test showed:

- `/scratch1/nextflow_work/nfcore_rnaseq_test` → about 318M
- `/scratch2/container_cache/singularity` → about 12G
- `/scratch2/nextflow_cache` → about 209M
- `/projects/test_runs/nfcore_rnaseq_test/results` → about 81M

These values will vary over time and with additional pipeline runs, but they provide a good example of the intended distribution of workload and cache usage.

---

## RDS validation

Prometheus was also validated for persistent access to KCL Research Data Storage.

The RDS share was mounted successfully at:

```text
/rds/prj/bcn_whitema_rbp
```

This matched the CREATE HPC path style and resolved the earlier network/shared-storage blocker.

The RDS mount was validated as:

- persistent via `/etc/fstab`
- mounted read/write
- accessible across reboots
- reachable under the CREATE-consistent path
- suitable as a shared lab sync/archive destination

This completed the intended storage model for the workstation.

---

## Overall validation conclusion

Prometheus has now been validated as a:

- Linux-first bioinformatics workstation
- local Nextflow and nf-core development platform
- moderate-scale RNA-seq execution machine
- CREATE-aligned pre-production workflow environment
- local NVMe-backed execution layer with shared RDS integration

The validation process did not only confirm that individual tools were installed. It confirmed that the whole machine behaves correctly as a pipeline workstation under realistic conditions.

---

## Key takeaways

### What worked as intended

- hardware detection
- ECC and memory layout
- split storage layout
- local scratch execution
- centralized cache placement
- Singularity execution
- Nextflow installation
- nf-core workflow execution
- resume behaviour
- RDS integration

### What needed tuning

- explicit Singularity cache routing via `NXF_SINGULARITY_CACHEDIR`
- explicit `-work-dir` usage in documented run patterns

These adjustments improved clarity and produced the final validated execution model.

---

## Final practical verdict

Prometheus is now ready for:

- local pipeline development
- debugging
- moderate-scale real project execution
- staged validation before CREATE deployment
- repo-documented, reproducible workflow use within the lab
