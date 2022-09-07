# TP1 iSCSI (G2 Teth)
## Initiator - Accès Réseau
- Verification de la configuration de l'interface tap62 du commutateur
```
lekienemery@oscar:~/vm/1-iscsi$ sudo ovs-vsctl list port tap62
_uuid               : 139d6d9d-a09e-4dd6-abf3-429662221ad5
bond_active_slave   : []
bond_downdelay      : 0
bond_fake_iface     : false
bond_mode           : []
bond_updelay        : 0
cvlans              : []
external_ids        : {}
fake_bridge         : false
interfaces          : [7f82a1a5-afdf-47a9-a815-b69cfe891ef2]
lacp                : []
mac                 : []
name                : tap62
other_config        : {}
protected           : false
qos                 : []
rstp_statistics     : {}
rstp_status         : {}
statistics          : {}
status              : {}
tag                 : 147
trunks              : []
vlan_mode           : access
```
- Personnalisation du script de lancement de la VM Initiator
```
lekienemery@oscar:~/vm/1-iscsi$ cat ./startup.sh 
#!/bin/bash

VLAN=147 # À changer !!!
CORDON=62 # À changer !!!

VM_FILENAME=initiator # À changer !!!
SECOND_DISK=initiatordisk # À changer !!!

SECOND_DISK_SIZE=32G
RAM=1024

# Attribution du VLAN au port du commutateur
sudo ovs-vsctl set port tap$CORDON tag=$VLAN

# Création de l'unité de disque supplémentaire
if [[ ! -e $SECOND_DISK.qcow2 ]]
  then qemu-img create -f qcow2 $SECOND_DISK.qcow2 $SECOND_DISK_SIZE
fi

# Lancement d'une VM avec le rôle iSCSI # À changer !!!
$HOME/vm/scripts/ovs-startup.sh $VM_FILENAME.qcow2 $RAM $CORDON \
  -drive if=none,id=$SECOND_DISK,format=qcow2,media=disk,file=$SECOND_DISK.qcow2 \
  -device virtio-blk,drive=$SECOND_DIS
```
- Lancement de la VM
```
lekienemery@oscar:~/vm/1-iscsi$ ./startup.sh 
~> Virtual machine filename   : initiator.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 5962
~> telnet console port number : 2362
~> MAC address                : b8:ad:ca:fe:00:3e
~> Switch port interface      : tap62, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:3e%vlan147
```
- Renommage de la VM
- Mise à jour des paquets
- Configuration de l'interface réseau
```
etu@initiator:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:00:3e brd ff:ff:ff:ff:ff:ff
    inet 192.168.147.65/25 brd 192.168.147.127 scope global enp0s1
       valid_lft forever preferred_lft forever
    inet6 2001:678:3fc:93:baad:caff:fefe:3e/64 scope global dynamic mngtmpaddr 
       valid_lft 2591892sec preferred_lft 604692sec
    inet6 fe80::baad:caff:fefe:3e/64 scope link 
       valid_lft forever preferred_lft forever
```
libération de l'interface
```
etu@initiator:~$ sudo ifdown --force enp0s1
```
- Configuration statique de l'interface
```
etu@initiator:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s1

# This is an autocst
iface enp0s1 inet static
        address 192.168.147.65/25
        gateway 192.168.147.1
#nfigured IPv6 interface
#iface enp0s1 inet
```
```
etu@initiator:~$ sudo ifup enp0s1 
etu@initiator:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:00:3e brd ff:ff:ff:ff:ff:ff
    inet 192.168.147.65/25 brd 192.168.147.127 scope global enp0s1
       valid_lft forever preferred_lft forever
    inet6 2001:678:3fc:93:baad:caff:fefe:3e/64 scope global dynamic mngtmpaddr 
       valid_lft 2591999sec preferred_lft 604799sec
    inet6 fe80::baad:caff:fefe:3e/64 scope link 
       valid_lft forever preferred_lft forever
etu@initiator:~$ ip route ls
default via 192.168.147.1 dev enp0s1 onlink 
192.168.147.0/25 dev enp0s1 proto kernel scope link src 192.168.147.65 
etu@initiator:~$ ip -6 route ls
::1 dev lo proto kernel metric 256 pref medium
2001:678:3fc:93::/64 dev enp0s1 proto kernel metric 256 expires 2591957sec pref medium
fe80::/64 dev enp0s1 proto kernel metric 256 pref medium
default via fe80:93::1 dev enp0s1 proto ra metric 1024 expires 1757sec hoplimit 64 pref high
etu@initiator:~$ ping 192.168.147.1
PING 192.168.147.1 (192.168.147.1) 56(84) bytes of data.
64 bytes from 192.168.147.1: icmp_seq=1 ttl=255 time=3.11 ms
64 bytes from 192.168.147.1: icmp_seq=2 ttl=255 time=4.39 ms
^C
--- 192.168.147.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 3.110/3.751/4.393/0.641 ms
etu@initiator:~$ ping 9.9.9.9
PING 9.9.9.9 (9.9.9.9) 56(84) bytes of data.
64 bytes from 9.9.9.9: icmp_seq=1 ttl=53 time=14.0 ms
64 bytes from 9.9.9.9: icmp_seq=2 ttl=53 time=13.5 ms
^C
--- 9.9.9.9 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 13.493/13.748/14.004/0.255 ms
etu@initiator:~$ ping 2620:fe::fe
PING 2620:fe::fe(2620:fe::fe) 56 data bytes
64 bytes from 2620:fe::fe: icmp_seq=1 ttl=59 time=47.5 ms
64 bytes from 2620:fe::fe: icmp_seq=2 ttl=59 time=47.8 ms
^C
--- 2620:fe::fe ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 47.544/47.684/47.825/0.140 ms
lekienemery@oscar:~/vm/1-iscsi$ ssh -p 2222 etu@192.168.147.65
etu@192.168.147.65's password: 
Linux initiator 5.18.0-4-amd64 #1 SMP PREEMPT_DYNAMIC Debian 5.18.16-1 (2022-08-10) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have no mail.
Last login: Wed Aug 24 18:43:37 2022
etu@initiator:~$
```


## Initiator - Preparation de l'unité de stockage
- Partionnement du disque virtuel
```
etu@initiator:~$ dpkg -L util-linux | grep bin/ls
/bin/lsblk
/usr/bin/lscpu
/usr/bin/lsipc
/usr/bin/lslocks
/usr/bin/lslogins
/usr/bin/lsmem
/usr/bin/lsns
etu@initiator:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sr0     11:0    1  1024M  0 rom  
vda    254:0    0   120G  0 disk 
├─vda1 254:1    0   512M  0 part /boot/efi
├─vda2 254:2    0 118,5G  0 part /
└─vda3 254:3    0   977M  0 part [SWAP]
vdb    254:16   0    32G  0 disk
etu@initiator:~$ sudo apt install parted -y
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait      
Les paquets supplémentaires suivants seront installés : 
  libparted2
Paquets suggérés :
  libparted-dev libpa
etu@initiator:~$ sudo parted /dev/vdb print
Error: /dev/vdb: unrecognised disk label
                                                                Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 34,4GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
etu@initiator:~$ sudo dd if=/dev/zero of=/dev/vdb bs=512 count=44+0 enregistrements lus
4+0 enregistrements écrits
2048 octets (2,0 kB, 2,0 KiB) copiés, 0,00877679 s, 233 kB/s
etu@initiator:~$ sudo parted /dev/vdb
GNU Parted 3.5
Using /dev/vdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
                                                                (parted) print
Error: /dev/vdb: unrecognised disk label
                                                                Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 34,4GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags: 
                                                                (parted) mklabel gpt
                                                                (parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 34,4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  Flags

                                                                (parted) mkpart ext4 0% 100%
                                                                (parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 34,4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  34,4GB  34,4GB               ext4

                                                                (parted) quit
Information: You may need to update /etc/fstab.
```
- Formatage de la nouvelle partition en ext4
```
etu@initiator:~$ sudo mkfs.ext4 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done                            
Creating filesystem with 8388096 4k blocks and 2097152 inodes
Filesystem UUID: 1cfcacf5-681e-4b60-bd26-2f51770b1665
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000, 7962624

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information:   0/2done 
etu@initiator:~$ sudo tune2fs -l /dev/vdb1
tune2fs 1.46.5 (30-Dec-2021)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          1cfcacf5-681e-4b60-bd26-2f51770b1665
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              2097152
Block count:              8388096
Reserved block count:     419404
Overhead clusters:        176700
Free blocks:              8211390
Free inodes:              2097141
First block:              0
Block size:               4096
Fragment size:            4096
Group descriptor size:    64
Reserved GDT blocks:      1024
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
Filesystem created:       Tue Aug 23 19:15:15 2022
Last mount time:          n/a
Last write time:          Tue Aug 23 19:15:15 2022
Mount count:              0
Maximum mount count:      -1
Last checked:             Tue Aug 23 19:15:15 2022
Check interval:           0 (<none>)
Lifetime writes:          4182 kB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:	          256
Required extra isize:     32
Desired extra isize:      32
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      84237e7f-9f1d-448b-90ed-0097b1d47856
Journal backup:           inode blocks
Checksum type:            crc32c
Checksum:                 0x0ba314ab
```
- Montage de la partition nouvellement créée
```
etu@initiator:~$ sudo blkid /dev/vdb1
/dev/vdb1: UUID="1cfcacf5-681e-4b60-bd26-2f51770b1665" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="ext4" PARTUUID="40654eff-3c63-465f-858d-8e401314ecb7"
etu@initiator:~$ grep -v '^#' /etc/fstab
UUID=a438137b-4270-4cb3-bace-f14ae619b8d1 /               ext4    errors=remount-ro 0       1
UUID=6E00-0763  /boot/efi       vfat    umask=0077      0       1
UUID=ee8bd24d-5fc3-4664-bfb2-eda3aedc0058 none            swap    sw              0       0
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
```

```
etu@initiator:~$ sudo mount /dev/vdb1 /mnt
[sudo] Mot de passe de etu : 
etu@initiator:~$ mount | grep '/dev/vd'
/dev/vda2 on / type ext4 (rw,relatime,errors=remount-ro)
/dev/vda1 on /boot/efi type vfat (rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro)
/dev/vdb1 on /mnt type ext4 (rw,relatime)
## Initiator - Service open-iscsi
- Installation et démarrage d'open-iscsi
```
etu@initiator:~$ aptitude search "?description(scsi)?description(initiator)"
p   open-iscsi               - iSCSI initiator tools            
p   open-isns-discoveryd     - Internet Storage Name Service - i
p   resource-agents          - Cluster Resource Agents          
etu@initiator:~$ sudo apt install open-iscsi
[sudo] Mot de passe de etu : 
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait  
```
```
etu@initiator:~$ sudo systemctl restart open-iscsi.service 
etu@initiator:~$ systemctl status open-iscsi.service
○ open-iscsi.service - Login to default iSCSI targets
     Loaded: loaded (/lib/systemd/system/open-iscsi.service; enabled; preset: enabled)
     Active: inactive (dead)
  Condition: start condition failed at Wed 2022-08-24 16:34:03 CEST; 3s ago
             ├─ ConditionDirectoryNotEmpty=|/etc/iscsi/nodes was not met
             └─ ConditionDirectoryNotEmpty=|/sys/class/iscsi_session was not met
       Docs: man:iscsiadm(8)
             man:iscsid(8)

août 24 18:04:13 initiator systemd[1]: Login to default iSCSI targets was skipped because all trigger condition checks failed.
août 24 16:34:03 initiator systemd[1]: Login to default iSCSI targets was skipped because all trigger condition checks failed.
etu@initiator:~$ grep -i iscsi /var/log/syslog
Aug 23 19:01:57 initiator systemd[1]: Listening on Open-iSCSI iscsid Socket.
Aug 23 19:01:57 initiator systemd[1]: Login to default iSCSI targets was skipped because all trigger condition checks failed.
Aug 24 18:04:13 initiator systemd[1]: Listening on Open-iSCSI iscsid Socket.
Aug 24 18:04:13 initiator systemd[1]: Login to default iSCSI targets was skipped because all trigger condition checks failed.
Aug 24 16:34:03 initiator systemd[1]: Login to default iSCSI targets was skipped because all trigger condition checks failed.
```
- Configuration du service
```
etu@initiator:~$ dpkg -L open-iscsi | grep 'bin'
/sbin
/sbin/iscsi-iname
/sbin/iscsi_discovery
/sbin/iscsiadm
/sbin/iscsid
/sbin/iscsistart
/usr/bin
/usr/bin/iscsiadm
```
```
## Authentification CHAP
- Ajout des identifiants dans le fichier /etc/iscsi/iscsid.conf 
- Decouverte du nouveau volume
etu@initiator:~$ sudo iscsiadm -m discovery --type sendtargets --portal=[2001:678:3fc:93:baad:caff:fefe:3f]:3260
[2001:678:3fc:93:baad:caff:fefe:3f]:3260,1 iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321
etu@initiator:~$ sudo iscsiadm -m node
192.168.147.66:3260,1 iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321
[2001:678:3fc:93:baad:caff:fefe:3f]:3260,1 iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321
- Ouverture de session avec authentification CHAP
etu@initiator:~$ sudo iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321 -p 2001:678:3fc:93:baad:caff:fefe:3f -l
Logging in to [iface: default, target: iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321, portal: 2001:678:3fc:93:baad:caff:fefe:3f,3260]
Login to [iface: default, target: iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321, portal: 2001:678:3fc:93:baad:caff:fefe:3f,3260] successful.
etu@initiator:~$ sudo iscsiadm -m session
tcp: [1] [2001:678:3fc:93:baad:caff:fefe:3f]:3260,1 iqn.2003-01.org.linux-iscsi.target.x8664:sn.887488016321 (non-flash)


Config permanente
-modification du fichier /etc/iscsi/iscsid.conf 
set node.startup = automatic
-ajout des identifiants dans la partie discovery 
- Montage automatique de sda1
etu@initiator:~$ sudo lsblk /dev/sda1
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sda1   8:1    0  32G  0 part 
etu@initiator:~$ sudo blkid /dev/sda1
/dev/sda1: UUID="df1497d5-efd6-4fba-a6eb-1439958b0377" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="ext4" PARTUUID="5710e971-7431-44f3-bc16-7e98da6e066c"
etu@initiator:~$ echo "df1497d5-efd6-4fba-a6eb-1439958b0377 \
> /var/cache/iscsi-vol0 \
> btrfs \
> _netdev \
> 0   2" | sudo tee -a /etc/fstab
df1497d5-efd6-4fba-a6eb-1439958b0377 /var/cache/iscsi-vol0 btrfs _netdev 0   2

Après un reboot, la session iscsi est ouverte 
et le volume est monté 


