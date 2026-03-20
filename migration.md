## Phase 1 — Inventory \& Audit

- [ ] Document current partition layout: `lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,UUID`
NAME   FSTYPE   SIZE MOUNTPOINT UUID
sda           931.5G            
├─sda1 vfat     300M /boot/efi  2CBC-CD5D
├─sda2 btrfs  922.4G /srv       9f31942d-3f98-400b-83f4-3c0bb1c7309b
└─sda3 swap     8.8G [SWAP]     5ca1dae4-c551-436d-a6dd-b985c8b59e37
sdb             7.3T            
└─sdb1 btrfs    7.3T /data      a520d5ae-8150-4e88-8837-f0ca9d643b2b
zram0  swap     7.6G [SWAP]     5c8f2ca3-b2b3-4786-8691-89159765589b

- [ ] List all running services: `systemctl list-units --type=service --state=running`

  UNIT                               LOAD   ACTIVE SUB     DESCRIPTION                                                        
  avahi-daemon.service               loaded active running Avahi mDNS/DNS-SD Stack
  caddy.service                      loaded active running Caddy web server
  containerd.service                 loaded active running containerd container runtime
  dbus-broker.service       UNIT FILE                             STATE   PRESET  
grub-btrfs-snapper.path               enabled disabled
avahi-daemon.service                  enabled disabled
bluetooth.service                     enabled disabled
caddy.service                         enabled disabled
docker.service                        enabled disabled
fail2ban.service                      enabled disabled
garuda-pacman-lock.service            enabled disabled
garuda-pacman-snapshot-reject.service enabled disabled
getty@.service                        enabled enabled 
kodi-gbm.service                      enabled disabled
libvirtd.service                      enabled disabled
linux-modules-cleanup.service         enabled disabled
NetworkManager-dispatcher.service     enabled disabled
NetworkManager-wait-online.service    enabled disabled
NetworkManager.service                enabled disabled
sshd.service                          enabled disabled
systemd-oomd.service                  enabled disabled
systemd-resolved.service              enabled enabled 
systemd-timesyncd.service             enabled enabled 
avahi-daemon.socket                   enabled disabled
libvirtd-admin.socket                 enabled disabled
libvirtd-ro.socket                    enabled disabled
libvirtd.socket                       enabled disabled
systemd-journal-gatewayd.socket       enabled disabled
virtlockd-admin.socket                enabled disabled
virtlockd.socket                      enabled disabled
virtlogd-admin.socket                 enabled disabled
virtlogd.socket                       enabled disabled
remote-fs.target                      enabled enabled 
btrfs-balance.timer                   enabled disabled
btrfs-defrag.timer                    enabled disabled
btrfs-scrub.timer                     enabled disabled
btrfs-trim.timer                      enabled disabled
snapper-cleanup.timer                 enabled disabled
         loaded active running D-Bus System Message Bus
  dirmngr@etc-pacman.d-gnupg.service loaded active running GnuPG network certificate management daemon for /etc/pacman.d/gnupg
  docker.service                     loaded active running Docker Application Container Engine
  fail2ban.service                   loaded active running Fail2Ban Service
  kodi-gbm.service                   loaded active running Kodi standalone (GBM)
  libvirtd.service                   loaded active running libvirt legacy monolithic daemon
  NetworkManager.service             loaded active running Network Manager
  polkit.service                     loaded active running Authorization Manager
  sshd.service                       loaded active running OpenSSH Daemon
  syncthing@kodi.service             loaded active running Syncthing - Open Source Continuous File Synchronization for kodi
  systemd-journald.service           loaded active running Journal Service
  systemd-logind.service             loaded active running User Login Management
  systemd-machined.service           loaded active running Virtual Machine and Container Registration Service
  systemd-oomd.service               loaded active running Userspace Out-Of-Memory (OOM) Killer
  systemd-resolved.service           loaded active running Network Name Resolution
  systemd-timesyncd.service          loaded active running Network Time Synchronization
  systemd-udevd.service              loaded active running Rule-based Manager for Device Events and Files
  udisks2.service                    loaded active running Disk Manager
  user@1000.service                  loaded active running User Manager for UID 1000
  virtlogd.service                   loaded active running libvirt logging daemon


- [ ] List all enabled services: `systemctl list-unit-files --state=enabled`

UNIT FILE                             STATE   PRESET  
grub-btrfs-snapper.path               enabled disabled
avahi-daemon.service                  enabled disabled
bluetooth.service                     enabled disabled
caddy.service                         enabled disabled
docker.service                        enabled disabled
fail2ban.service                      enabled disabled
garuda-pacman-lock.service            enabled disabled
garuda-pacman-snapshot-reject.service enabled disabled
getty@.service                        enabled enabled 
kodi-gbm.service                      enabled disabled
libvirtd.service                      enabled disabled
linux-modules-cleanup.service         enabled disabled
NetworkManager-dispatcher.service     enabled disabled
NetworkManager-wait-online.service    enabled disabled
NetworkManager.service                enabled disabled
sshd.service                          enabled disabled
systemd-oomd.service                  enabled disabled
systemd-resolved.service              enabled enabled 
systemd-timesyncd.service             enabled enabled 
avahi-daemon.socket                   enabled disabled
libvirtd-admin.socket                 enabled disabled
libvirtd-ro.socket                    enabled disabled
libvirtd.socket                       enabled disabled
systemd-journal-gatewayd.socket       enabled disabled
virtlockd-admin.socket                enabled disabled
virtlockd.socket                      enabled disabled
virtlogd-admin.socket                 enabled disabled
virtlogd.socket                       enabled disabled
remote-fs.target                      enabled enabled 
btrfs-balance.timer                   enabled disabled
btrfs-defrag.timer                    enabled disabled
btrfs-scrub.timer                     enabled disabled
btrfs-trim.timer                      enabled disabled
snapper-cleanup.timer                 enabled disabled

***

## Phase 2 — System Config Backup

Back everything up to the **SATA SSD** or an external device, never to the M.2 you're about to wipe.

- [ ] Backup `/etc` in full: `sudo rsync -aAXv /etc/ /mnt/sata/backup/etc/`
- [ ] Backup home directories: `sudo rsync -aAXv /home/ /mnt/sata/backup/home/`
- [ ] Backup `/root` (root user configs/scripts): `sudo rsync -aAXv /root/ /mnt/sata/backup/root/`

***

## Phase 3 — Docker Backup

- [ ] Export all Docker Compose files (usually in `/home/`, `/opt/`, or `/srv/`): `find / -name "docker-compose.yml" 2>/dev/null`
- [ ] Stop all containers gracefully: `docker compose down` (per stack) or `docker stop $(docker ps -q)`
- [ ] Backup all named Docker volumes:[^1_2]

```bash
for vol in $(docker volume ls -q); do
  docker run --rm -v $vol:/data -v /mnt/sata/backup/docker-volumes:/backup \
    alpine tar czf /backup/${vol}.tar.gz -C /data .
done
```

- [ ] Save custom Docker images not on a registry: `docker save <image> | gzip > /mnt/sata/backup/images/<image>.tar.gz`
- [ ] Backup `/etc/docker/daemon.json` if customized
- [ ] Note any bind mounts pointing to the M.2 (outside of `/var/lib/docker/volumes/`) — back those up separately with `rsync`

***

## Phase 4 — Virtual Machine Backup

Assuming libvirt/KVM:

- [ ] List all VMs: `virsh list --all`
- [ ] Export VM XML definitions:[^1_3]

```bash
for vm in $(virsh list --all --name); do
  virsh dumpxml $vm > /mnt/sata/backup/vms/${vm}.xml
done
```

- [ ] Shut down all VMs cleanly: `virsh shutdown <vmname>` (use `virsh destroy` only as last resort)
- [ ] Copy all disk images (`.qcow2`, `.img`) to the SATA SSD:

```bash
rsync -aAXv --progress /var/lib/libvirt/images/ /mnt/sata/backup/libvirt-images/
```

- [ ] Backup `/etc/libvirt/` for network/pool definitions: `sudo rsync -aAXv /etc/libvirt/ /mnt/sata/backup/libvirt-conf/`

***

## Phase 5 — Final Pre-Wipe Checks

- [ ] Verify all backups are readable and complete (spot-check a few files, test a tar extraction)
- [ ] Double-check the SATA SSD is **not** the target disk in your installer
- [ ] Note the M.2 device name (`nvme0n1` or similar) to avoid confusion: `lsblk`
- [ ] Write down network config (static IP, hostname, DNS): `ip a`, `cat /etc/hostname`, `cat /etc/resolv.conf`
- [ ] Reboot once more to confirm SATA SSD data is intact and accessible before proceeding

***

## Phase 6 — Fresh Arch Install on M.2

Follow the standard [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide), targeting only the M.2.[^1_4]

- [ ] Boot Arch ISO
- [ ] Partition **only** the M.2 (verify device node carefully with `lsblk`)
- [ ] Format and mount M.2 partitions (`/`, `/boot`)
- [ ] **Do NOT mount or format the SATA SSD during install** — leave it disconnected or just unformatted/unmounted
- [ ] Install base system: `pacstrap /mnt base linux linux-firmware ...`
- [ ] Generate initial fstab: `genfstab -U /mnt >> /mnt/etc/fstab`
- [ ] Configure locale, hostname, network, bootloader (same as before)

***

## Phase 7 — Restore System

- [ ] Restore `/etc` selectively (don't blindly overwrite new install's defaults): prioritize custom configs like `/etc/fstab`, `/etc/ssh/`, `/etc/hosts`, service configs
- [ ] Add SATA SSD mount entries to `/etc/fstab` using the **original UUIDs** noted in Phase 1
- [ ] Mount SATA SSD and verify data: `mount -a && ls /mnt/sata/`
- [ ] Reinstall packages from list:[^1_1]

```bash
pacman -S --needed - < /mnt/sata/backup/pkglist.txt
```

- [ ] Reinstall AUR packages (after setting up AUR helper): `yay -S --needed - < /mnt/sata/backup/pkglist-aur.txt`
- [ ] Re-enable all previously enabled services: `systemctl enable <service>`
- [ ] Restore SSH keys: `cp -r /mnt/sata/backup/ssh/ /etc/ssh/`

***

## Phase 8 — Restore Docker

- [ ] Install Docker and Docker Compose, start the service
- [ ] Restore Docker volumes:[^1_5]

```bash
for backup in /mnt/sata/backup/docker-volumes/*.tar.gz; do
  vol=$(basename $backup .tar.gz)
  docker volume create $vol
  docker run --rm -v $vol:/data -v /mnt/sata/backup/docker-volumes:/backup \
    alpine sh -c "cd /data && tar xzf /backup/${vol}.tar.gz"
done
```

- [ ] Load saved custom images: `docker load < /mnt/sata/backup/images/<image>.tar.gz`
- [ ] Restore Docker Compose files to their original paths
- [ ] Restore `/etc/docker/daemon.json`
- [ ] Start stacks one by one and verify: `docker compose up -d && docker ps`

***

## Phase 9 — Restore Virtual Machines

- [ ] Install `libvirt`, `qemu`, `virt-manager` (or headless equivalent)
- [ ] Start and enable `libvirtd`
- [ ] Copy disk images back: `rsync -aAXv /mnt/sata/backup/libvirt-images/ /var/lib/libvirt/images/`
- [ ] Restore VM XML definitions:[^1_3]

```bash
for xml in /mnt/sata/backup/vms/*.xml; do
  virsh define $xml
done
```

- [ ] Restore libvirt network/pool configs: `cp -r /mnt/sata/backup/libvirt-conf/ /etc/libvirt/`
- [ ] Start VMs and verify: `virsh start <vmname> && virsh list`

***

## Phase 10 — Post-Migration Validation

- [ ] All previously enabled systemd services are running
- [ ] Network connectivity and hostname correct
- [ ] SATA SSD data fully accessible and intact
- [ ] All Docker containers healthy: `docker ps` and check logs
- [ ] All VMs reachable (ping from host or via `virsh console`)
- [ ] Run `journalctl -p err -b` to check for boot errors
- [ ] Verify crontabs restored and timing is correct
- [ ] Update system: `pacman -Syu`
- [ ] Take a first backup snapshot of the new clean state
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_6][^1_7][^1_8][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://bbs.archlinux.org/viewtopic.php?id=311270

[^1_2]: https://cubepath.com/docs/server-migration/docker-container-migration

[^1_3]: https://www.vinchin.com/vm-migration/kvm-migrate-vm-to-another-host.html

[^1_4]: https://wiki.archlinux.org/title/Migrate_installation_to_new_hardware

[^1_5]: https://eastondev.com/blog/en/posts/dev/20251217-docker-volume-backup/

[^1_6]: https://www.reddit.com/r/archlinux/comments/1er4zzm/how_do_you_backup_and_restore_if_you_change_your/

[^1_7]: https://www.facebook.com/groups/archlinuxen/posts/10158990351103393/

[^1_8]: https://www.tencentcloud.com/techpedia/100661

[^1_9]: http://libvirtbackup.grinser.de

[^1_10]: https://www.youtube.com/watch?v=2cLfiIAGl88

[^1_11]: https://wiki.archlinux.org/title/System_maintenance

[^1_12]: https://docs.docker.com/dhi/migration/checklist/

[^1_13]: https://dohost.us/index.php/2025/09/09/backing-up-and-restoring-virtual-machines-in-kvm/

[^1_14]: https://wiki.archlinux.org/title/System_backup

[^1_15]: https://www.reddit.com/r/docker/comments/qotavh/how_do_you_backup_your_docker_volumes/


---

# please output as downloadable markdown

I can’t attach a file in this message, but here is a ready-to-save Markdown document. It follows standard Arch migration/package export guidance and the Arch installation documentation structure used in the earlier plan.[^2_1][^2_2]

```markdown
# Server Migration Checklist

## Scenario

- Current server: Arch Linux
- Storage:
  - M.2 SSD: current system installation, Docker, virtual machines, data
  - SATA SSD: contains data and should remain unchanged
- Goal:
  - Keep the SATA SSD as-is
  - Wipe and reinstall Arch Linux on the M.2 SSD
  - Restore system configuration, Docker, VMs, and required data afterward

---

## 1. Preparation

- [ ] Define a maintenance window
- [ ] Confirm you have physical or remote console access during reinstall
- [ ] Prepare external backup target if the SATA SSD does not have enough free space
- [ ] Ensure you have the Arch install medium ready
- [ ] Record current hostname, IP configuration, DNS, gateway, and SSH access details
- [ ] Record disk layout:
  ```bash
  lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT,UUID
  blkid
```

- [ ] Clearly identify:
    - [ ] M.2 device name (for example `nvme0n1`)
    - [ ] SATA SSD device name
- [ ] Verify which mount points belong to the SATA SSD
- [ ] Make sure you can distinguish both disks without ambiguity

---

## 2. Inventory

### Packages

- [ ] Export explicitly installed native packages:

```bash
pacman -Qqe > ~/pkglist.txt
```

- [ ] Export AUR packages:

```bash
pacman -Qqem > ~/pkglist-aur.txt
```

- [ ] Note which AUR helper you use (`yay`, `paru`, etc.)


### Services

- [ ] List enabled services:

```bash
systemctl list-unit-files --state=enabled
```

- [ ] List running services:

```bash
systemctl list-units --type=service --state=running
```


### Storage and mounts

- [ ] Save current fstab:

```bash
cp /etc/fstab ~/fstab.backup
```

- [ ] Record all UUIDs and mount points
- [ ] Note any bind mounts used by Docker, VMs, or applications


### Network and access

- [ ] Save hostname:

```bash
cat /etc/hostname
```

- [ ] Save hosts file:

```bash
cat /etc/hosts
```

- [ ] Save network info:

```bash
ip a
ip r
resolvectl status
```


### Applications and runtime

- [ ] List Docker containers:

```bash
docker ps -a
```

- [ ] List Docker volumes:

```bash
docker volume ls
```

- [ ] List Docker images:

```bash
docker images
```

- [ ] List libvirt VMs:

```bash
virsh list --all
```

- [ ] Note storage locations for:
    - [ ] Docker compose files
    - [ ] Docker bind mounts
    - [ ] VM XML definitions
    - [ ] VM disk images
    - [ ] Databases
    - [ ] Custom scripts and automation

---

## 3. Backup

> Important: Store backups on the SATA SSD or on external storage, not on the M.2 SSD that will be reinstalled.

### Core system config

- [ ] Backup `/etc`:

```bash
rsync -aAXHv /etc/ /path/to/backup/etc/
```

- [ ] Backup `/root`:

```bash
rsync -aAXHv /root/ /path/to/backup/root/
```

- [ ] Backup relevant user home directories:

```bash
rsync -aAXHv /home/ /path/to/backup/home/
```


### SSH and certificates

- [ ] Backup SSH configuration and keys:

```bash
rsync -aAXHv /etc/ssh/ /path/to/backup/etc-ssh/
```

- [ ] Backup user SSH keys:

```bash
rsync -aAXHv ~/.ssh/ /path/to/backup/user-ssh/
```

- [ ] Backup TLS/certificate material if present:

```bash
rsync -aAXHv /etc/letsencrypt/ /path/to/backup/letsencrypt/
```


### Scheduled jobs and automation

- [ ] Backup root crontab:

```bash
crontab -l > /path/to/backup/root-crontab.txt
```

- [ ] Backup user crontabs as needed
- [ ] Backup:
    - [ ] `/etc/cron.d/`
    - [ ] `/etc/systemd/system/`
    - [ ] custom scripts under `/usr/local/bin`, `/opt`, `/srv`, or home directories


### Docker

- [ ] Save compose files and environment files
- [ ] Save Docker daemon config if present:

```bash
cp /etc/docker/daemon.json /path/to/backup/daemon.json
```

- [ ] Stop containers cleanly:

```bash
docker stop $(docker ps -q)
```

- [ ] Backup bind-mounted directories
- [ ] Backup named volumes

Example for named volumes:

```bash
mkdir -p /path/to/backup/docker-volumes
for vol in $(docker volume ls -q); do
  docker run --rm \
    -v "${vol}:/data" \
    -v /path/to/backup/docker-volumes:/backup \
    alpine \
    sh -c "cd /data && tar czf /backup/${vol}.tar.gz ."
done
```

- [ ] Save custom images if needed:

```bash
docker save image:tag | gzip > /path/to/backup/image_tag.tar.gz
```


### Virtual machines

- [ ] Gracefully shut down VMs:

```bash
virsh shutdown <vmname>
```

- [ ] Export VM definitions:

```bash
mkdir -p /path/to/backup/vms
for vm in $(virsh list --all --name); do
  virsh dumpxml "$vm" > "/path/to/backup/vms/${vm}.xml"
done
```

- [ ] Backup VM disk images:

```bash
rsync -aAXHv /var/lib/libvirt/images/ /path/to/backup/libvirt-images/
```

- [ ] Backup libvirt configuration:

```bash
rsync -aAXHv /etc/libvirt/ /path/to/backup/etc-libvirt/
```


### Databases and app data

- [ ] Create logical dumps for databases where appropriate
- [ ] Backup all application data directories
- [ ] Verify that important state is not stored only inside containers

---

## 4. Backup Validation

- [ ] Confirm package lists exist
- [ ] Confirm `/etc`, home directories, SSH material, compose files, VM XML, and VM images are present in backup
- [ ] Open a few restored files from backup to test readability
- [ ] Test extracting at least one Docker volume archive:

```bash
tar tzf /path/to/backup/docker-volumes/<volume>.tar.gz | head
```

- [ ] Confirm total backup size looks plausible
- [ ] Confirm SATA SSD data is still mounted and intact
- [ ] Confirm nothing critical remains only on the M.2 without backup

---

## 5. Cutover Preparation

- [ ] Schedule service shutdown
- [ ] Stop application services
- [ ] Stop Docker containers
- [ ] Stop or shut down VMs
- [ ] Run one final incremental backup for changed data
- [ ] Record final state and timestamps
- [ ] If possible, disconnect the SATA SSD physically during install for safety
- [ ] If not possible, be extra careful to select only the M.2 during partitioning

---

## 6. Fresh Arch Install on M.2

- [ ] Boot Arch installation media
- [ ] Identify disks again with:

```bash
lsblk
```

- [ ] Double-check the target disk is the M.2 SSD
- [ ] Partition only the M.2 SSD
- [ ] Create filesystems on the new M.2 partitions
- [ ] Mount target filesystems
- [ ] Install base system
- [ ] Generate `fstab`
- [ ] Install bootloader
- [ ] Configure:
    - [ ] timezone
    - [ ] locale
    - [ ] hostname
    - [ ] network
    - [ ] users
    - [ ] sudo
    - [ ] SSH
- [ ] Reboot into the fresh system

---

## 7. Base Restore

### Mount SATA SSD

- [ ] Attach or mount the unchanged SATA SSD
- [ ] Verify its UUID and mount point
- [ ] Add it to `/etc/fstab` carefully
- [ ] Test:

```bash
mount -a
```


### Restore package baseline

- [ ] Reinstall standard packages:

```bash
pacman -S --needed - < /path/to/backup/pkglist.txt
```

- [ ] Reinstall AUR helper
- [ ] Reinstall AUR packages:

```bash
yay -S --needed - < /path/to/backup/pkglist-aur.txt
```


### Restore selected configuration

> Do not blindly overwrite the entire new `/etc`. Merge selectively.

- [ ] Restore:
    - [ ] `/etc/fstab` entries for non-system mounts
    - [ ] `/etc/ssh/` as needed
    - [ ] `/etc/hosts`
    - [ ] network-related configs
    - [ ] service configs
    - [ ] custom systemd units
    - [ ] sysctl settings
    - [ ] journald/logrotate/smartd/timers if used
- [ ] Restore scripts to their original locations
- [ ] Restore crontabs and timers
- [ ] Re-enable required services:

```bash
systemctl enable <service>
```


---

## 8. Docker Restore

- [ ] Install and enable Docker
- [ ] Restore `/etc/docker/daemon.json` if used
- [ ] Restore compose projects to their original locations
- [ ] Restore bind-mounted data directories
- [ ] Recreate named volumes

Example:

```bash
for backup in /path/to/backup/docker-volumes/*.tar.gz; do
  vol=$(basename "$backup" .tar.gz)
  docker volume create "$vol"
  docker run --rm \
    -v "${vol}:/data" \
    -v /path/to/backup/docker-volumes:/backup \
    alpine \
    sh -c "cd /data && tar xzf /backup/${vol}.tar.gz"
done
```

- [ ] Load saved custom images if required:

```bash
gunzip -c /path/to/backup/image_tag.tar.gz | docker load
```

- [ ] Start stacks one by one:

```bash
docker compose up -d
```

- [ ] Validate:
    - [ ] container status
    - [ ] logs
    - [ ] ports
    - [ ] bind mounts
    - [ ] persistent data

---

## 9. VM Restore

- [ ] Install `qemu`, `libvirt`, and related tools
- [ ] Enable and start libvirt
- [ ] Restore VM disk images:

```bash
rsync -aAXHv /path/to/backup/libvirt-images/ /var/lib/libvirt/images/
```

- [ ] Restore libvirt config if needed
- [ ] Define VMs again:

```bash
for xml in /path/to/backup/vms/*.xml; do
  virsh define "$xml"
done
```

- [ ] Check storage pools and networks
- [ ] Start VMs
- [ ] Verify console/network/disk access

---

## 10. Validation After Migration

- [ ] Confirm system boots cleanly
- [ ] Confirm hostname and network settings are correct
- [ ] Confirm SATA SSD mounts correctly on boot
- [ ] Confirm all expected data is present
- [ ] Confirm Docker containers are healthy
- [ ] Confirm VM storage paths are correct
- [ ] Confirm VMs start successfully
- [ ] Confirm SSH access works
- [ ] Confirm scheduled jobs and timers run
- [ ] Review logs:

```bash
journalctl -b -p warning
```

- [ ] Review failed services:

```bash
systemctl --failed
```


---

## 11. Cleanup

- [ ] Keep old backups until the new system has been stable for several days
- [ ] Remove obsolete entries from `fstab`, compose files, and service configs
- [ ] Document the new disk layout and mount scheme
- [ ] Create a fresh full-system backup after successful migration

---

## 12. High-Risk Points

- [ ] Accidentally formatting the SATA SSD
- [ ] Forgetting Docker bind mounts outside of named volumes
- [ ] Forgetting VM XML definitions or libvirt network config
- [ ] Restoring `/etc` too aggressively onto the fresh install
- [ ] Missing database dumps or app-specific state
- [ ] Changing UUIDs without updating `fstab` or boot config
- [ ] Losing SSH host keys or authorized keys
- [ ] Forgetting custom systemd units, timers, scripts, or secrets

---

## 13. Minimal Downtime Order

1. Final inventory check
2. Stop containers and VMs
3. Final incremental backup
4. Reinstall Arch on M.2
5. Restore mounts and base packages
6. Restore configs
7. Restore Docker
8. Restore VMs
9. Validate services
10. Reopen access

---

## 14. Notes

- Prefer selective restore over full overwrite for `/etc`
- Keep the SATA SSD untouched until the new system is fully verified
- If possible, test restore steps on a non-production system first
