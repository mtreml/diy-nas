# Goals

- Build a DIY NAS just for the fun of it
- RAID5 for read performance & failure safety (parity)
- Full disk encryption
- < 10 users
- Server will shutdown frequently for energy saving



# Table of contents

- [Goals](#goals)
- [Hardware](#hardware)
  - [HPE Proliant Microserver Gen8](#hpe-proliant-microserver-gen8)
  - [LTE Modem / Router](#lte-modem--router)
- [Software](#software)
  - [Installation and initial configuration of OpenMediaVault (OMV) 6](#installation-and-initial-configuration-of-openmediavault-omv-6)
  - [Encrypted data drive](#encrypted-data-drive)
    - [Partitions of same size on every physical disk](#partitions-of-same-size-on-every-physical-disk)
    - [RAID device on sda, sdb, sdc, sdd](#raid-device-on-sda-sdb-sdc-sdd)
    - [Encrypted LUKS container on the RAID device](#encrypted-luks-container-on-the-raid-device)
    - [LVM](#lvm)
    - [File system](#file-system)
    - [Resources](#resources)
  - [Benchmarks](#benchmarks)
    - [Resources](#resources-1)
- [Secure your server](#secure-your-server)
  - [SSH](#ssh)
  - [Resources](#resources-2)
- [Administration](#administration)
  - [Create a shared folder](#create-a-shared-folder)
  - [Mount smb/cifs share in Ubuntu](#mount-smbcifs-share-in-ubuntu)
  - [Mount NFS share in Ubuntu](#mount-nfs-share-in-ubuntu)


TOC created with https://imthenachoman.github.io/nGitHubTOC/


---


# Hardware

## HPE Proliant Microserver Gen8

See the official specifications at https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c03793258#N1048F

My Hardware
- Intel Xeon E3-1220LV2 (17 W)
- 8 GB RAM
- 16 GB SanDisk A1 micro SD-card for OS (OS-drive on internal slot)
- 4 x 4 TB Western Digital Red Plus (data-drives on SATA slots)

BIOS Update v2014.06.06 -> v2019.04.04
- Get the firmware file *SP99427.exe* at https://support.hpe.com/connect/s/search
- Create USB-Key on Windows & boot server from it.

iLO Firmware Update v2.10 (15.1.2015) -> v2.78 (22.4.2021)
- Get the firmware file *cp046467.exe* at https://support.hpe.com/connect/s/search
- Connect the server at the iLO network interface
- Login into the iLO web interface via IPv6 (credentials on the sticker at the backside of the microserver)
- Upload new firmware and flash it.

Get an ILO Advanced Key and enter it in the iLO web interface

## LTE Modem / Router

- Set a static IP outside of dhcp range for your server machine in your router's web console
      ```
      IPv4: 192.168.x.xxx
      Mac-address: xx:xx:xx:xx:xx:xx
      ```
- Get a real public IP from ISP for your LTE sim card (if it does not have one)
- Use a DDNS service to map a domain name to your changing IP
- Configure your router for DDNs


---


# Software

This guide follows the documentation of OMV version 5.x since 6.x is still under development.

## Installation and initial configuration of OpenMediaVault (OMV) 6
- Installation: follow the official documentation at https://openmediavault.readthedocs.io/en/5.x/new_user_guide/newuserguide.html#amd64-64-bit-platforms
- Initial configuration of OMV: follow the official documentation at https://openmediavault.readthedocs.io/en/5.x/new_user_guide/newuserguide.html#initial-configuration
- Login to the OMV web console
- Change the admin password
- Change auto logout time interval to 30 min
- Update OMV
    ```sh
    apt-get update && apt-get dist-upgrade && omv-update
    ```
- Install additional packages
    ```sh
    apt-get apt-get install -y screen htop
    ```
- Install omv-extras to have access to more plugins
    ```sh
    wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash 
    ```
- Install OMV plugins
    - a
- Install flash memory plugin: https://openmediavault.readthedocs.io/en/5.x/new_user_guide/newuserguide.html?highlight=flashmemory#the-flash-memory-plugin-amd64-users-only
- Configure network interface in OMV's web console
    ```
    IPv4: 192.168.x.xxx
    Netmask: 255:255:255:0
    Gateway: 192.168.x.xxx
    DNS-Server: 8.8.8.8
    [x] wake on LAN
    ```
    
## Encrypted data drive

The following scheme is used for the data drives: `RAID --> LUKS --> LVM --> ext4`.

- Having RAID as bottom layer has the advantage that a disk can be easily swapped in case of failure.
- Putting LUKS as second layer allows you to have only one password.
- Adding LVM as a third layer offers flexibility for future shrinking/extending. 


### Partitions of same size on every physical disk

- Wipe disks before doing anything else
    ```sh
    sth
    ```

- Create
    ```sh
    sth
    ```

### RAID device on sda, sdb, sdc, sdd

- Create
    ```sh
    screen -d -m mdadm --create --verbose --level=5 --metadata=1.2 --raid-devices=4 /dev/md0 /dev/sd[abcd]1
    ```
- Watch the process:  this takes ~ 5 h
    ```sh
    mdadm --readwrite /dev/md0
    ```
- Save the RAID configuration
    ```sh
    mdadm --detail --scan >> /etc/mdadm/mdadm.conf
    ```

### Encrypted LUKS container on the RAID device

- Create a LUKS container
    ```sh
     cryptsetup luksFormat --cipher aes-xts-plain64 --hash sha512 /dev/md0
    ```
- Open the new LUKS container
    ```sh
    cryptsetup luksDump /dev/md0
    ```
- Get crypt setup
    ```sh
    cryptsetup luksDump /dev/md0
    ```
    
### LVM

- Create
    ```sh
    sth
    ```

### File system

- `Storage > File Systems > + > Create`

- EXT4 on RAID device: this takes ~ 10 mins
    ```sh
    screen -d -m mkfs.ext4 -b 4096 -m 0 -E lazy_itable_init=0,lazy_journal_init=0 -O 64bit /dev/mapper/crypt-raid
    ```
- Verify
    ```sh
    lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
    sda      8:0    0  3.6T  0 disk  
    `-md0    9:0    0 10.9T  0 raid5 
    sdb      8:16   0  3.6T  0 disk  
    `-md0    9:0    0 10.9T  0 raid5 
    sdc      8:32   0  3.6T  0 disk  
    `-md0    9:0    0 10.9T  0 raid5 
    sdd      8:48   0  3.6T  0 disk  
    `-md0    9:0    0 10.9T  0 raid5 
    sde      8:64   0 14.8G  0 disk  
    |-sde1   8:65   0 13.9G  0 part  /
    |-sde2   8:66   0    1K  0 part  
    `-sde5   8:69   0  976M  0 part  [SWAP]
    ```
### Resources

- https://superuser.com/questions/1193290/best-order-of-raid-lvm-and-luks
- https://github.com/gandalfb/openmediavault-full-disk-encryption
- https://www.paulligocki.com/open-media-vault-essentials/#Set-up-LUKS-Encrypted-Volume-Using-Plugin
- https://thelinuxchronicles.blogspot.com/2014/04/encrypted-software-raid-5-on-debian.html


## Benchmarks

- Hard Disk Drive (HDD) bays 1 and 2 support 6.0 Gb/s = 750 MB/s SATA
- HDD bays 3 and 4 support 3.0 Gb/s = 375 MB/s SATA
- Copied via 1 Gb/s = 125 MB/s ethernet port
- Testfile has ~32 GB
- Read: `dd if=/mnt/share/testfile.tar.gz of=/dev/null bs=1M`
- Write: `dd if=/dev/zero of=/mnt/share/testfile_1.tar.gz bs=1M count=32000 conv=sync`

|       | RAID5    | RAID5 - NFS | RAID5 - CIFS |
|-------|----------|-------------|--------------|
| Read  | 356 MB/s | 50.79 MB/s  | 85.3 MB/s    |
| Write | 336 MB/s | 14.80 MB/s  | 87.8 MB/s    |

### Resources
- https://louwrentius.com/zfs-performance-on-hp-proliant-microserver-gen8-g1610t.html


---


# Secure your server

## SSH

- Copy public ssh key to omv
    ```bash
    ssh-copy-id root@192.168.x.xxx
    ```
- Login via ssh
    ```sh
    ssh root@192.168.x.xxx
    ```
- Edit ssh settings
    ```sh
    nano /etc/ssh/sshd_config
    ```
    
## Resources
- https://github.com/imthenachoman/How-To-Secure-A-Linux-Server


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

## Mount smb/cifs share in Ubuntu

**On the NAS server:**

Create a NAS user: `User Management > Users > + > nas_user`
- Add to groups `sudo`, `ssh`
- Add public ssh key(s) of the Ubuntu client

**On the Ubuntu client:**

- Install cifs-utils on 
    ```sh
    sudo apt install cifs-utils
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
    //192.168.x.xxx/share /media/share cifs credentials=/root/.smbcredentials,uid=ubuntu_user,noperm,rw 0 0
    ```

## Mount NFS share in Ubuntu

**On the Ubuntu client:**
- Install NFS client
    ```sh
    sudo apt-get install nfs-common
    ```
- Mount
    ```sh
    sudo mount 192.168.8.99:/share /media/share_nfs
    ```
