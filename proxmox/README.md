# Proxmox NAS — MergerFS + SnapRAID

NAS configuration for the N100 mini-ITX running Proxmox VE. Two data disks are pooled with MergerFS and protected by SnapRAID parity on a third disk. Samba exposes everything to macOS, Windows, and Linux clients.

## Disk Layout

| Slot | Drive | Size | Role | Mount |
|------|-------|------|------|-------|
| NVMe 0 | WD Red SN700 | 500 GB | Proxmox OS (do not touch) | — |
| NVMe 1 | WD Red SN700 | 500 GB | Active projects | `/srv/projects` |
| SATA A | Samsung 870 EVO | 1 TB | Archive / general data | `/srv/archive1` |
| SATA B | Samsung 870 EVO | 1 TB | SnapRAID parity | `/srv/parity1` |

The MergerFS pool at `/mnt/storage` spans `/srv/projects` + `/srv/archive1`. Parity is never part of the pool.

## Files

| File | Installed to | Purpose |
|------|-------------|---------|
| `mergerfs.fstab` | append to `/etc/fstab` | Mount entries for data disks, parity, and the MergerFS pool |
| `snapraid.conf` | `/etc/snapraid.conf` | SnapRAID parity, content, data disk, and exclusion rules |
| `smb.conf` | `/etc/samba/smb.conf` | Samba global settings and share definitions |
| `crontab` | `crontab -e` (root) | Nightly SnapRAID sync + weekly scrub |

## Quick Start

### 1. Install packages

```bash
apt update
apt install -y mergerfs snapraid samba smartmontools
systemctl enable --now fstrim.timer
sed -i 's/^#user_allow_other/user_allow_other/' /etc/fuse.conf
```

### 2. Partition and format

Each disk gets a single GPT partition formatted ext4. Replace the placeholder serials with actual `/dev/disk/by-id` values.

```bash
# Projects (NVMe)
parted -s /dev/nvme1n1 mklabel gpt
parted -s /dev/nvme1n1 mkpart primary ext4 0% 100%
mkfs.ext4 -L projects /dev/disk/by-id/<NVME_DATA_SERIAL>-part1

# Archive (SATA SSD)
parted -s /dev/sda mklabel gpt
parted -s /dev/sda mkpart primary ext4 0% 100%
mkfs.ext4 -L archive1 /dev/disk/by-id/<SSD_DATA_SERIAL>-part1

# Parity (SATA SSD)
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary ext4 0% 100%
mkfs.ext4 -L parity1 /dev/disk/by-id/<SSD_PARITY_SERIAL>-part1
```

### 3. Mount

```bash
mkdir -p /srv/projects /srv/archive1 /srv/parity1 /mnt/storage
```

Copy the entries from `mergerfs.fstab` into `/etc/fstab`, then:

```bash
mount -a
df -h | egrep 'projects|archive1|parity1|storage'
```

### 4. SnapRAID

Copy `snapraid.conf` to `/etc/snapraid.conf`, then run the initial sync:

```bash
snapraid touch
snapraid sync
snapraid status
```

Install the cron jobs from `crontab` via `crontab -e` as root.

### 5. Samba

Copy `smb.conf` to `/etc/samba/smb.conf`. Create the share user:

```bash
adduser <USERNAME>
smbpasswd -a <USERNAME>
chown -R <USERNAME>:<USERNAME> /srv/projects /srv/archive1
testparm -s
systemctl restart smbd
```

Connect from clients at `smb://<proxmox-ip>/storage` (or `/projects`, `/archive1`).

## Adding More Disks

1. Partition, format, and mount the new disk under `/srv/<name>`.
2. Add the mount to `/etc/fstab`.
3. Append the path to the MergerFS line in fstab (e.g., `/srv/projects:/srv/archive1:/srv/<name>`).
4. Add a `data <name> /srv/<name>/` line to `snapraid.conf`.
5. `mount -a && snapraid touch && snapraid sync`.

## Maintenance

```bash
smartctl -a /dev/sda              # SMART health
systemctl status fstrim.timer     # weekly TRIM
df -h /mnt/storage                # pool usage
snapraid status                   # parity health
snapraid scrub -p 12 -o 7        # manual scrub
```

## Placeholders

These configs use `<PLACEHOLDER>` tokens so no real hardware serials or usernames are committed. Replace before deploying:

| Placeholder | Where to find it |
|-------------|-----------------|
| `<NVME_DATA_SERIAL>` | `ls -l /dev/disk/by-id/ \| grep nvme` |
| `<SSD_DATA_SERIAL>` | `ls -l /dev/disk/by-id/ \| grep ata` |
| `<SSD_PARITY_SERIAL>` | `ls -l /dev/disk/by-id/ \| grep ata` |
| `<HOSTNAME>` | `hostname` |
| `<USERNAME>` | your NAS user account |
