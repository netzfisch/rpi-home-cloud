# Raspberry Pi Cloud Server - AI Agent Instructions

## Project Overview

This is a **cloud-init based provisioning system** for automated Raspberry Pi server setup using Ubuntu Server 25.10 ARM64. The system creates a headless Docker host with RAID1 storage, Samba shares, and network services.

**Architecture**: Single YAML file (`user-data`) drives complete system provisioning via cloud-init on first boot.

## Critical Source of Truth

⚠️ **Template + Secrets Architecture** - configuration is split for security:
- `user-data.template` → cloud-init template with ${VARIABLE} placeholders (commit to git)
- `secrets.env.example` → example secrets template (commit to git)
- `secrets.env` → actual secrets configuration (gitignored, NEVER commit!)
- `generate-userdata.sh` → validation and generation script
- `user-data` → generated final configuration (gitignored)
- `agent.md` → comprehensive technical documentation and procedures
- `README.md` → quick start guide for end users

## Key Architecture Decisions

### Why cloud-init?
Enables declarative, version-controlled infrastructure. The entire system state is defined in one file - no manual configuration steps after flashing.

### Why RAID1 with mdadm?
Data redundancy for server reliability. Two external USB drives (`/dev/sda`, `/dev/sdb`) mirror data to `/mnt/raid1/`.

### Why Docker services, not native installs?
Isolation, easy updates, and portability. Three containers: `unifi` (network controller), `ddclient` (dynamic DNS), `vpnserver` (optional remote access).

### Network Configuration Strategy
Static IP (`192.168.1.10`) via `/etc/dhcpcd.conf` written by cloud-init, not netplan. Gateway at `192.168.1.1` is likely another Pi running DNS/routing.

## Critical Workflows

### Deploying Changes

```bash
# 1. Configure secrets
cp secrets.env.example secrets.env
vim secrets.env  # Replace all CHANGE_ME_* values

# 2. Generate user-data
./generate-userdata.sh      # Validates and generates user-data from template

# 3. Flash new SD card
unxz ubuntu-25.10-preinstalled-server-arm64+raspi.img.xz
pv ubuntu-25.10-preinstalled-server-arm64+raspi.img | \
  sudo dd iflag=fullblock of=/dev/mmcblk0 bs=64M oflag=direct && sync
cp user-data /media/$USER/system-boot/user-data

# 4. Boot Pi, monitor provisioning (10-15 min)
ssh pirate@192.168.1.10
sudo tail -f /var/log/cloud-init-output.log
```

### Re-running cloud-init (DANGER ZONE)

**MUST set `fs_setup: overwrite: false`** before re-running or RAID data is destroyed!

```bash
# Wait for RAID sync completion
watch cat /proc/mdstat

# Clean generated files
sudo rm -R /etc/dhcpcd.conf /etc/mdadm/mdadm.conf /etc/samba/smb.conf \
  /opt/unifi/compose.yml /opt/unifi/unifi.service
docker stop ddclient unifi && docker rm ddclient unifi

# Clean cloud-init state and reboot
sudo cloud-init clean --logs --reboot
```

### Debugging Failed Provisioning

```bash
# Check full cloud-init output
sudo cat /var/log/cloud-init-output.log

# Test individual modules
sudo cloud-init single --name disk_setup --frequency always
sudo cloud-init single --name runcmd --frequency always

# Verify RAID status
cat /proc/mdstat
mdadm --detail /dev/md0
```

## Project-Specific Conventions

### User Management Pattern
- **pirate**: Admin user with shell access, Docker permissions, GitHub SSH key import
- **user1/user2**: Samba-only users (`inactive: true`, `no_create_home: true`, `shell: /usr/sbin/nologin`)

### File Writing Pattern
All configuration files use `write_files` module with absolute paths:
- `/etc/dhcpcd.conf` → static network config
- `/etc/samba/smb.conf` → share definitions
- `/opt/unifi/compose.yml` + `unifi.service` → systemd-managed Docker service

### Command Execution Sequence
The `runcmd` section follows this order:
1. Restart services (pick up hostname/network changes)
2. Configure MOTD (custom status display)
3. Create RAID array (`mdadm --create`)
4. Mount and persist in fstab
5. Create share directories with correct permissions
6. Set Samba passwords (piped to `smbpasswd -s`)
7. Launch Docker containers
8. Enable systemd services

### Secrets Management Pattern
Configuration is managed via external secrets file:
- Edit `secrets.env` with actual values (gitignored, never committed)
- Run `./generate-userdata.sh` to generate `user-data` from `user-data.template`
- Validation includes: required variables check, placeholder detection, cloud-init schema validation
- All secrets stay local - only templates are version controlled

## Common Pitfalls for AI Agents

❌ **Don't** edit `user-data` directly (it's generated)  
✅ **Do** edit `user-data.template` and run `./generate-userdata.sh`

❌ **Don't** commit `secrets.env` or generated `user-data` to git  
✅ **Do** only commit templates (`user-data.template`, `secrets.env.example`)

❌ **Don't** leave placeholder credentials (`CHANGE_ME_*`)  
✅ **Do** ensure all secrets.env values are real before deployment

❌ **Don't** suggest netplan for network config  
✅ **Do** modify `/etc/dhcpcd.conf` via `write_files` module in template

❌ **Don't** set `overwrite: true` on existing RAID partitions  
✅ **Do** use `overwrite: false` to preserve data

❌ **Don't** modify `/boot/user-data` path  
✅ **Do** use `/boot/firmware/user-data` (Ubuntu Server location)

❌ **Don't** forget to validate after template changes  
✅ **Do** run `./generate-userdata.sh` which validates automatically

## Integration Points

### Docker Services
- **unifi**: Web UI at `https://192.168.1.10:8443`, systemd-managed via `/opt/unifi/unifi.service`
- **ddclient**: Runs detached, updates `liberty.spdns.eu` with current public IP
- **vpnserver**: Commented out by default, requires firewall port forwarding (500/udp, 4500/udp)

### External Dependencies
- **Securepoint SPDNS**: Dynamic DNS provider (token in user-data)
- **Docker Hub images**: `ryansch/unifi-rpi`, `netzfisch/rpi-dyndns`, `netzfisch/rpi-vpn-server`
- **GitHub SSH keys**: Imported via `ssh_import_id: gh:netzfisch`

## Quick Reference

### Key Files
- `user-data.template` → main configuration template with ${VARIABLE} placeholders
- `secrets.env.example` → secrets template (commit to git)
- `secrets.env` → actual secrets (gitignored)
- `generate-userdata.sh` → validation and generation script
- `user-data` → generated final configuration (gitignored)
- `agent.md` → comprehensive technical documentation and procedures
- `README.md` → quick start guide for end users
- `readme_ubuntu-server.md` → boot partition file descriptions

### Key Directories (on target system)
- `/boot/firmware/` → contains `user-data`, `network-config`, boot files
- `/mnt/raid1/` → RAID1 mount point with share subdirectories
- `/opt/unifi/` → Unifi Controller compose + systemd files
- `/var/log/cloud-init-output.log` → provisioning logs

### Access Points
- SSH: `ssh pirate@192.168.1.10` (default password: `hypriot`, expires on first login)
- Samba: `smb://192.168.1.10/[user1|user2|shared|public]`
- Unifi: `https://192.168.1.10:8443`
- Hostname: `liberty10.local` (via Avahi mDNS)
