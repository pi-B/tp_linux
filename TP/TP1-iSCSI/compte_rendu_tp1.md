# TP1 Introduction au réseau de stockage iSCSI

### Eleve : Pierre BAYLE
### Date : 29/08/2023

## Partie 1 

Création des machines virtuelles, target sur le port 428 et initiator sr le 429

```
08:03:34 pierrebayle@bob vm → bash startup.sh -v 230 -i 428 -t 429
~> iSCSI lab VLAN identifier: 230
~> Target VM tap port number: 429
~> Initiator VM tap port number: 428
Configuring tap429 port...
Configuring tap428 port...
~> Virtual machine filename   : target_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6329
~> telnet console port number : 2729
~> MAC address                : b8:ad:ca:fe:01:ad
~> Switch port interface      : tap429, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1ad%vlan230
~> Virtual machine filename   : initiator_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6328
~> telnet console port number : 2728
~> MAC address                : b8:ad:ca:fe:01:ac
~> Switch port interface      : tap428, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1ac%vlan230

```
On se connecte en telnet sur localhost [portVM] et on change les adresses IP :
Sur Corellia on a 10.0.228.1/22, je choisi les deux dernieres adresse 10.0.228.253 et 10.0.228.254

```
etu@vm0:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:01:ac brd ff:ff:ff:ff:ff:ff
    inet6 fe80::baad:caff:fefe:1ac/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever

etu@vm0:~$ sudo ifdown --force enp0s1
[sudo] Mot de passe de etu : 
Killed old client process
Internet Systems Consortium DHCP Client 4.4.3-P1
Copyright 2004-2022 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/enp0s1/b8:ad:ca:fe:01:ac
Sending on   LPF/enp0s1/b8:ad:ca:fe:01:ac
Sending on   Socket/fallback
DHCPRELEASE of 198.18.69.0 on enp0s1 to 198.18.68.1 port 67
send_packet: Network is unreachable
send_packet: please consult README file regarding broadcast address.
dhclient.c:3124: Failed to send 300 byte long packet over fallback interface.

# The primary network interface
allow-hotplug enp0s1
iface enp0s1 inet static
        address 10.0.228.253/22
        gateway 10.0.228.1
# This is an autoconfigured IPv6 interface
iface enp0s1 inet6 auto

etu@vm0:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:01:ac brd ff:ff:ff:ff:ff:ff
    inet 10.0.228.253/22 brd 10.0.231.255 scope global enp0s1
       valid_lft forever preferred_lft forever
    inet6 2001:678:3fc:e6:baad:caff:fefe:1ac/64 scope global dynamic mngtmpaddr proto kernel_ra 
       valid_lft 2591975sec preferred_lft 604775sec
    inet6 fe80::baad:caff:fefe:1ac/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```
=====================================================================

## Mise en route du service iscsi sur l'initiateur

On installe le paquet **open-iscsi** et on le redémarre sur l'initiator. On constate qu'on n'a pas démarré de session, la relation avec la target n'est pas établi et on le vérifie dans les logs :

```
etu@initiator:~$ sudo systemctl restart open-iscsi.service 
etu@initiator:~$ systemctl status open-iscsi.service 
○ open-iscsi.service - Login to default iSCSI targets
     Loaded: loaded (/lib/systemd/system/open-iscsi.service; enabled; preset: e>
     Active: inactive (dead)
  Condition: start condition unmet at Sat 2023-09-02 12:35:56 CEST; 7s ago
             ├─ ConditionDirectoryNotEmpty=|/etc/iscsi/nodes was not met
             └─ ConditionDirectoryNotEmpty=|/sys/class/iscsi_session was not met
       Docs: man:iscsiadm(8)
             man:iscsid(8)
etu@initiator:~$ sudo grep -i iscsi /var/log/syslog 
2023-09-01T20:16:02.437259+02:00 initiator systemd[1]: open-iscsi.service - Login to default iSCSI targets was skipped because no trigger condition checks were met.
2023-09-02T12:21:51.240758+02:00 initiator systemd[1]: open-iscsi.service - Login to default iSCSI targets was skipped because no trigger condition checks were met.
2023-09-02T12:35:56.575677+02:00 initiator systemd[1]: open-iscsi.service - Login to default iSCSI targets was skipped because no trigger condition checks were met.

```

## Configuration de la target 

### Préparation de l'espace de stockage 

Quelle est la commande apparentée à ls qui permet d'obtenir la liste des périphériques de
stockage en mode bloc ?

-> lsblk 

```
etu@vm0:~$ lsblk 
NAME   MAJ:MIN RM  SIZE R   O TYPE MOUNTPOINTS
sr0     11:0    1    2K  0 rom  
vda    254:0    0   60G  0 disk 
├─vda1 254:1    0  512M  0 part /boot/efi
├─vda2 254:2    0 58,5G  0 part /
└─vda3 254:3    0  976M  0 part [SWAP]
vdb    254:16   0   32G  0 disk
```

```
etu@vm0:~$ sudo parted /dev/vda print
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 64,4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system     Name  Flags
 1      1049kB  538MB   537MB   fat32                 boot, esp
 2      538MB   63,4GB  62,9GB  ext4
 3      63,4GB  64,4GB  1023MB  linux-swap(v1)        swap

etu@vm0:~$ sudo dd if=/dev/zero of=/dev/vdb bs=512 count=4
4+0 enregistrements lus
4+0 enregistrements écrits
2048 octets (2,0 kB, 2,0 KiB) copiés, 0,0227578 s, 90,0 kB/s

```
### Mise en route de la target ISCSI

A partir du paquet targetcli nous allons paramétrer la target ISCSI pour :
- lier un volume de stockage à la target
- définir la target ISCSI 
- rattacher le volume et la target 
- publier cette target sur le système hote

On vérifie ensuite depuis l'initiator que le service est disponible en renseignant l'adresse IP de la target.

```
etu@target:~$ sudo apt install targetcli-fb
etu@target:~$ sudo targetcli
Warning: Could not load preferences file /root/.targetc  li/prefs.bin.
targetcli shell version 2.1.53
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> cd backstores/block 
/backstores/block> create blocvol0 /dev/vdb
Created block storage object blocvol0 using /dev/vdb.
/backstores/block> ls
o- block .................................................. [Storage Objects: 1]
  o- blocvol0 ...................... [/dev/vdb (32.0GiB) write-thru deactivated]
    o- alua ................................................... [ALUA Groups: 1]
      o- default_tg_pt_gp ....................... [ALUA state: Active/optimized]


/backstores/block> cd ../fileio 
/backstores/fileio> ls
o- fileio ................................................. [Storage Objects: 0]
/backstores/fileio> create filevol0 /var/cache/filevol0 32G
Created fileio filevol0 with size 34359738368
/backstores/fileio> ls
o- fileio ................................................. [Storage Objects: 1]
  o- filevol0 ........... [/var/cache/filevol0 (32.0GiB) write-back deactivated]
    o- alua ................................................... [ALUA Groups: 1]
      o- default_tg_pt_gp ....................... [ALUA state: Active/optimized]

/> cd iscsi 
/iscsi> create
Created target iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/iscsi> ls
o- iscsi .......................................................... [Targets: 1]
  o- iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886 ........ [TPGs: 1]
    o- tpg1 ............................................. [no-gen-acls, no-auth]
      o- acls ........................................................ [ACLs: 0]
      o- luns ........................................................ [LUNs: 0]
      o- portals .................................................. [Portals: 1]
        o- 0.0.0.0:3260 ................................................... [OK]

/iscsi/iqn.20...886/tpg1/luns> create /backstores/block/blocvol0 
Created LUN 0.
/iscsi/iqn.20...886/tpg1/luns> ls
o- luns .............................................................. [LUNs: 1]
  o- lun0 ....................... [block/blocvol0 (/dev/vdb) (default_tg_pt_gp)]

/iscsi/iqn.20.../tpg1/portals> delete 0.0.0.0 3260
Deleted network portal 0.0.0.0:3260
/iscsi/iqn.20.../tpg1/portals> create ::0
Using default IP port 3260
Created network portal ::0:3260.
/iscsi/iqn.20.../tpg1/portals> ls
o- portals ........................................................ [Portals: 1]
  o- [::0]:3260 ........................................................... [OK]

etu@target:~$ ss -tan '( sport = :3260 )'
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port  Process  
LISTEN   0        256                    *:3260                *:*              

etu@initiator:~$ sudo iscsiadm -m discovery --type sendtargets --portal=10.0.228.253
10.0.228.253:3260,1 iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886

```

## Création d'un session initiator - target

Une fois que nous avons vérifier que le service est disponible depuis l'initiator nous essayons d'ouvrir une session

```
etu@initiator:~$ sudo iscsiadm -m node \
> -T iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886 \
> -p 10.0.228.253 \
> -l
[sudo] Mot de passe de etu : 
Logging in to [iface: default, target: iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886, portal: 10.0.228.253,3260]
iscsiadm: Could not login to [iface: default, target: iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886, portal: 10.0.228.253,3260].
iscsiadm: initiator reported error (24 - iSCSI login failed due to authorization failure)
iscsiadm: Could not log into all portals

etu@target:~$ sudo grep -i iscsi /var/log/syslog
2023-09-03T14:34:24.665086+02:00 target kernel: [150572.134525] iSCSI Initiator Node: iqn.1993-08.org.debian:01:2aabe42616 is not authorized to access iSCSI target portal group: 1.
2023-09-03T14:34:24.665086+02:00 target kernel: [150572.135495] iSCSI Login negotiation failed.
2023-09-03T14:35:04.603827+02:00 target kernel: [153052.875677] iSCSI Initiator Node: iqn.1993-08.org.debian:01:2aabe42616 is not authorized to access iSCSI target portal group: 1.
2023-09-03T14:35:04.603878+02:00 target kernel: [153052.876906] iSCSI Login negotiation failed.
2023-09-03T14:36:04.335734+02:00 target kernel: [153112.610284] iSCSI Initiator Node: iqn.1993-08.org.debian:01:2aabe42616 is not authorized to access iSCSI target portal group: 1.
2023-09-03T14:36:04.335847+02:00 target kernel: [153112.611217] iSCSI Login negotiation failed.
```

Il n'est pas possible de se connecter à cause d'une erreur de droits.  Nous allons donc devoir ajouter l'identité  iscsi de l'initiator dans la liste des clients autorisés à accéder le LUN :

```
etu@initiator:~$ sudo grep -v ^# /etc/iscsi/initiatorname.iscsi 
InitiatorName=iqn.1993-08.org.debian:01:2aabe42616

/iscsi/iqn.20...886/tpg1/acls> ls
o- acls .............................................................. [ACLs: 0]
/iscsi/iqn.20...886/tpg1/acls> create iqn.1993-08.org.debian:01:2aabe4261
Created Node ACL for iqn.1993-08.org.debian:01:2aabe42616
Created mapped LUN 0.

etu@initiator:~$ sudo iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886 -p 10.0.228.253 -l
Logging in to [iface: default, target: iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886, portal: 10.0.228.253,3260]
Login to [iface: default, target: iqn.2003-01.org.linux-iscsi.target.x8664:sn.32e29cb5f886, portal: 10.0.228.253,3260] successful.
```

Nous avons réussi à ajouter l'IQN de l'initiator à la liste des clients autorisés à joindre la target et nous observons que le noeud ISCSI est bien joignable depuis ce dernier.

## RAID1

```
etu@initiator:~$ sudo mdadm --create /dev/md0 --level=raid1 --raid-devices=2 /dev/sda /dev/vdb
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: partition table exists on /dev/vdb
mdadm: partition table exists on /dev/vdb but will be lost or
       meaningless after creating array
Continue creating array? 
Continue creating array? (y/n) y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.


etu@initiator:~$ cat /proc/mdstat 
Personalities : [raid1] 
md0 : active raid1 vdb[1] sda[0]
      33520640 blocks super 1.2 [2/2] [UU]
      
unused devices: <none>
etu@initiator:~$ sudo mdadm /dev/
Display all 174 possibilities? (y or n)
etu@initiator:~$ sudo mdadm /dev/md0 
/dev/md0: 31.97GiB raid1 2 devices, 0 spares. Use mdadm --detail for more detail.
etu@initiator:~$ sudo mdadm --detail /dev/md0 
/dev/md0:
           Version : 1.2
     Creation Time : Mon Sep  4 09:30:58 2023
        Raid Level : raid1
        Array Size : 33520640 (31.97 GiB 34.33 GB)
     Used Dev Size : 33520640 (31.97 GiB 34.33 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Mon Sep  4 09:34:57 2023
             State : clean 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : initiator:0  (local to host initiator)
              UUID : 23f83ad5:2fa8a09f:f51be04d:145188e5
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8        0        0      active sync   /dev/sda
       1     254       16        1      active sync   /dev/vdb
etu@initiator:~$ sudo mdadm --detail /dev/md0 | sudo tee -a /etc/mdadm/mdadm.conf 
/dev/md0:
           Version : 1.2
     Creation Time : Mon Sep  4 09:30:58 2023
        Raid Level : raid1
        Array Size : 33520640 (31.97 GiB 34.33 GB)
     Used Dev Size : 33520640 (31.97 GiB 34.33 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Mon Sep  4 09:34:57 2023
             State : clean 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : initiator:0  (local to host initiator)
              UUID : 23f83ad5:2fa8a09f:f51be04d:145188e5
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8        0        0      active sync   /dev/sda
       1     254       16        1      active sync   /dev/vdb
```
## Création de snapshot LVM
```
etu@initiator:~$ sudo pvcreate /dev/md0 
  Physical volume "/dev/md0" successfully created.
etu@initiator:~$ sudo pvs
  PV         VG Fmt  Attr PSize   PFree  
  /dev/md0      lvm2 ---  <31,97g <31,97g
etu@initiator:~$ sudo pvdisplay 
  "/dev/md0" is a new physical volume of "<31,97 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/md0
  VG Name               
  PV Size               <31,97 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               euHWCy-GgMa-52Xc-0u1O-BO7w-HQUy-NCOHsY
```

On initialise la partition /dev/md0 :

```
etu@initiator:~$ sudo vgcreate lab-vg /dev/md0
WARNING: sun signature detected on /dev/md0 at offset 508. Wipe it? [y/n]: y
  Wiping sun signature on /dev/md0.
  Physical volume "/dev/md0" successfully created.
  Volume group "lab-vg" successfully created

etu@initiator:~$ sudo lvcreate --size 16Go lab-vg
  Logical volume "lvol0" created.

etu@initiator:~$ sudo mkfs.ext4 /dev/lab-vg/lvol0 
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done                            
Creating filesystem with 4194304 4k blocks and 1048576 inodes
Filesystem UUID: ea7467ac-95f6-45b8-8839-c2f25a9c8fd1
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done   
```

On monte et on accède au nouveau système de fichiers.

```
etu@initiator:~$ sudo parted /dev/md0 print
Error: /dev/md0: unrecognised disk label
Model: Linux Software RAID Array (md)                                     
Disk /dev/md0: 34,3GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
etu@initiator:~$ sudo dd if=/dev/zero of=/dev/md0 bs=512 count=4
4+0 enregistrements lus
4+0 enregistrements écrits
2048 octets (2,0 kB, 2,0 KiB) copiés, 0,00734575 s, 279 kB/s
etu@initiator:~$ sudo parted /dev/md0 print
Error: /dev/md0: unrecognised disk label
Model: Linux Software RAID Array (md)                                     
Disk /dev/md0: 34,3GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
etu@initiator:~$ sudo parted /dev/md0 
GNU Parted 3.6
Using /dev/md0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpr                                                      
parted: invalid token: gpr
New disk label type? ^C
Error: Expecting a disk label type.
(parted) mklabel gpt
(parted) print                                                            
Model: Linux Software RAID Array (md)
Disk /dev/md0: 34,3GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  Flags
(parted) mkpart ext4 0% 100%
(parted) print                                                            
Model: Linux Software RAID Array (md)
Disk /dev/md0: 34,3GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  34,3GB  34,3GB               ext4


```

### Ajout de contenu sur le dossier monté

```
etu@initiator:~$ sudo mkdir /mnt/lvol0
[sudo] Mot de passe de etu : 
etu@initiator:~$ sudo mount /dev/lab-vg/lvol0 /mnt/lvol0/
etu@initiator:~$ mount | grep lvol0
/dev/mapper/lab--vg-lvol0 on /mnt/lvol0 type ext4 (rw,relatime)
etu@initiator:~$ 

etu@initiator:~$ sudo mkdir /mnt/lvol0/etu-files
etu@initiator:~$ sudo chown etu.etu /mnt/lvol0/etu-files/
chown: warning: '.' should be ':': « etu.etu »

etu@initiator:~$ ls -la /mnt/lvol0/
total 28
drwxr-xr-x 4 root root  4096  4 sept. 13:11 .
drwxr-xr-x 3 root root  4096  4 sept. 13:05 ..
drwxr-xr-x 2 etu  etu   4096  4 sept. 13:11 etu-files
drwx------ 2 root root 16384  4 sept. 11:20 lost+found

etu@initiator:~$ touch /mnt/lvol0/etu-files/my-first-file
etu@initiator:~$ ls -la /mnt/lvol0/etu-files/
total 8
drwxr-xr-x 2 etu  etu  4096  4 sept. 13:15 .
drwxr-xr-x 4 root root 4096  4 sept. 13:15 ..
-rw-r--r-- 1 etu  etu     0  4 sept. 13:15 my-first-file

etu@initiator:~$ df -hT
Sys. de fichiers          Type     Taille Utilisé Dispo Uti% Monté sur
udev                      devtmpfs   451M       0  451M   0% /dev
tmpfs                     tmpfs       94M    704K   94M   1% /run
/dev/vda2                 ext4        58G    1,9G   53G   4% /
tmpfs                     tmpfs      470M       0  470M   0% /dev/shm
tmpfs                     tmpfs      5,0M       0  5,0M   0% /run/lock
/dev/vda1                 vfat       511M    5,9M  506M   2% /boot/efi
tmpfs                     tmpfs       94M    8,0K   94M   1% /run/user/1000
/dev/mapper/lab--vg-lvol0 ext4        16G     28K   15G   1% /mnt/lvol0

etu@initiator:~$ lsblk 
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sr0                11:0    1    2K  0 rom   
vda               254:0    0   60G  0 disk  
├─vda1            254:1    0  512M  0 part  /boot/efi
├─vda2            254:2    0 58,5G  0 part  /
└─vda3            254:3    0  976M  0 part  [SWAP]
vdb               254:16   0   32G  0 disk  
└─md0               9:0    0   32G  0 raid1 
  └─lab--vg-lvol0 253:0    0   16G  0 lvm   /mnt/lvol0


```
### Création de deux snapshot avec un nombre de fichiers différents


```
etu@initiator:~$ bash fill_files.sh 
etu@initiator:~$ ls -la /mnt/lvol0/etu-files/
total 8
drwxr-xr-x 2 etu  etu  4096  4 sept. 14:14 .
drwxr-xr-x 4 root root 4096  4 sept. 13:15 ..
-rw-r--r-- 1 etu  etu     0  4 sept. 14:10 first-00-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-01-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-02-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-03-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-04-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-05-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-06-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-07-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-08-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-09-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 first-10-file
-rw-r--r-- 1 etu  etu     0  4 sept. 13:15 my-first-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-01-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-02-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-03-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-04-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-05-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-06-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-07-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-08-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-09-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:14 second-10-file

etu@initiator:~$ sudo lvcreate --snapshot --name second-snap -L 500M /dev/lab-vg/lvol0
  Logical volume "second-snap" created.

etu@initiator:~$ sudo lvs
  LV          VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  first-snap  lab-vg swi-a-s--- 500,00m      lvol0  0,02                                   
  lvol0       lab-vg owi-aos---  16,00g                                                    
  second-snap lab-vg swi-a-s--- 500,00m      lvol0  0,01

etu@initiator:~$ sudo rm -f /mnt/lvol0/etu-files/*
etu@initiator:~$ sudo ls -la /mnt/lvol0/etu-files/
total 8
drwxr-xr-x 2 etu  etu  4096  4 sept. 14:17 .
drwxr-xr-x 4 root root 4096  4 sept. 13:15 ..

etu@initiator:~$ sudo lvconvert --merge /dev/lab-vg/first-snap 
  Delaying merge since origin is open.
  Merging of snapshot lab-vg/first-snap will occur on next activation of lab-vg/lvol0.
etu@initiator:~$ sudo lvchange --activate n lab-vg/lvol0
  Logical volume lab-vg/lvol0 contains a filesystem in use.
etu@initiator:~$ sudo umount /mnt/lvol0 
etu@initiator:~$ sudo lvchange --activate n lab-vg/lvol0
etu@initiator:~$ sudo lvchange --activate y lab-vg/lvol0
etu@initiator:~$ sudo lvscan 
  ACTIVE   Original '/dev/lab-vg/lvol0' [16,00 GiB] inherit
  ACTIVE   Snapshot '/dev/lab-vg/second-snap' [500,00 MiB] inherit

etu@initiator:~$ sudo mount /dev/lab-vg/
lvol0        second-snap  
etu@initiator:~$ sudo mount /dev/lab-vg/lvol0 /mnt/lvol0/
etu@initiator:~$ ls -la /mnt/lvol0/
etu-files/  lost+found/ 
etu@initiator:~$ ls -la /mnt/lvol0/etu-files/
total 8
drwxr-xr-x 2 etu  etu  4096  4 sept. 14:12 .
drwxr-xr-x 4 root root 4096  4 sept. 13:15 ..
-rw-r--r-- 1 etu  etu     0  4 sept. 14:10 first-00-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-01-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-02-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-03-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-04-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-05-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-06-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-07-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-08-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-09-file
-rw-r--r-- 1 etu  etu     0  4 sept. 14:12 first-10-file
-rw-r--r-- 1 etu  etu     0  4 sept. 13:15 my-first-file

```

   iscsiadm -m discovery --type sendtargets --portal=[2001:678:3fc:e6:baad:caff:fefe:1ad]

Pas besoin de copier l'image qcow2 à la main, il faut le faire avec le script fourni
    Ce script sera complété au fil des séances
    Remplir avec ses propres parametres 
Le script copie le fichier .qcow2 et le .fd pour l'amorceur uefi

telnet sur les deux machines crées
    utiliser resize pour ajuste la taille du terminal 


sudo apt install open-iscsi