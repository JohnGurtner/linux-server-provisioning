# Linux Server Provisioning

> Built a three-OS server environment from scratch on Nutanix (Windows Server 2022, Rocky Linux 10.2 and
> Ubuntu Server 24.04), then set up each Linux host the same way: custom partitioning, static networking,
> guest tools, enforced password complexity, per-user disk quotas and network printing.

*CNIT 340 (Unix/Linux System Administration), Lab 1, Group 063.*

## Planning

| Host | OS | vRAM / Disk | Subnet | Address |
| --- | --- | --- | --- | --- |
| `CNIT340.G063.Gurtner.WS` | Windows Server 2022 Standard (Desktop Experience) | 8 GB / 48 GB | CNIT340WL Windows | 44.101.0.0/23 range, gateway 44.101.0.1 |
| `CNIT340.G063.Gurtner.RL` | Rocky Linux 10.2 (Minimal) | 4 GB / 60 GB | CNIT340WL Linux A | 44.101.2.63/23, gateway 44.101.2.1 |
| `CNIT340.G063.Gurtner.U` | Ubuntu Server 24.04 (minimized) | 4 GB / 60 GB | CNIT340WL Linux B | 44.101.4.63/23, gateway 44.101.4.1 |

Shared settings: DNS `44.2.1.44` / `44.2.1.45`, memory overcommit on, **Secure Boot + vTPM** on every VM.

**Partition layout.** `/var` and `/home` are kept separate so that runaway logs or user data can't
fill the root filesystem:

| Mount | Rocky | Ubuntu |
| --- | --- | --- |
| `/` | 43 GiB | 46 GB |
| `/home` | 4 GiB | 4 GB |
| `/var` | 4 GiB | 4 GB |
| `/boot`, `/boot/efi` | auto | 4 GB, 1 GB |
| swap | 6 GiB partition | 6 GB swap file |

## Procedure

### 1. Windows Server 2022
1. Created the VM in Prism and attached CD-ROMs cloned from the Windows Server 2022 image and **Nutanix VirtIO 1.2.5**.
2. In setup, chose *Custom install* and used **Load driver → VirtIO → Windows Server 2022 → x64 →
   VirtIO SCSI pass-through controller** so the installer could see the disk.
3. Unmounted the ISO and installed **Nutanix Guest Tools (NGT)** through *Guest Tools → Mount Installer*.
4. Set a static IPv4 address and DNS, then applied all Windows updates.

### 2. Rocky Linux 10.2
1. Minimal install, time zone *America/Indianapolis*.
2. **Custom partitioning** per the plan, with `/var` and swap added by hand.
3. Manual IPv4 (address, `/23` netmask, gateway, DNS) and hostname `jgurtne.rocky.c340.cit.lcl`.
   Created the admin user `jgurtne`.
4. Patched and installed NGT:
   ```bash
   sudo dnf update
   blkid -L NUTANIX_TOOLS                      # find the mounted NGT installer
   sudo python3 /mnt/installer/linux/install_ngt.py --operation install
   ```
5. Set `SELINUX=disabled` in `/etc/selinux/config` and rebooted (see the note below).

### 3. Ubuntu Server 24.04
1. Minimized server install with a manual IPv4 setup (`44.101.4.0/23`) and the partition plan above.
   Turned on **OpenSSH** during install.
2. The installer doesn't create swap the same way, so I added a swap file afterward:
   ```bash
   sudo fallocate -l 6G /swapfile
   sudo chmod 600 /swapfile      # swap can hold memory contents, so only root may read it
   sudo mkswap /swapfile
   sudo swapon /swapfile
   swapon --show
   ```
3. `sudo apt-get update && sudo apt-get upgrade -y`, then installed NGT the same way as on Rocky.

### 4. Password complexity with PAM (both Linux hosts)
`/etc/security/pwquality.conf`:
```ini
minlen  = 8     # at least 8 characters
dcredit = -1    # at least 1 digit
ucredit = -1    # at least 1 uppercase letter
ocredit = -1    # at least 1 symbol
```
Then created test users and confirmed their home directories:
```bash
sudo useradd -m jack && sudo passwd jack
sudo useradd -m jill && sudo passwd jill
ls -ld /home/jack /home/jill
```

### 5. Disk quotas on `/home` (both Linux hosts)
```bash
sudo dnf install quota -y        # apt-get on Ubuntu
# /etc/fstab: add usrquota,grpquota to the /home mount options
sudo systemctl daemon-reload
sudo mount -o remount /home
sudo quotaon /home
sudo setquota -u jack 500000 600000 0 0 /home   # ~500 MB soft / ~600 MB hard
sudo setquota -u jill 500000 600000 0 0 /home
repquota /home
```
One user can no longer fill the shared `/home` partition. The soft limit warns, and the hard limit stops writes.

### 6. Network printing with CUPS (both Linux hosts)
```bash
sudo dnf install cups -y         # apt-get on Ubuntu
sudo systemctl enable --now cups
sudo lpadmin -p labprinter -E -v socket://44.44.113.254 -m drv:///sample.drv/generic.ppd
lpstat -p -d
```

## Verification
`lsblk -f` showed the partition layouts, `swapon --show` showed the Ubuntu swap, `repquota /home`
showed both users' quotas, and `systemctl status cups` showed the print service running on each host.

## Rocky vs Ubuntu: what I noticed
| | Rocky Linux | Ubuntu Server |
| --- | --- | --- |
| Package manager | `dnf` | `apt-get` |
| Installer | One hub screen for partitioning, network, hostname and users | A separate screen for each step |
| Swap | Created as a partition during install | Added as a swap file after install |
| Post-install admin | Same tools (`pwquality`, `quota`, CUPS, NGT installer) | Same |

## A note on SELinux
SELinux was **disabled** on Rocky as part of the lab steps. In production, I'd keep it **enforcing**
and fix whatever is blocked (for example with `audit2allow` or the right file contexts), or at most switch
to *permissive* while troubleshooting. SELinux confines services even after one is compromised, so turning
it off removes a real layer of defense.

## Skills demonstrated

Nutanix VM provisioning · VirtIO and guest tools · Windows Server install · Rocky and Ubuntu server installs ·
custom partitioning & swap · static networking · PAM password policy (`pwquality`) · disk quotas · CUPS ·
package management (`dnf` / `apt`) · cross-distro administration

## Screenshots

See [`screenshots/`](screenshots/) for the `lsblk -f`, `swapon`, `repquota`, `pwquality.conf` and CUPS output.
