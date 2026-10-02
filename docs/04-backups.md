# 04 – Backups

*Date: 2 October 2026*

With Keycloak configured and the first SSO integration working, the lab now holds work I do not want to lose. This part sets up scheduled backups of the Docker container and verifies that a backup can actually be restored.

---

## 1. Backup storage

For now, backups are stored on the Proxmox host itself, on the `local` directory storage.

*Datacenter → Storage → local → Edit*: added **Backup** to the allowed content types.

**Next step – dedicated backup disk.** A separate disk will be formatted and added to Proxmox as backup storage, so backups no longer live on the same SSD as the container they protect. The backup job will then be pointed at that storage.

---

## 2. Backup job

*Datacenter → Backup → Add*

| Setting | Value | Why |
|---|---|---|
| Storage | `local` | Until the dedicated backup disk is in place |
| Schedule | `sun 02:00` | Weekly, outside working hours |
| Selection mode | All | New containers and VMs are included automatically |
| Mode | Snapshot | The container keeps running during the backup |
| Compression | ZSTD | Fast with a good compression ratio |
| Retention | Keep last 2 | Limits disk usage on `local` |

**Result of the first runs** (started manually with *Run now*):

| Item | Value |
|---|---|
| Duration | about 75 seconds |
| Backup size | 4.44 GiB (compressed) |
| Data in the container | 9.0 GiB |

---

## 3. Restore test

A backup that has never been restored is an assumption, not a backup. So the first backup was restored straight away.

1. *local (pve) → Backups* → selected the latest `vzdump-lxc-110-…tar.zst` → **Restore**.
2. Restored to a **new container ID (999)** on `local-lvm`, with *Start after restore* disabled.
3. The task finished with **TASK OK**: 9.0 GiB restored at about 335 MiB/s.
4. Removed container 999 again without starting it.

**Why a new ID:** restoring to ID 110 would overwrite the running container.

**Why not start it:** the restored container has the same IP address (`192.168.178.11`) as the original. Starting both would cause an address conflict on the network.

---

## What a backup contains

The backup covers the **entire LXC container**, including all Docker volumes:

- Keycloak's PostgreSQL database (realms, users, clients, MFA settings)
- Portainer's configuration, including the OAuth settings and the client secret
- All Docker images and the compose stacks

Restoring it brings back the whole environment in one step, without reconfiguring Keycloak or the SSO integration.

---

## Lessons

- **Test the restore, not just the backup.** A green *TASK OK* on the backup job says nothing about whether the result is usable.
- **Never restore over the original while testing.** Use a new ID and keep the copy switched off.
- **Retention is part of the design.** With 4.44 GiB per backup, an unlimited history would fill the storage; *keep-last* prevents that.
- **A backup contains secrets.** Passwords, the database and client secrets are all in the archive, so backup storage needs the same protection as the system itself.

## Next steps

- Format a dedicated backup disk and add it to Proxmox as backup storage; move the backup job to it.
- Database-level backups of Keycloak with `pg_dump`, in addition to the container backup.
- Later: Proxmox Backup Server on the second mini PC (deduplication, verification, encryption).
