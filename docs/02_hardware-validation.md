# Hardware validation on Prometheus

## Overview

Prometheus was validated first at the hardware and operating system level before any serious workflow execution or storage redesign was attempted.

The aim of this stage was to confirm that:

- the workstation delivered the expected hardware configuration
- the operating system could see all major components correctly
- storage devices were present and identifiable
- the GPU and driver stack were functional
- network interfaces were present
- firmware and BIOS information matched expectations
- the machine started from a known-good baseline before moving into storage, software, and pipeline setup

This was an important first step because all later workflow and storage decisions depend on the underlying hardware being present and healthy.

---

## Host and operating system

Prometheus was identified as running:

- Ubuntu 24.04.4 LTS
- Linux kernel 6.17.0-19-generic

Motherboard and firmware were reported as:

- ASUS Pro WS WRX80E-SAGE SE WIFI
- BIOS / firmware version 1801
- firmware date 2025-10-16

The hostname was later changed from the original desktop-style default to:

```text
prometheus
```

This was done to provide clearer prompts, logs, and machine identity during administration and workflow execution.

---

## CPU validation

The processor was correctly detected as:

- AMD Ryzen Threadripper PRO 5975WX

Validated CPU properties included:

- 32 physical cores
- 64 threads
- single socket
- boost enabled

This matched the expected final build specification.

The CPU topology and thread count were therefore confirmed to be correct for workflow execution, parallel local testing, and moderate-scale production-style runs.

---

## Memory validation

Memory detection confirmed the expected large ECC configuration.

Validated memory properties included:

- approximately 256 GB total installed memory
- all 8 memory channels populated
- ECC-capable RDIMM configuration
- operating speed at 3200 MT/s
- Micron DDR4 registered ECC DIMMs

The system reported multi-bit ECC support, and DMI inspection confirmed that all expected DIMM positions were populated.

This was important because the final workstation design deliberately prioritized a stable 256 GB ECC configuration over chasing a larger but harder-to-source 512 GB configuration.

The validated result confirmed that the intended memory design was correctly delivered.

---

## GPU validation

The NVIDIA GPU and driver stack were validated successfully.

Detected GPU:

- NVIDIA GeForce RTX 5060 Ti
- 16 GB VRAM

Driver and CUDA status:

- `nvidia-smi` working
- NVIDIA driver loaded correctly
- CUDA visible

This confirmed that the GPU was correctly recognized by the operating system and that the NVIDIA user-space tooling was functional.

The GPU is not central to the primary RNA-seq pipeline use case for Prometheus, but correct driver function is still important for future flexibility and general workstation stability.

---

## Storage device validation

All expected storage devices were detected successfully.

### OS disk
- 1 × 1 TB Kioxia NVMe SSD

### High-speed local NVMe devices
- 3 × 4 TB Samsung 990 Pro NVMe SSDs

### Bulk local storage
- 1 × 8 TB WD Red Pro HDD

This matched the intended final storage build.

The OS disk was confirmed to be separate from the three larger workflow-oriented NVMe drives, which was essential for the planned split between:

- operating system and software
- active work directories
- cache-heavy storage
- retained local project outputs

---

## Network interface validation

Prometheus was validated to contain the expected network interfaces.

Detected interfaces included:

- dual Intel X550 10GbE Ethernet adapters
- Intel Wi-Fi 6 AX200

During early setup, the workstation was initially using Wi-Fi while wired networking and later RDS access were being brought online.

This was sufficient for early package installation and test workflow execution, but some first-time container pulls were noticeably slower over Wi-Fi, as expected.

Later setup work successfully enabled access to KCL RDS, confirming that network-dependent storage access was functioning correctly.

---

## Storage health and monitoring validation

After core tooling installation, storage and hardware health checks were performed using:

- `smartmontools`
- `nvme-cli`
- `lm-sensors`

### SMART validation

All NVMe drives reported healthy SMART status.

The 8 TB HDD also reported healthy overall SMART status.

This confirmed that the storage devices were visible, functioning, and not showing immediate health issues during initial deployment.

### Thermal validation

Temperature monitoring confirmed:

- reasonable CPU temperatures
- reasonable NVMe temperatures
- visible DIMM temperature reporting
- visible motherboard sensor reporting

A few motherboard sensor channels reported implausible auxiliary values, which is common on workstation/server boards where some channels are unmapped or poorly labeled.

The readings considered trustworthy and useful included:

- CPU temperatures
- NVMe temperatures
- DIMM temperatures
- main system board temperatures

These all appeared acceptable during setup.

---

## Initial disk and partition state

The first disk audit confirmed:

- Ubuntu installed on the Kioxia 1 TB NVMe
- the Samsung 4 TB drives present and ready for reassignment
- the HDD present and formatted
- no hardware-level disk detection issues

This allowed the storage redesign to proceed confidently into the final mount layout:

- `/scratch1`
- `/scratch2`
- `/projects`
- `/archive`

---

## Container and software prerequisites at hardware stage

Several key checks at this stage also confirmed that the machine was suitable for later workflow work:

- Java could be installed cleanly
- Docker was already present and functional
- NVIDIA tooling was healthy
- local NVMe storage was available for high-speed work and cache placement

This meant there were no hardware or firmware blockers to continuing with:

- filesystem setup
- software installation
- Singularity setup
- Nextflow validation
- nf-core pipeline validation

---

## Why this hardware validation mattered

The workstation was intended to serve as:

- a local bioinformatics development platform
- a moderate-scale RNA-seq execution workstation
- a staging and debugging layer before CREATE HPC
- a Linux-first, reproducible workflow environment

To trust that role, it was necessary to validate that:

- the CPU and RAM actually matched the purchased configuration
- ECC memory was correctly recognized
- all storage tiers were present
- the operating system had healthy access to GPU and networking
- monitoring tools could inspect the system meaningfully

This stage provided that foundation.

---

## Validated hardware summary

Prometheus was validated with the following final hardware state:

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

---

## Practical conclusion

The hardware validation phase confirmed that Prometheus was delivered and detected as expected, with no major hardware surprises or blockers.

This provided the confidence needed to proceed with:

- filesystem redesign
- software stack installation
- Nextflow and Singularity setup
- nf-core workflow validation
- RDS integration
- documentation of the final workstation model
