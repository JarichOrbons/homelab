# 01 – Proxmox VE and Docker

*Date: 29–30 September 2026*

In this part I install Proxmox VE on mini PC 1, set up a Docker host in an LXC container and run the first services: Portainer and Keycloak.

---

## 1. Installing Proxmox VE

**Starting point:** a mini PC with a clean Windows 11 installation (Hyper-V enabled but unused) and a 512 GB SSD. The Proxmox installation wipes the entire disk.

**Preparation**
- Network cable connected (Proxmox does not support Wi-Fi during installation).
- In the BIOS: virtualisation (VT-x) and VT-d/IOMMU enabled.
- Booted from a USB stick with Proxmox VE 9.2.

**Installation choices**

| Setting | Value | Why |
|---|---|---|
| File system | ext4 (LVM-thin) | Single disk; lighter on RAM than ZFS |
| Hostname | `pve.home.lan` | Fully qualified domain name (FQDN) |
| IP address | `192.168.178.10/24` (static) | Outside the router's DHCP range |
| Gateway / DNS | `192.168.178.1` | Router |

**After installation:** management via `https://192.168.178.10:8006`.

### Problem 1 – Server unreachable after installation

**Symptom:** `ERR_CONNECTION_TIMED_OUT` in the browser. From the server, `ping 192.168.178.1` returned *Destination Host Unreachable*.

**Diagnosis:** `ip a` showed that the network bridge `vmbr0` had the address `192.168.179.10/24`: a typo (`179` instead of `178`). As a result, the server was in a different subnet than the router.

**Attempted fix:** corrected the address in `/etc/network/interfaces`. In the process, the lines under `vmbr0` were accidentally merged onto one line, so the bridge was no longer created at all.

Correct configuration for reference:

```
auto vmbr0
iface vmbr0 inet static
        address 192.168.178.10/24
        gateway 192.168.178.1
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
```

**Solution:** reinstalled Proxmox with the correct settings. During the first attempt the installer did not receive an IPv4 address from the router (falling back to `192.168.100.2` with an IPv6 gateway); during the second installation DHCP did work, and I replaced the suggested address with the static address `.10`.

**Lesson:** check network settings before clicking *Install*, and edit configuration files carefully; a single merged line breaks the entire network.

### Problem 2 – Updates fail (401 Unauthorized)

**Symptom:** `apt-get update` failed with `401 Unauthorized` on `enterprise.proxmox.com`.

**Cause:** Proxmox is configured by default to use the enterprise repositories, which require a paid subscription.

**Solution:** under *pve → Updates → Repositories*:
1. Disabled `pve-enterprise` and the Ceph enterprise repository.
2. Added the **No-Subscription** repository.
3. *Refresh* (TASK OK) and *Upgrade* → Proxmox VE 9.2.21.

---

## 2. Docker host in an LXC container

**Choice:** Docker in an unprivileged LXC instead of a VM. Proxmox officially recommends a VM for Docker because of better isolation; for this lab I chose an LXC because it is much more efficient with RAM and disk space.

| Setting | Value |
|---|---|
| CT ID / hostname | `110` / `docker` |
| Template | `debian-13-standard` |
| Disk | `local-lvm` (enlarged later) |
| CPU / RAM | 2 cores / 4 GB (increased later) |
| Network | `192.168.178.11/24`, gateway `192.168.178.1` |
| Features | `nesting=1`, `keyctl=1` (required for Docker in an unprivileged LXC) |
| Options | Start at boot enabled |

**Installing Docker:**

```bash
apt update && apt full-upgrade -y
apt install -y curl
curl -fsSL https://get.docker.com | sh
docker run hello-world
```

---

## 3. Portainer

```bash
docker volume create portainer_data
docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:lts
```

### Problem 3 – Setup token and timeout

**Symptom:** during the first setup, Portainer asked for a *setup token* and locked itself after a few minutes (*timed out for security purposes*). A first token returned `403`.

**Cause:**
- The setup token is a security measure: only someone with access to the server can create the first administrator account.
- After a restart, `docker logs` also shows the log lines of **previous** runs, including old, invalid tokens.

**Solution:**

```bash
docker restart portainer
docker logs portainer 2>&1 | grep setup_token= | tail -n 1
```

Then immediately reloaded the page and created the account. Edge Compute was deliberately **not** enabled: it is meant for managing remote Docker hosts and opens an extra port that I do not need.

**Lesson:** pasting into the Proxmox web console can add stray characters (`^[[200~`). Via SSH (`ssh root@192.168.178.10` followed by `pct enter 110`) copying and pasting works more reliably.

---

## 4. Keycloak with PostgreSQL

See [`docker/keycloak/docker-compose.yml`](../docker/keycloak/docker-compose.yml). Deployed as a stack in Portainer; passwords are set as environment variables in Portainer, not in the compose file.

- Keycloak currently runs in **dev mode** (`start-dev`, HTTP on port 8080).
- Production mode with HTTPS will follow once the reverse proxy and certificates are in place.

### Problem 4 – Full disk and a corrupted image

**Symptom:**
1. Keycloak was unreachable (`ERR_CONNECTION_REFUSED`) and `keycloak-db` was stuck in *restarting*.
2. Portainer reported `no space left on device`.
3. After freeing up space, Keycloak refused to start: `exec: "/opt/keycloak/bin/kc.sh": no such file or directory`.

**Cause:** the LXC disk was full due to an earlier test with large images. The Keycloak image had been pulled while the disk was filling up and was therefore incomplete.

**Solution:**
1. Removed the test environment and cleaned up unused volumes and images with `docker image prune -a`.
2. Enlarged the root disk of LXC 110 via *Resources → Root Disk → Volume Action → Resize*.
3. Removed the corrupted image and pulled it again:

```bash
docker rm -f keycloak
docker rmi quay.io/keycloak/keycloak:latest
docker pull quay.io/keycloak/keycloak:latest
```

4. Redeployed the stack in Portainer.

**Lessons:**
- An error message does not always point to the real cause. "File does not exist" meant "the image was only partially pulled because the disk was full".
- Monitor disk usage with `df -h /` and `docker system df`, and plan enough disk space before deploying new stacks.

---

## Result

| Service | Address | Status |
|---|---|---|
| Proxmox VE | `https://192.168.178.10:8006` | ✅ |
| Portainer | `https://192.168.178.11:9443` | ✅ |
| Keycloak | `http://192.168.178.11:8080` | ✅ (dev mode) |

## Next steps

- Configure Keycloak: replace the temporary bootstrap admin with a permanent administrator account, create the `homelab` realm and a user with MFA (OTP).
- Snapshot of LXC 110.
- Reverse proxy with its own CA and TLS certificates.
