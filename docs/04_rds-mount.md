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
```

### Local mount point

```text
/rds/prj/bcn_whitema_rbp
```

---

## Package requirement

Prometheus uses a CIFS/SMB mount for the RDS share.

Install the required package with:

```bash
sudo apt install cifs-utils
```

---

## Mount point creation

The local mount path must exist before mounting:

```bash
sudo mkdir -p /rds/prj/bcn_whitema_rbp
```

---

## Credentials handling

### Important principle

The KCL password is **not** embedded directly in `/etc/fstab`.

Instead, the mount uses a protected root-only SMB credentials file stored at:

```text
/root/.smb/rds_bcn_whitema_rbp.cred
```

### File contents

The credentials file contains:

```ini
username=k1643702
password=<current KCL password>
domain=kclad
```

### Permissions

The credentials file must be locked down:

```bash
sudo chown root:root /root/.smb/rds_bcn_whitema_rbp.cred
sudo chmod 600 /root/.smb/rds_bcn_whitema_rbp.cred
```

This prevents accidental exposure of institutional credentials.

---

## Critical maintenance note

If the KCL account password changes, the credentials file on Prometheus **must also be updated manually**.

Otherwise the persistent RDS mount will fail.

Update the file with:

```bash
sudo nano /root/.smb/rds_bcn_whitema_rbp.cred
```

Then edit the `password=` line to the current KCL password.

This is one of the most important operational maintenance points for the RDS mount and should be clearly documented for future maintenance.

---

## Persistent mount configuration

The RDS share is mounted persistently through `/etc/fstab`.

The final working setup uses the remote share:

```text
//rds.er.kcl.ac.uk/prj/bcn_whitema_rbp
```

mounted to:

```text
/rds/prj/bcn_whitema_rbp
```

using the credentials file above and local user mapping.

### Operationally important options

The working mount uses the following behaviour:

- read/write mount
- local `uid=1000`
- local `gid=1000`
- `file_mode=0660`
- `dir_mode=0770`

This ensures the mounted share behaves sensibly for the main local user account while keeping permissions reasonably restrictive.

---

## Example `/etc/fstab` entry

```fstab
//rds.er.kcl.ac.uk/prj/bcn_whitema_rbp /rds/prj/bcn_whitema_rbp cifs credentials=/root/.smb/rds_bcn_whitema_rbp.cred,uid=1000,gid=1000,file_mode=0660,dir_mode=0770,rw,nofail 0 0
```

If site-specific tuning is needed later, it should be adjusted in this line rather than relying on ad hoc manual mounts.

---

## Reloading and testing the mount

After updating `/etc/fstab`, reload systemd and test the mount cleanly:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

If successful, verify with:

```bash
mount | grep bcn_whitema_rbp
df -h | grep bcn_whitema_rbp
ls /rds/prj/bcn_whitema_rbp
```

---

## Verified working state

The mount has been verified as:

- persistent across reboots
- mounted read/write
- visible at `/rds/prj/bcn_whitema_rbp`
- showing the expected shared storage contents
- consistent with CREATE path conventions

A validated example state showed:

- about 10T total
- about 5.7T used
- about 4.4T free

These figures will naturally change over time.

---

## Role of RDS in the Prometheus storage model

Prometheus should use RDS as:

- canonical shared lab storage
- sync destination for validated outputs
- shared project handoff location
- CREATE-aligned project storage
- destination for datasets or outputs that should not remain only on local disks

RDS should **not** be treated as a replacement for fast local NVMe execution space.

---

## Recommended workflow pattern

### Local execution

Use local NVMe for active execution:

- `/scratch1` for `-work-dir`
- `/scratch2` for container and framework cache
- `/projects` for organised local outputs

### Shared destination

Use RDS for promotion or sync of retained results:

```bash
rsync -avh --progress /projects/test_runs/<run_name>/results/ \
  /rds/prj/bcn_whitema_rbp/<destination_path>/
```

This keeps local execution fast while ensuring important outputs are copied to shared institutional storage.

---

## Troubleshooting

### Mount fails after password change

Most likely cause:
- `/root/.smb/rds_bcn_whitema_rbp.cred` contains an outdated password

Fix:
- update the `password=` line
- rerun `sudo mount -a`

### Mount point exists but share is not mounted

Check:

```bash
mount | grep bcn_whitema_rbp
```

If not mounted:
- inspect `/etc/fstab`
- test with `sudo mount -a`

### Permission behaviour looks wrong

Check that the `/etc/fstab` entry still contains:
- `uid=1000`
- `gid=1000`
- `file_mode=0660`
- `dir_mode=0770`

### Network-dependent failure

Because RDS is network-backed, the mount depends on:
- valid network access
- valid DNS resolution
- a reachable KCL service endpoint
- valid credentials

---

## Key operational note to preserve

Prometheus mounts the KCL RDS share persistently at `/rds/prj/bcn_whitema_rbp` using a root-only SMB credentials file stored at `/root/.smb/rds_bcn_whitema_rbp.cred`. If the KCL password changes, the `password=` entry in this credentials file must be updated manually, otherwise the RDS mount will fail.
