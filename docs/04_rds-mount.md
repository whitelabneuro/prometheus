# KCL RDS mount on Prometheus

## Overview

Prometheus mounts the White Lab KCL Research Data Storage (RDS) share so that the shared institutional storage path matches the path style used on CREATE HPC.

This gives a consistent location on both systems:

- CREATE: `/rds/prj/bcn_whitema_rbp`
- Prometheus: `/rds/prj/bcn_whitema_rbp`

This consistency makes it easier to:

- reuse path conventions across systems
- document workflows once
- stage or sync outputs between local and HPC environments
- treat RDS as the canonical shared lab storage destination

---

## RDS share details

### Remote share path

```text
//rds.er.kcl.ac.uk/prj/bcn_whitema_rbp
