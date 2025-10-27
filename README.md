# Cloud Server for the Raspberry PI

Turn your [Raspberry PI](http://raspberrypi.org) within **15 minutes** into a **Cloud Server** allowing **access from remote** networks as well as serving any **Docker Container**.

Inspired by he [The MagPi Magazine](https://magpi.raspberrypi.org/articles/build-a-raspberry-pi-nas) about **creating a cloud server with the new Raspberry Pi 4** I added

* [cloud-init](https://cloud-init.io/) for automatic **provisioning of customized cloud instances**,
* [rpi-dyndns](https://github.com/netzfisch/rpi-dyndns) for **dynamic DNS** resolution and
* [rpi-vpn-server](https://github.com/netzfisch/rpi-vpn-server) to **access the server** from a remote network and
* based it on **Ubuntu Server 25.10** (ARM64) for **docker host/server** functionality.

If you find this useful, **do not forget to star** the repository ;-)

## Requirements

- [Raspberry PI](http://raspberrypi.org)
- other [Hardware](https://magpi.raspberrypi.org/articles/build-a-raspberry-pi-nas)
- Dynamic DNS service provider, e.g. from [Securepoint](https://www.spdns.de/)

### Setup

#### Step 1: Configure Secrets

- **Create secrets file** from the example template:

```sh
$ cp secrets.env.example secrets.env
```

- **Edit secrets.env** and replace ALL placeholder values with your actual configuration:
  - User credentials (admin password, Samba passwords)
  - Network settings (static IP, gateway, DNS servers)
  - Dynamic DNS hostname and token

**IMPORTANT:** Never commit `secrets.env` to version control - it's already in `.gitignore`.

#### Step 2: Generate User-Data

- **Make the generation script executable** (first time only):

```sh
$ chmod +x generate-userdata.sh
```

- **Run the generation script** to generate the final user-data file:

```sh
$ ./generate-userdata.sh
```

This will:
- Validate that all required secrets are configured
- Warn about placeholder values (CHANGE_ME)
- Generate the final `user-data` file with your secrets
- Validate the cloud-init schema (if cloud-init is installed)

#### Step 3: Flash SD Card

- Download the latest **Ubuntu Server image** for Raspberry Pi from [ubuntu.com/download/raspberry-pi](https://ubuntu.com/download/raspberry-pi)
- Flash the SD-Card using `dd` and copy the generated user-data file to the system-boot partition:

```sh
$ unxz ubuntu-25.10-preinstalled-server-arm64+raspi.img.xz
$ pv ubuntu-25.10-preinstalled-server-arm64+raspi.img | \
  sudo dd iflag=fullblock of=/dev/mmcblk0 bs=64M oflag=direct && sync
$ cp user-data /media/$USER/system-boot/user-data
```

- Put the SD-Card back into the Raspberry and boot. The server will be **automatically provisioned** and set up - **repeatable and reliable** :-)
- The system will automatically set up:
  - RAID1 storage with two external drives (/dev/sda, /dev/sdb)
    - **Note**: The provisioning script handles all RAID scenarios (fresh disks, existing arrays, or reused drives) by automatically cleaning any previous RAID metadata before creating the array. This ensures the RAID is always created as `md0` instead of auto-assembling as `md127`.
    - **CRITICAL WARNING**: Setting `fs_setup: overwrite: true` will destroy RAID data! Always set `overwrite: false` for existing partitions.
  - Samba shares (user1, user2, shared, public)
  - Docker containers: **Unifi Controller** (network management) and **ddclient** (dynamic DNS)
  - Custom MOTD with system status display
- **First login**: SSH as `pirate@192.168.1.10` with password `hypriot`
  - You will be prompted to change the password immediately
  - After changing the password, the session will close (this is expected)
  - Log in again with your new password
- For configuration of DynDNS and VPN check the respective documentation,
  especially
  - set the update token for [rpi-dyndns](https://github.com/netzfisch/rpi-dyndns),
  - import/generate the secrets for [rpi-vpn-server](https://github.com/netzfisch/rpi-vpn-server), and
  - enable port forwarding at your firewall for the UDP ports 500 and 4500.
- Access the system via SSH: `ssh pirate@192.168.1.10` (default password: hypriot, must change on first login)
- Access Unifi Controller web UI: `https://192.168.1.10:8443`
- **Done!**

### Debugging

Log into the instance, check the logs at `/var/log/cloud-init-output.log`, and run single commands to verify:

    $ sudo cloud-init single --name users --frequency always
    $ sudo cloud-init single --name disk_setup --frequency always
    $ sudo cloud-init single --name runcmd --frequency always

Check RAID status and system configuration:

    $ cat /proc/mdstat
    $ mdadm --detail /dev/md0
    $ df -h /mnt/raid1
    $ docker ps

Edit `/boot/firmware/user-data`, see the [documentation](https://cloudinit.readthedocs.io/) for details. 

**Before reboot,** wait for RAID sync to complete and clean up:

    $ watch cat /proc/mdstat                                                # wait until sync finished
    $ sudo rm -R /etc/dhcpcd.conf /etc/mdadm/mdadm.conf /etc/samba/smb.conf \
      /opt/unifi/compose.yml /opt/unifi/unifi.service                       # remove auto-generated files
    $ docker stop ddclient unifi && docker rm ddclient unifi                # stop and remove containers
    $ sudo cloud-init clean --logs --reboot

## Project Structure

```
rpi-cloud-server/
├── .gitignore              # Ensures secrets stay private
├── generate-userdata.sh    # Script to generate user-data from template
├── README.md               # This tradional file ;-)
├── secrets.env             # Your actual secrets (gitignored, never commit!)
├── secrets.env.example     # Example template (commit to git)
├── user-data               # Generated file (gitignored, created by generate-userdata.sh)
└── user-data.template      # Template with variable placeholders (commit to git)
```

## TODO

- [x] Reference and load secrets from local environment file via generation script
- [ ] Add uninterruptible power supply, e.g. USB powerbank
- [ ] Add script to act on "power loss" to shutdown server properly

## License

The MIT License (MIT), see [LICENSE](https://github.com/netzfisch/rpi-cloud-server/blob/master/LICENSE) file.
