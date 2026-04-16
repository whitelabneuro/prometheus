# Prometheus

Prometheus is the White Lab local bioinformatics workstation for Linux-first pipeline development, debugging, and moderate-scale execution prior to scaling heavier workloads to CREATE HPC.

It is designed to provide:

- fast local NVMe-backed execution for active workflows
- reproducible Nextflow and nf-core pipeline testing
- local development for short-read, long-read, and single-cell transcriptomics workflows
- clean alignment with CREATE HPC storage and execution patterns
- straightforward promotion of validated outputs to shared KCL Research Data Storage (RDS)

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

Prometheus is **not** intended to be the lab’s long-term archival storage system.

---

## Final storage model

Prometheus uses a split-storage model so that heavy execution, caches, retained outputs, and shared institutional storage remain clearly separated.

| Mount point | Role |
|---|---|
| `/` | Ubuntu OS, installed software, admin tools, user home |
| `/scratch1` | Primary high-speed Nextflow work directories |
| `/scratch2` | Cache-heavy space: container cache, Nextflow cache, temp, micromamba package cache |
| `/projects` | Structured local project folders and retained active outputs |
| `/archive` | Slower local overflow / holding space |
| `/rds/prj/bcn_whitema_rbp` | Persistent shared KCL Research Data Storage, aligned with CREATE |

This mirrors the CREATE HPC path style for RDS:

- CREATE: `/rds/prj/bcn_whitema_rbp`
- Prometheus: `/rds/prj/bcn_whitema_rbp`

---

## Standard execution model

The intended execution pattern on Prometheus is:

- keep heavy task execution on `/scratch1`
- keep container and framework caches on `/scratch2`
- keep organised retained outputs under `/projects`
- sync or promote appropriate outputs to `/rds/prj/bcn_whitema_rbp`

Standard local run pattern:

```bash
cd /projects/test_runs/<run_name>

nextflow run <pipeline> \
  -profile singularity \
  --outdir /projects/test_runs/<run_name>/results \
  -work-dir /scratch1/nextflow_work/<run_name>
