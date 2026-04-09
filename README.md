
Whether you're pulling data off a NAS, sharing storage between VMs, or giving an LXC container access to a central media folder, mounting network shares is a fundamental skill in any homelab. This guide covers three scenarios: mounting a network share on the **Proxmox host itself**, inside an **LXC container**, and inside a **Linux virtual machine**.

We'll be using **NFS** and **SMB/CIFS** — the two most common share protocols you'll encounter. NFS is native to Linux and generally faster for local network use. SMB (Samba/CIFS) is what Windows and most NAS devices use by default, and is great for cross-platform shares.

---

## Prerequisites

Before you start, you'll need:

- A network share already set up and accessible on your LAN (e.g. from a QNAP, TrueNAS, Asustor, or Samba server)
- The IP address or hostname of your NAS/share server
- The share path (e.g. `/volume1/data` for NFS, or `\\nas-ip\sharename` for SMB)
- Credentials if using SMB with authentication

---

## 1. Mounting a Network Share on the Proxmox Host

Mounting directly on the Proxmox host is useful when you want to make storage available to multiple VMs or containers via the Proxmox storage backend, or if you're running something directly on the host (though that's generally not recommended for most workloads).

### Option A: NFS

**Install the NFS client:**

```bash
apt update && apt install -y nfs-common
```

**Create a mount point:**

```bash
mkdir -p /mnt/nas-nfs
```

**Test the mount manually:**

```bash
mount -t nfs 192.168.1.100:/volume1/data /mnt/nas-nfs
```

Replace `192.168.1.100` with your NAS IP and `/volume1/data` with your actual export path.

**Verify it worked:**

```bash
df -h | grep nas-nfs
ls /mnt/nas-nfs
```

**Make it persistent via `/etc/fstab`:**

```
192.168.1.100:/volume1/data  /mnt/nas-nfs  nfs  defaults,_netdev  0  0
```

> The `_netdev` option tells the system to wait for the network before mounting — important on boot.

**Apply without rebooting:**

```bash
mount -a
```

---

### Option B: SMB/CIFS

**Install the CIFS utilities:**

```bash
apt update && apt install -y cifs-utils
```

**Create a mount point:**

```bash
mkdir -p /mnt/nas-smb
```

**Store credentials securely:**

```bash
nano /etc/.smbCreds
```

Add the following:

```
username=your_smb_user
password=your_smb_password
```

Lock down the file:

```bash
chmod 600 /etc/.smbCreds
```

**Test the mount manually:**

```bash
mount -t cifs //192.168.1.100/sharename /mnt/nas-smb -o credentials=/etc/.smbCreds
```

**Make it persistent via `/etc/fstab`:**

```
//192.168.1.100/sharename  /mnt/nas-smb  cifs  credentials=/etc/.smbCreds,_netdev,uid=0,gid=0  0  0
```

**Apply without rebooting:**

```bash
mount -a
```

---

### Adding the Share to Proxmox Storage

Once mounted on the host, you can expose the share as a Proxmox storage backend via the UI:

1. Go to **Datacenter → Storage → Add → Directory**
2. Set the **Directory** path to your mount point (e.g. `/mnt/nas-nfs`)
3. Choose what content types to allow (ISOs, backups, disk images, etc.)
4. Click **Add**

> Proxmox also has built-in NFS and SMB/CIFS storage backends under **Datacenter → Storage → Add → NFS / SMB/CIFS** which handle the mount for you — useful if you want Proxmox itself to manage the connection.

---

## 2. Mounting a Network Share in an LXC Container

LXC containers share the host kernel, so the recommended approach is to mount the share **on the Proxmox host** first, then **bind mount it into the container**. This avoids installing NFS/SMB clients inside every container and keeps things centralised.

### Step 1: Mount the Share on the Host

Follow the NFS or SMB steps from Section 1 above to mount the share on the host (e.g. at `/mnt/nas-nfs`).

### Step 2: Bind Mount into the LXC Container

With the container stopped, edit its config file. LXC configs live at:

```
/etc/pve/lxc/<CTID>.conf
```

Add a bind mount line:

```
mp0: /mnt/nas-nfs,mp=/mnt/data
```

- `/mnt/nas-nfs` — the path **on the host**
- `/mnt/data` — the path **inside the container** (will be created automatically)
- `mp0`, `mp1`, `mp2`... — increment the index for each additional mount point

**For unprivileged containers**, you'll need to handle UID/GID mapping. The host's `root` (UID 0) is mapped to UID 100000 inside the container. This means files owned by `root` on the NAS may appear as `nobody` inside the container, or vice versa.

The easiest fix for NFS shares is to set the NFS export's `all_squash` + `anonuid`/`anongid` options on the NAS side to map everything to a consistent UID. For SMB, use the `uid` and `gid` mount options.

Alternatively, use a **privileged container** if you don't need the extra isolation (useful for homelab/trusted environments):

In the container config, set:

```
unprivileged: 0
```

> ⚠️ Privileged containers run as root on the host — fine for a homelab, but be aware of the security trade-off.

**Start the container and verify:**

```bash
pct start <CTID>
pct enter <CTID>
ls /mnt/data
```

### Alternative: Mount Inside the Container Directly

If you prefer not to bind mount from the host, you can install NFS/SMB clients directly inside the container and mount from within — same process as Section 1. Just note that **NFS inside unprivileged LXC** requires some extra steps:

For NFS in an unprivileged container, you need to enable the `nfs` feature flag in the container config:

```
features: nfs=1
```

And ensure `rpcbind` and `nfs-common` are installed inside the container.

---

## 3. Mounting a Network Share in a Linux Virtual Machine

VMs are fully isolated machines with their own kernel, so there's no bind-mount trick here — you install the client tools **inside the VM** and mount directly, just like you would on any Linux machine.

### Option A: NFS

**Inside the VM, install NFS client:**

```bash
# Debian/Ubuntu
apt update && apt install -y nfs-common

# RHEL/Rocky/AlmaLinux
dnf install -y nfs-utils
```

**Create mount point:**

```bash
mkdir -p /mnt/nas-nfs
```

**Test mount:**

```bash
mount -t nfs 192.168.1.100:/volume1/data /mnt/nas-nfs
```

**Persist in `/etc/fstab`:**

```
192.168.1.100:/volume1/data  /mnt/nas-nfs  nfs  defaults,_netdev  0  0
```

---

### Option B: SMB/CIFS

**Inside the VM, install CIFS utilities:**

```bash
# Debian/Ubuntu
apt update && apt install -y cifs-utils

# RHEL/Rocky/AlmaLinux
dnf install -y cifs-utils
```

**Create credentials file:**

```bash
nano /etc/samba/credentials
```

```
username=your_smb_user
password=your_smb_password
```

```bash
chmod 600 /etc/samba/credentials
```

**Create mount point:**

```bash
mkdir -p /mnt/nas-smb
```

**Test mount:**

```bash
mount -t cifs //192.168.1.100/sharename /mnt/nas-smb -o credentials=/etc/samba/credentials
```

**Persist in `/etc/fstab`:**

```
//192.168.1.100/sharename  /mnt/nas-smb  cifs  credentials=/etc/samba/credentials,_netdev,uid=1000,gid=1000  0  0
```

> Adjust `uid` and `gid` to match the user inside the VM that needs access (find them with `id your_username`).

---

## Troubleshooting Tips

| Problem | Likely Cause | Fix |
|---|---|---|
| `mount: no such file or directory` | Mount point doesn't exist | `mkdir -p /your/mountpoint` |
| `mount: access denied` | Wrong credentials or NFS export restriction | Check NAS export settings; verify IP is allowed |
| `mount: connection timed out` | Firewall or wrong IP/hostname | Ping the NAS; check NAS firewall rules |
| Files show as `nobody:nogroup` | UID/GID mismatch (common in LXC) | Use `all_squash` on NFS export or explicit `uid=/gid=` mount options |
| Share doesn't mount on boot | Missing `_netdev` in fstab | Add `_netdev` to mount options |
| LXC bind mount fails | Share not mounted on host yet | Ensure host fstab entry exists and `mount -a` has been run |

---

## Quick Reference

| Scenario | Protocol | Key Package | Mount Type |
|---|---|---|---|
| Proxmox Host | NFS | `nfs-common` | Direct |
| Proxmox Host | SMB | `cifs-utils` | Direct |
| LXC Container | NFS or SMB | (on host) | Bind mount from host |
| LXC Container | NFS (direct) | `nfs-common` + `features: nfs=1` | Direct (unprivileged needs flag) |
| Linux VM | NFS | `nfs-common` / `nfs-utils` | Direct |
| Linux VM | SMB | `cifs-utils` | Direct |
