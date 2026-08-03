# vm-init

Post-clone initial setup script for Ubuntu 24.04 / Debian 13 VMs on Proxmox.

Clone a VM from a prepared template, run `vm-init`, answer the prompts. It
configures identity, networking, security, packages, MOTD, swap, and disk
resize in one pass.

## Install

Bake it into the template before snapshotting so every clone has it:

```bash
curl -fsSL https://raw.githubusercontent.com/vinneamtsel/VM-init/main/vm-init -o /usr/local/sbin/vm-init
chmod +x /usr/local/sbin/vm-init
```

Or copy it to a single clone with scp.

## Usage

```bash
sudo vm-init
```

Every step asks a question with a default; press Enter to accept it. Safe to
re-run.

The script runs in two phases:

1. **Collect**: answers every question up front. Nothing is changed yet.
2. **Execute**: shows you the full plan, and applies everything only after you
   confirm. Aborting at the plan screen leaves the VM untouched.

## What it does

| Step | Description |
| ------ | ------------- |
| 1. Identity | hostname + /etc/hosts, regenerate machine-id and SSH host keys |
| 2. Network | DHCP or static IP (netplan on Ubuntu, ifupdown on Debian); static is applied last so an SSH session survives until the end |
| 3. Time | timezone, NTP (systemd-timesyncd) |
| 4. Security | admin sudo user, SSH password-auth and root-login toggles, ufw, fail2ban, unattended-upgrades |
| 5. Packages | apt update + full upgrade, base tools, herdr |
| 6. MOTD + prompt | dynamic login banner (hostname, IP, disk, memory, pending updates) and a colored two-line bash prompt with git branch and exit status |
| 7. Swap | optional 2G swapfile with swappiness 10 |
| 8. Disk resize | grows partition and filesystem (LVM and plain partitions, ext4/xfs) when the clone disk is larger |
| 9. Cleanup | purges cloud-init, autoremove, log vacuum |

Base tools installed in step 5: curl, wget, git, vim, htop, unzip, jq, rsync,
net-tools, ca-certificates, gnupg.

Optional software (multi-select menu): Docker CE + Compose (get.docker.com),
Node.js LTS (nvm), Nginx, python3-pip, PostgreSQL.

## Notes

- Targets Ubuntu 24.04 and Debian 13; other versions and distros warn but continue.
- On Debian the network step writes `/etc/network/interfaces.d/99-vm-init` (ifupdown, no extra packages) instead of netplan, and writes `/etc/resolv.conf` directly when neither resolvconf nor systemd-resolved manages it (ifupdown's resolved hook covers the symlink case). The MOTD banner works out of the box on both, via pam_motd + `/etc/update-motd.d`.
- With a static IP the SSH connection drops when the network config is applied; reconnect on the new IP.
- Node is installed via nvm for root and symlinked to /usr/local/bin, so every user gets node/npm/npx.
- herdr installs system-wide to /usr/local/bin.
- If you keep cloud-init, it may re-run after a machine-id change; purging it is recommended.
