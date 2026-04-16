# Prometheus

Prometheus is the White Lab local bioinformatics workstation for Linux-first pipeline development, debugging, and moderate-scale execution prior to scaling heavier workloads to CREATE HPC.

It is designed to provide:

- fast local NVMe-backed execution for active workflows
- reproducible Nextflow and nf-core pipeline testing
- local development for short-read, long-read, and single-cell transcriptomics workflows
- clean alignment with CREATE HPC storage and execution patterns
- straightforward promotion of validated outputs to shared KCL Research Data Storage (RDS)

Prometheus is intended to complement CREATE HPC, not replace it.

---

## Intended role

Prometheus is intended for:

- local pipeline development and debugging
- moderate-scale local analysis runs
- nf-core and Nextflow workflow testing
- RNA-seq workflow development
- short-read RNA-seq
- long-read and isoform workflow development
- single-cell workflow prototyping
- validating storage, cache, and container behaviour before running larger jobs on CREATE HPC

Prometheus is **not** intended to be:

- the lab’s long-term archival storage system
- the default platform for very large production cohorts
- a replacement for scheduler-backed HPC for heavy parallel workloads

---

## Validated hardware summary

Prometheus was built and validated with the following core hardware configuration:

- AMD Ryzen Threadripper PRO 5975WX
- 32 cores / 64 threads
- ASUS Pro WS WRX80E-SAGE SE WIFI
- 256 GB ECC DDR4 RDIMM
- NVIDIA RTX 5060 Ti 16 GB
- 1 × 1 TB Kioxia NVMe SSD
- 3 × 4 TB Samsung 990 Pro NVMe SSD
- 1 × 8 TB WD Red Pro HDD
- Intel X550 dual 10GbE
- Intel AX200 Wi-Fi
- Ubuntu 24.04.4 LTS

This hardware profile was chosen to prioritise:

- strong local CPU throughput
- high NVMe scratch performance
- ECC memory stability
- practical local workflow execution
- fast iteration during development and debugging

---

## Final storage model

Prometheus uses a split-storage model so that operating system files, active workflow execution, caches, retained outputs, and shared institutional storage remain clearly separated.

| Mount point | Role |
|---|---|
| `/` | Ubuntu OS, installed software, admin tools, user home |
| `/scratch1` | Primary high-speed local Nextflow work directories |
| `/scratch2` | Cache-heavy space for containers, Nextflow cache, temp, and micromamba package cache |
| `/projects` | Organised local project folders and retained active outputs |
| `/archive` | Slower local holding or overflow space |
| `/rds/prj/bcn_whitema_rbp` | Persistent shared KCL Research Data Storage |

A key design goal is that the shared institutional storage path matches CREATE HPC:

- CREATE: `/rds/prj/bcn_whitema_rbp`
- Prometheus: `/rds/prj/bcn_whitema_rbp`

This keeps workstation and HPC path conventions aligned.

---

## Software model

Prometheus uses a layered software strategy:

- system packages via APT for core tooling
- micromamba for user-space package environments
- Nextflow as workflow engine
- Singularity CE as the primary workflow container runtime
- Docker retained as a secondary development/runtime option

Key configured locations include:

- `MAMBA_ROOT_PREFIX=/projects/micromamba`
- `CONDA_PKGS_DIRS=/scratch2/micromamba/pkgs`
- `NXF_HOME=/scratch2/nextflow_cache`
- `TMPDIR=/scratch2/tmp`
- `NXF_SINGULARITY_CACHEDIR=/scratch2/container_cache/singularity`

This keeps heavy workflow cache and execution pressure away from the OS disk.

---

## Standard execution model

Prometheus is intended to run workflows with:

- launch directories under `/projects`
- active work directories under `/scratch1`
- framework and container caches under `/scratch2`
- retained outputs under `/projects`
- shared copy or archive destinations under `/rds/prj/bcn_whitema_rbp`

The standard documented run pattern is:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

For nf-core test runs, the pattern becomes:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile test,singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
```

This is the default operational model for local workflow execution on Prometheus.

---

## Relationship to CREATE HPC

Prometheus and CREATE serve different but complementary roles.

### Prometheus is best for
- local workflow development
- debugging
- iterative reruns
- storage and output validation
- moderate-scale local execution
- rapid development without scheduler overhead

### CREATE is best for
- large cohort processing
- very heavy parallel execution
- long-running production workflows
- jobs that benefit from scheduler-managed cluster resources

A common workflow pattern is:

1. develop or validate locally on Prometheus
2. confirm storage, cache, and output behaviour
3. scale up on CREATE where appropriate
4. retain or promote shared outputs via RDS

---

## Validation status

Prometheus has been validated end to end for its intended role.

Validated successfully:

- hardware detection and system health
- split local storage layout
- persistent mount behaviour
- persistent KCL RDS mount
- Singularity container pull and execution
- Nextflow installation and execution
- `nextflow-io/hello`
- `nf-core/demo`
- `nf-core/rnaseq`
- strong resume behaviour with high cache reuse

Notable validated `nf-core/rnaseq` behaviour included:

- successful initial run
- successful resume run
- high reuse of cached tasks
- correct separation between work directories, cache locations, and retained outputs

This confirmed that Prometheus is not just installed, but functioning correctly as a real workflow workstation.

---

## RDS note

Prometheus mounts the KCL RDS share persistently at:

```text
/rds/prj/bcn_whitema_rbp
```

using a root-only SMB credentials file stored at:

```text
/root/.smb/rds_bcn_whitema_rbp.cred
```

If the KCL password changes, the `password=` entry in this credentials file must be updated manually, otherwise the persistent RDS mount will fail.

---

## Documentation map

Detailed documentation is provided in `docs/`:

- `01_overview.md` — high-level workstation purpose and design
- `02_hardware-validation.md` — hardware and system validation summary
- `03_storage-layout.md` — storage tiers, mount points, and usage rules
- `04_rds-mount.md` — persistent KCL RDS mount configuration and maintenance
- `05_software-stack.md` — software layering and tool strategy
- `06_nextflow-and-containers.md` — Nextflow, Singularity, cache, and execution setup
- `07_validation-runs.md` — validated test workflows and observed behaviour
- `08_operational-playbook.md` — day-to-day usage guidance and routine operations

---

## Practical summary

Prometheus should be thought of as:

- the lab’s local Nextflow and nf-core development workstation
- a high-speed NVMe-backed execution machine
- a staging and debugging platform before CREATE HPC
- a structured local analysis environment with shared RDS access

Its value comes from disciplined storage separation, reproducible workflow tooling, and practical alignment with the way the lab already works on CREATE.
