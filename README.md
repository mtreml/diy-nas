# Goals

- Build a DIY NAS 
- OS drive: OpenMediaVault 6.0-16 on microSD card (cloned for backup)
- Data drive: RAID5 -> LUKS -> LVM2 -> EXT4 for read performance, failure safety and privacy
- Automatic unlocking of encrypted drive in a known network with Tang & Clevis
- Server should be able to do unattended shutdowns / reboots for energy saving


# Table of contents

- [Hardware](#hardware)
  - [HPE Proliant Microserver Gen8](#hpe-proliant-microserver-gen8)
    - [Specifications](#specifications)
    - [BIOS Update v2014.06.06 -> v2019.04.04**](#bios-update-v20140606---v20190404)
    - [iLO Firmware Update v2.10 (15.1.2015) -> v2.78 (22.4.2021)](#ilo-firmware-update-v210-1512015---v278-2242021)
  - [LTE modem / router](#lte-modem--router)
    - [DDNS](#ddns)
    - [Tang server on OpenWrt](#tang-server-on-openwrt)
    - [Resources](#resources)
- [Software](#software)
  - [Installation and initial configuration of OpenMediaVault 6.0-16](#installation-and-initial-configuration-of-openmediavault-60-16)
    - [Login via web console](#login-via-web-console)
    - [Login via SSH](#login-via-ssh)
  - [Encrypted data drive](#encrypted-data-drive)
    - [Partitions of same size on every physical disk](#partitions-of-same-size-on-every-physical-disk)
    - [RAID device on sda, sdb, sdc, sdd](#raid-device-on-sda-sdb-sdc-sdd)
    - [Encrypted LUKS container on the RAID device](#encrypted-luks-container-on-the-raid-device)
    - [LVM](#lvm)
    - [File system](#file-system)
    - [Clevis](#clevis)
    - [Resources](#resources-1)
  - [I/O benchmarks](#io-benchmarks)
    - [Resources](#resources-2)
- [Secure your server](#secure-your-server)
  - [Force strong user passwords](#force-strong-user-passwords)
  - [Enable unattended updates](#enable-unattended-updates)
  - [Limit access to sudo & su](#limit-access-to-sudo--su)
  - [SSH](#ssh)
  - [Network](#network)
  - [Resources](#resources-3)
- [Administration](#administration)
  - [Create a shared folder](#create-a-shared-folder)
  - [Mount smb/cifs share on the (Ubuntu) client](#mount-smbcifs-share-on-the-ubuntu-client)
  - [Mount NFS share in Ubuntu](#mount-nfs-share-in-ubuntu)
- [Backup](#backup)


TOC created with https://imthenachoman.github.io/nGitHubTOC/


---


# Hardware

## HPE Proliant Microserver Gen8

See the official specifications at https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c03793258#N1048F

### Specifications
- Intel Xeon E3-1220LV2 (17 W)
- 8 GB RAM
- 16 GB SanDisk A1 micro SD-card for OS (OS-drive on internal slot)
- 4 x 4 TB Western Digital Red Plus (data-drives on SATA slots)

### BIOS Update v2014.06.06 -> v2019.04.04**
- Get the firmware file *SP99427.exe* at https://support.hpe.com/connect/s/search
- Create USB-Key on Windows & boot server from it.

### iLO Firmware Update v2.10 (15.1.2015) -> v2.78 (22.4.2021)
- Get the firmware file *cp046467.exe* at https://support.hpe.com/connect/s/search
- Connect the server at the iLO network interface
- Login into the iLO web interface via IPv6 (credentials on the sticker at the backside of the microserver)
- Upload new firmware and flash it.
- Get an ILO Advanced Key and enter it in the iLO web interface

## LTE modem / router

- Install OpenWrt on your LTE modem / router if you can
- Set a static IP outside of dhcp range for your server machine in your router's web console
      ```
      IPv4: 192.168.x.xxx
      Mac-address: xx:xx:xx:xx:xx:xx
      ```
- Get a real public IP from ISP for your LTE sim card (if it does not have one)

### DDNS
- Use a DDNS service to map a domain name to your changing IP
- Configure a DDNS client on your router: https://openwrt.org/docs/guide-user/services/ddns/client
    
### Tang server on OpenWrt   
- Install a tang server (tested with tang 6.1 on OpenWrt 21.02.0)
- Edit the xinetd config
    ```sh
    vi /etc/xinetd.conf
    ```
- Add
    ```sh
    service tangd
    {
        port            = 8888
        socket_type     = stream
        wait            = no
        user            = root
        server          = /usr/libexec/tangdw
        server_args     = /usr/share/tang/cache
        log_on_success  += USERID
        log_on_failure  += USERID
        disable         = no
    }
   ```    
- Generate keys:
    ```sh
    /usr/libexec/tangd-keygen /usr/share/tang/db
    /usr/libexec/tangd-update /usr/share/tang/db /usr/share/tang/cache
    service xinetd reload
   ```    
- Check log
    ```sh
    cat /var/log/tangd.log
    ```    

### Resources
- https://openwrt.org/
- https://github.com/latchset/tang
- https://moin.meidokon.net/furinkan/sysadmin/Clevis_and_Tang
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/configuring-automated-unlocking-of-encrypted-volumes-using-policy-based-decryption_security-hardening


[TOC](https://github.com/mtreml/diy-nas/blob/main/README.md#table-of-contents)
---


# Software

This guide follows the documentation of OMV version 5.x since 6.x is still under development.

## Installation and initial configuration of OpenMediaVault 6.0-16
- Prepare two installation media: 1. ISO to boot from and 2. formated with ext4 to install to
- Installation: follow the official documentation at https://openmediavault.readthedocs.io/en/5.x/new_user_guide/newuserguide.html#amd64-64-bit-platforms
- Initial configuration of OMV: follow the official documentation at https://openmediavault.readthedocs.io/en/5.x/new_user_guide/newuserguide.html#initial-configuration

### Login via web console
- Change the admin password
- Change auto logout time interval to 30 min: `System > Workbench > Auto logout`
- Configure network interface: `Network > Interfaces > eth0`
    ```
    IPv4: 192.168.x.xxx
    Netmask: 255.255.255.0
    Gateway: 192.168.x.xxx
    DNS-Server: 8.8.8.8
    [x] wake on LAN
    ```
- Create a NAS user: `User Management > Users > + > nas_user`
- **Careful:** adding/removing public ssh keys via the omv web console will change `sshd_config`!

### Login via SSH
- Generate & copy new public ssh key from the client machine to the NAS server
    ```bash
    ssh-keygen -t ed25519
    ssh-copy-id nas_user@192.168.x.xxx
    ```
- Login via ssh
    ```sh
    ssh nas_user@192.168.x.xxx
    ```
- Update OMV
    ```sh
    sudo -i
    nano /etc/locale.gen
    locale-gen
    apt-get update && apt-get dist-upgrade -y && omv-upgrade -y
    ```
- Install additional packages
    ```sh
    apt-get install -y screen htop
    ```
- Install omv-extras to have access to more plugins
    ```sh
    wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash 
    ```
- Install OMV plugins
    - clam-av
    - openmediavault-flashmemory


## Encrypted data drive

The following scheme is used for the data drives: `RAID --> LUKS --> LVM --> ext4`.

- Having RAID as bottom layer has the advantage that a disk can be easily swapped in case of failure.
- Putting LUKS as second layer allows you to have only one password.
- Adding LVM as a third layer offers flexibility for future shrinking/extending. 


### Partitions of same size on every physical disk

- Wipe disks before doing anything else
    ```sh
    haveged -n 0 | dd of=/dev/sd[abcd]
    ```

### RAID device on sda, sdb, sdc, sdd

- Create RAID array: `Storage > RAID Management > +`

- Save the RAID configuration
    ```sh
    mdadm --detail --scan >> /etc/mdadm/mdadm.conf
    ```

### Encrypted LUKS container on the RAID device

- Install
    ```sh
     sudo apt install -y cryptsetup
    ```

- Benchmark 
    ```sh
    cryptsetup benchmark
    ```
  This gives 1555.8 MiB/s (encryption) and 1652.4 MiB/s (decryption) for aes-xts with 256b key.
 
- Create a LUKS container
    ```sh
    cryptsetup luksFormat --cipher aes-xts-plain64 --hash sha512 -s 512 /dev/md0
    ```
- Show info
    ```sh
    cryptsetup luksDump /dev/md0
    ```
- Open the new LUKS container
    ```sh
    cryptsetup luksOpen /dev/md0 crypt-raid
    ```
- Find UUID of /dev/md0
    ```sh
    blkid
    ```
- Edit `/etc/crypttab` and add
    ```sh
    crypt-raid UUID=xxxx none luks
    ```
- Recreate initramfs
    ```sh
    update-initramfs -u -k all
    ```
- Change GRUB_TIMEOUT to 1s
   ```sh
   nano /etc/default/grub
   update-grub
   ``` 
- Purge old kernels
   ```sh
   uname -r
   apt remove --purge linux-image-
   ``` 
   
### LVM

- Specify md0 as physical volume (make sure dataalignment matches chunk size of RAID)
    ```sh
    pvcreate --dataalignment=512 /dev/mapper/crypt-raid
    ```
- Create new volume group
    ```sh
    vgcreate raid-vg /dev/mapper/crypt-raid
    ```
- Create logical volume that uses the whole free disk space
    ```sh
    lvcreate -l 100%FREE --name mrm raid-vg
    ```

### File system

- Create EXT4 filesystem (this takes a while): `Storage > File Systems > + > Create > mrm-raid-vg, ext4`
    
### Clevis


- Install
    ```sh
    sudo apt install -y clevis clevis-luks clevis-dracut
    ```

- Check if the tang server is responding
    ```sh
    echo hi | clevis encrypt tang '{"url": "192.168.x.xxx:8888"}' > hi.jwe
    ```
- List and unbind existing binds
    ```sh
    clevis luks list -d /dev/md0
    clevis luks unbind -d /dev/md0 -s 1
    ``` 
- Bind the device to tang
    ```sh
    clevis luks bind -d /dev/md0 tang '{"url": "192.168.x.xxx:8888"}'
    ``` 
 - Update initial RAM filesystem using dracut
    ```sh
    dracut -fv --regenerate-all
    ``` 
    
### Resources

- https://superuser.com/questions/1193290/best-order-of-raid-lvm-and-luks
- https://github.com/gandalfb/openmediavault-full-disk-encryption
- https://thelinuxchronicles.blogspot.com/2014/04/encrypted-software-raid-5-on-debian.html
- https://github.com/latchset/clevis
- https://moin.meidokon.net/furinkan/sysadmin/Clevis_and_Tang 


## I/O benchmarks

- Hard Disk Drive (HDD) bays 1 and 2 support 6.0 Gb/s = 750 MB/s SATA
- HDD bays 3 and 4 support 3.0 Gb/s = 375 MB/s SATA
- Testfile has ~32 GB
- Local copies
  - Read: `dd if=/srv/disk-by-uid/share/testfile.tar.gz of=/dev/null bs=1M`
  - Write: `dd if=/dev/zero of=/srv/disk-by-uid/share/testfile_1.tar.gz bs=1M count=32000 conv=sync`
- Network (CIFS/NFS) via 1 Gb/s = 125 MB/s ethernet port
  - Read: `dd if=/mnt/share/testfile.tar.gz of=/dev/null bs=1M`
  - Write: `dd if=/dev/zero of=/mnt/share/testfile_1.tar.gz bs=1M count=32000 conv=sync`


|       | RAID5    | RAID5+LUKS+LVM | RAID5*+LUKS+LVM | RAID5 - NFS | RAID5 - CIFS |
|-------|----------|----------------|-----------------|-------------|--------------|
| Read  | 356 MB/s | 350 MB/s       | 375 MB/s        | 86.30 MB/s  | 85.3 MB/s    |
| Write | 336 MB/s | 368 MB/s       | 368 MB/s        | 43.60 MB/s  | 87.8 MB/s    |

*: with stripe_cache_size=8192

This is more or less the maximum performance the hardware controller allows since the RAID array uses all four HDD bays.

### Resources
- https://louwrentius.com/zfs-performance-on-hp-proliant-microserver-gen8-g1610t.html
- https://superuser.com/questions/305716/bad-performance-with-linux-software-raid5-and-luks-encryption



[TOC](https://github.com/mtreml/diy-nas/blob/main/README.md#table-of-contents)
---


# Secure your server

## Force strong user passwords
    ```sh
    sudo apt install -y libpam-pwquality
    sudo cp --archive /etc/pam.d/common-password /etc/pam.d/common-password-COPY-$(date +"%Y%m%d%H%M%S")
    sudo sed -i -r -e "s/^(password\s+requisite\s+pam_pwquality.so)(.*)$/# \1\2         # commented by $(whoami) on $(date +"%Y-%m-%d @ %H:%M:%S")\n\1 retry=3 minlen=10 difok=3 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 maxrepeat=3 gecoschec         # added by $(whoami) on $(date +"%Y-%m-%d @ %H:%M:%S")/" /etc/pam.d/common-password
    ```
## Enable unattended updates
    ```sh
    sudo apt install -y unattended-upgrades apt-listchanges apticron
    nano /etc/apt/apt.conf.d/51myunattended-upgrades    
    ```
    
    ```sh
    // Enable the update/upgrade script (0=disable)
    APT::Periodic::Enable "1";

    // Do "apt-get update" automatically every n-days (0=disable)
    APT::Periodic::Update-Package-Lists "1";

    // Do "apt-get upgrade --download-only" every n-days (0=disable)
    APT::Periodic::Download-Upgradeable-Packages "1";

    // Do "apt-get autoclean" every n-days (0=disable)
    APT::Periodic::AutocleanInterval "7";

    // Send report mail to root
    //     0:  no report             (or null string)
    //     1:  progress report       (actually any string)
    //     2:  + command outputs     (remove -qq, remove 2>/dev/null, add -d)
    //     3:  + trace on    APT::Periodic::Verbose "2";
    APT::Periodic::Unattended-Upgrade "1";

    // Automatically upgrade packages from these
    Unattended-Upgrade::Origins-Pattern {
          "o=Debian,a=stable";
          "o=Debian,a=stable-updates";
          "origin=Debian,codename=${distro_codename},label=Debian-Security";
    };

    // You can specify your own packages to NOT automatically upgrade here
    Unattended-Upgrade::Package-Blacklist {
    };

    // Run dpkg --force-confold --configure -a if a unclean dpkg state is detected to true to ensure that updates get installed even when the system got interrupted during a previous run
    Unattended-Upgrade::AutoFixInterruptedDpkg "true";

    //Perform the upgrade when the machine is running because we wont be shutting our server down often
    Unattended-Upgrade::InstallOnShutdown "false";

    // Send an email to this address with information about the packages upgraded.
    Unattended-Upgrade::Mail "root";

    // Always send an e-mail
    Unattended-Upgrade::MailOnlyOnError "false";

    // Remove all unused dependencies after the upgrade has finished
    Unattended-Upgrade::Remove-Unused-Dependencies "true";

    // Remove any new unused dependencies after the upgrade has finished
    Unattended-Upgrade::Remove-New-Unused-Dependencies "true";

    // Automatically reboot WITHOUT CONFIRMATION if the file /var/run/reboot-required is found after the upgrade.
    Unattended-Upgrade::Automatic-Reboot "true";

    // Automatically reboot even if users are logged in.
    Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
    ```
    
    ```sh
    sudo unattended-upgrade -d --dry-run
    ```

## Limit access to sudo & su
- Create user group sudo (already present in Debian) & add users
    ```sh
    sudo usermod -a -G sudo nas_user
    ```
- Create user group suusers & add users
    ```sh
    sudo groupadd suusers
    sudo usermod -a -G suusers root
    sudo usermod -a -G suusers nas_user
    sudo dpkg-statoverride --update --add root suusers 4750 /bin/su
    ```

## SSH
- **Careful:** adding/removing public ssh keys via the omv web console will change `sshd_config`!

- Create SSH user group for AllowGroups (already present in Debian) & add users
    ```sh
    sudo groupadd ssh
    sudo usermod -a -G ssh nas_user
    ```
- Edit ssh settings
    ```sh
    nano /etc/ssh/sshd_config
    sudo service sshd restart
    sudo sshd -T
    ```
- Remove short moduli
    ```sh
    sudo cp --archive /etc/ssh/moduli /etc/ssh/moduli-COPY-$(date +"%Y%m%d%H%M%S")
    sudo awk '$5 >= 3071' /etc/ssh/moduli | sudo tee /etc/ssh/moduli.tmp
    sudo mv /etc/ssh/moduli.tmp /etc/ssh/moduli    
    ```
    
## Network
- Firewall
    ```sh
    sudo apt install -y ufw

    sudo ufw default deny outgoing comment 'deny all outgoing traffic'
    sudo ufw default deny incoming comment 'deny all incoming traffic'
    sudo ufw allow in 22/tcp comment 'allow incoming SSH'
    sudo ufw allow out 53 comment 'allow DNS calls out'
    sudo ufw allow out 123 comment 'allow NTP out'
    sudo ufw allow out http comment 'allow HTTP traffic out'
    sudo ufw allow in http comment 'allow HTTP traffic in'
    sudo ufw allow out https comment 'allow HTTPS traffic out'
    sudo ufw allow in https comment 'allow HTTPS traffic in'
    sudo ufw allow out 445/tcp comment 'allow SMB traffic out'
    sudo ufw allow in 445/tcp comment 'allow SMB traffic in'
    sudo ufw allow out 8888/tcp comment 'allow tang traffic out'
    sudo ufw allow in 8888/tcp comment 'allow tang traffic in'

    sudo ufw enable
    
    sudo ufw status
   
    ```
- Iptables ntrusion detection with psad
    ```sh
    sudo apt install -y psad
    sudo cp --archive /etc/psad/psad.conf /etc/psad/psad.conf-COPY-$(date +"%Y%m%d%H%M%S")
    nano /etc/psad/psad.conf

    sudo cp --archive /etc/ufw/before.rules /etc/ufw/before.rules-COPY-$(date +"%Y%m%d%H%M%S")
    sudo cp --archive /etc/ufw/before6.rules /etc/ufw/before6.rules-COPY-$(date +"%Y%m%d%H%M%S")
    ```
    Add
    ```sh
    # log all traffic so psad can analyze
    -A INPUT -j LOG --log-tcp-options --log-prefix "[IPTABLES] "
    -A FORWARD -j LOG --log-tcp-options --log-prefix "[IPTABLES] "
    ```
    to `/etc/ufw/before.rules` and `/etc/ufw/before6.rules` at the end of the file before COMMIT.

    ```sh
    sudo ufw reload
    sudo psad -R
    sudo psad --sig-update
    sudo psad -H
    sudo psad --fw-analyze
    sudo psad --Status
    ```
- App intrusion detection with fail2ban
    ```sh
    sudo apt install -y fail2ban
    sudo nano /etc/fail2ban/jail.local
    ```
    Add
    ```sh
    [DEFAULT]
    # the IP address range we want to ignore
    ignoreip = 127.0.0.1/8 [LAN SEGMENT]

    # who to send e-mail to
    destemail = [your e-mail]

    # who is the email from
    sender = [your e-mail]

    # since we're using exim4 to send emails
    mta = mail

    # get email alerts
    action = %(action_mwl)s
    ```
    ```sh
    cat << EOF | sudo tee /etc/fail2ban/jail.d/ssh.local
    [sshd]
    enabled = true
    banaction = ufw
    port = ssh
    filter = sshd
    logpath = %(sshd_log)s
    maxretry = 5
    EOF
    ```
    ```sh
    sudo fail2ban-client start
    sudo fail2ban-client reload
    sudo fail2ban-client add sshd # This may fail on some systems if the sshd jail was added by default
    sudo fail2ban-client status
    sudo fail2baclevis luks unbind -d /dev/sda2 -s 1n-client status sshd
    ```


## Resources
- https://github.com/imthenachoman/How-To-Secure-A-Linux-Server


[TOC](https://github.com/mtreml/diy-nas/blob/main/README.md#table-of-contents)
---


# Administration

## Create a shared folder

- New shared folder: `Storage > Shared Folders > + > /dev/md0 > Everyone: read/write`

- SMB/CIFS service: `Services > SMB/CIFS > Shares > + > ` with
  - `Guests allowed`
  - [x] `Inherit ACLs`
  - [x] `Inherit permissions`
  - [x] `Extended attributes`
  - [x] `Store DOS attributes`

- NFS service: `Services > NFS > Shares > + > ` with
  - Client IP: 192.168.x.xxx
  - Read/Write

## Mount smb/cifs share on the (Ubuntu) client

- Install cifs-utils on 
    ```sh
    sudo apt install -y cifs-utils
    ```
- Credentials file `/root/.smbcredentials`
    ```sh
    username=nas_user
    password=yourpassword
    domain=WORKGROUP
    ```
    ```sh
    sudo chmod 600 /root/.smbcredentials
    ```

- Add a line to `/etc/fstab`
    ```sh
    //192.168.x.xxx/<share> /media/share cifs credentials=/root/.smbcredentials,uid=ubuntu_user,noperm,rw,vers=3.0 0 0
    ```
- Create mount point
    ```sh
    sudo mkdir /media/share
    ```
- Mount
    ```sh
    sudo mount -a
    ```

## Mount NFS share in Ubuntu

- Install NFS client
    ```sh
    sudo apt-get install -y nfs-common
    ```
- Create mount point
    ```sh
    sudo mkdir /media/share_nfs
    ```
- Mount
    ```sh
    sudo mount 192.168.x.xxx:/<share> /media/share_nfs
    ```


[TOC](https://github.com/mtreml/diy-nas/blob/main/README.md#table-of-contents)
---

# Backup

## Data drive
- Implement a backup policy

## OS
- Create a backup of the OS drive
    ```sh
    sudo dd if=/dev/sdcard of=~/Downloads/NAS/omv6_sdcard_backup.img bs=1M status=progress
    ```
- Test restoring it
    ```sh
    sudo dd bs=4M if=~/Downloads/NAS/omv6_sdcard_backup.img of=/dev/sdb status=progress
    ```
