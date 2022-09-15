# TP NFS - G2 Planète Teth
## NFS Server

- Installation de rpcbind
```
etu@nfsserver:~$ apt search rpcbind
En train de trier... Fait
Recherche en texte intégral... Fait
rpcbind/testing,now 1.2.6-6 amd64  [installé]
  conversion de numéros de programmes RPC en adresses universelles
...
etu@nfsserver:~$ ps aux | grep rpc[b]ind
_rpc         316  0.0  0.3   8028  3920 ?        Ss   14:11   0:00 /sbin/rpcbind -f -w
etu@nfsserver:~$ sudo lsof -i | grep rpcbind 
rpcbind 316 _rpc    4u  IPv4     42      0t0  TCP *:sunrpc (LISTEN)
rpcbind 316 _rpc    5u  IPv4  14361      0t0  UDP *:sunrpc 
rpcbind 316 _rpc    6u  IPv6   1115      0t0  TCP *:sunrpc (LISTEN)
rpcbind 316 _rpc    7u  IPv6  17323      0t0  UDP *:sunrpc 

etu@nfsserver:~$ sudo lsof -i | grep rpc[b]ind 
rpcbind 316 _rpc    4u  IPv4     42      0t0  TCP *:sunrpc (LISTEN)
rpcbind 316 _rpc    5u  IPv4  14361      0t0  UDP *:sunrpc 
rpcbind 316 _rpc    6u  IPv6   1115      0t0  TCP *:sunrpc (LISTEN)
rpcbind 316 _rpc    7u  IPv6  17323      0t0  UDP *:sunrpc 

etu@nfsserver:~$ grep sunrpc /etc/services
sunrpc          111/tcp         portmapper      # RPC 4.0 portmapper
sunrpc          111/udp         portmapper

etu@nfsserver:~$ dpkg -S $(which rpcinfo)
rpcbind: /usr/bin/rpcinfo
!rpcinfo -s ?
etu@nfsserver:~$ rpcinfo -s
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser
etu@nfsserver:~$ rpcinfo -s 172.23.121.211
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser
etu@nfsserver:~$ rpcinfo -s 172.23.121.166
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser
```

- Installation de tshark
- Ajout de etu au groupe wireshark

- Capture des paquet hors port 22
```
etu@nfsserver:~$ tshark -i enp0s1 -f "! port 22"
Capturing on 'enp0s1'
 ** (tshark:1370) 14:49:17.091685 [Main MESSAGE] -- Capture started.
 ** (tshark:1370) 14:49:17.091816 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s1QPKER1.pcapng"
    1 0.000000000 8e:19:d5:29:01:6d → LLDP_Multicast LLDP 232 MA/b4:96:91:9a:50:b8 MA/8e:19:d5:29:01:6d 120 SysN=oscar SysD=Debian GNU/Linux bookworm/sid Linux 5.18.0-4-amd64 #1 SMP PREEMPT_DYNAMIC Debian 5.18.16-1 (2022-08-10) x86_64 
    2 9.119876644 172.23.121.211 → 172.23.121.166 TCP 74 35440 → 111 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=788483933 TSecr=0 WS=128
    3 9.120957883 172.23.121.166 → 172.23.121.211 TCP 74 111 → 35440 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM=1 TSval=2077020029 TSecr=788483933 WS=128
    4 9.121023086 172.23.121.211 → 172.23.121.166 TCP 66 35440 → 111 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=788483934 TSecr=2077020029
    5 9.121372386 172.23.121.211 → 172.23.121.166 Portmap 110 V3 DUMP Call
    6 9.121525942 172.23.121.166 → 172.23.121.211 TCP 66 111 → 35440 [ACK] Seq=1 Ack=45 Win=65152 Len=0 TSval=2077020030 TSecr=788483934
    7 9.121729487 172.23.121.166 → 172.23.121.211 Portmap 754 V3 DUMP Reply (Call In 5)
    8 9.121737168 172.23.121.211 → 172.23.121.166 TCP 66 35440 → 111 [ACK] Seq=45 Ack=689 Win=64128 Len=0 TSval=788483935 TSecr=2077020031
    9 9.121967596 172.23.121.211 → 172.23.121.166 TCP 66 35440 → 111 [FIN, ACK] Seq=45 Ack=689 Win=64128 Len=0 TSval=788483935 TSecr=2077020031
   10 9.122091697 172.23.121.166 → 172.23.121.211 TCP 66 111 → 35440 [FIN, ACK] Seq=689 Ack=46 Win=65152 Len=0 TSval=2077020031 TSecr=788483935
   11 9.122105501 172.23.121.211 → 172.23.121.166 TCP 66 35440 → 111 [ACK] Seq=46 Ack=690 Win=64128 Len=0 TSval=788483935 TSecr=2077020031
   
etu@nfsserver:~$ tshark -i enp0s1 -f "! port 22" -w /var/tmp/rpcbind.pcap
Capturing on 'enp0s1'
 ** (tshark:1334) 14:46:43.507502 [Main MESSAGE] -- Capture started.
 ** (tshark:1334) 14:46:43.507624 [Main MESSAGE] -- File: "/var/tmp/rpcbind.pcap"
11 ^C
```

- Installation des paquets nfs-common et nfs-kernel-server
```
etu@nfsserver:~$ aptitude search ?name"(^nfs)" | grep -v ganesha
v  nfs-client - 
p  nfs-common - fichiers de prise en charge NFS communs au client et au serveur
p  nfs-kernel-server - gestion du serveur NFS du noyau
v  nfs-server - 
p  nfs4-acl-tools - utilitaires pour ACL graphiques et en ligne de commande pour un client NFSv4
p  nfstrace - NFS tracing/monitoring/capturing/analyzing tool
p  nfstrace-doc - NFS tracing/monitoring/capturing/analyzing tool (documentation)
p  nfswatch - programme de surveillance de trafic NFS pour la console

etu@nfsserver:~$ sudo apt install nfs-common

etu@nfsserver:~$ aptitude search '?and(nfs, server)'
p   nfs-kernel-server - gestion du serveur NFS du noyau                                                               
v   nfs-server                                                                                                

etu@nfsserver:~$  sudo apt install nfs-kernel-server
```

- Ajout des instruction d'exportations
```
etu@nfsserver:~$ cat << EOF | sudo tee -a /etc/exports
EOF
/home/exports     172.23.121.166/25(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home     172.23.121.166/25 (rw,sync,no_subtree_check)
EOF
/home/exports     172.23.121.166/25(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home     172.23.121.166/25 (rw,sync,no_subtree_check)

etu@nfsserver:~$ cat << EOF | sudo tee -a /etc/exports
/home/exports     2001:678:3fc:79:baad:caff:fefe:3f/64(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home     2001:678:3fc:79::/64(rw,sync,no_subtree_check)
EOF
/home/exports     2001:678:3fc:79:baad:caff:fefe:3f/64(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home     2001:678:3fc:79::/64(rw,sync,no_subtree_check)

etu@nfsserver:~$ sudo systemctl restart nfs-kernel-server

etu@nfsserver:~$ sudo exportfs
/home/exports   172.23.121.129/25
/home/exports   2001:678:3fc:79::/64
/home/exports/home
                172.23.121.129/25
/home/exports/home
                2001:678:3fc:79::/64
/home/exports/home
```

- Montage local
```
etu@nfsserver:~$ sudo mkdir /ahome
etu@nfsserver:~$ sudo mount --bind /home/exports/home /ahome
etu@nfsserver:~$ echo "/home/exports/home   /ahome  none   defaults,bind   0   0" | \
> sudo tee -a /etc/fstab
/home/exports/home   /ahome  none   defaults,bind   0   0

etu@nfsserver:~$ grep -v ^# /etc/fstab
UUID=a438137b-4270-4cb3-bace-f14ae619b8d1 /               ext4    errors=remount-ro 0       1
UUID=6E00-0763  /boot/efi       vfat    umask=0077      0       1
UUID=ee8bd24d-5fc3-4664-bfb2-eda3aedc0058 none            swap    sw              0       0
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
/home/exports/home   /ahome  none   defaults,bind   0   0
```

## NFS Server - Droits 
- Création du  compte etu-nfs
```
etu@nfsserver:~$ sudo adduser --home /ahome/etu-nfs etu-nfs
Ajout de l'utilisateur « etu-nfs » ...
Ajout du nouveau groupe « etu-nfs » (1001) ...
Ajout du nouvel utilisateur « etu-nfs » (1001) avec le groupe « etu-nfs » ...
Création du répertoire personnel « /ahome/etu-nfs »...
Copie des fichiers depuis « /etc/skel »...
Nouveau mot de passe : 
Retapez le nouveau mot de passe : 
passwd: password updated successfully
Changing the user information for etu-nfs
Enter the new value, or press ENTER for the default
        Full Name []: etu-nfs
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Cette information est-elle correcte ? [O/n]
etu@nfsserver:~$ su - etu-nfs
Mot de passe : 
etu-nfs@nfsserver:~$ echo "This file is mine" > textfile
etu-nfs@nfsserver:~$ exit
déconnexion
etu@nfsserver:~$ id etu-nfs
uid=1001(etu-nfs) gid=1001(etu-nfs) groupes=1001(etu-nfs)

etu@nfsserver:~$ sudo touch /ahome/etu-nfs/ThisOneIsMine
[sudo] Mot de passe de etu : 

etu@nfsserver:~$ sudo chown etu-nfs.etu-nfs /ahome/etu-nfs/ThisOneIsMine

etu@nfsserver:~$ sudo touch /ahome/etu-nfs/ThisOneIs-NOT-Mine

etu@nfsserver:~$ sudo chown 2000.2000 /ahome/etu-nfs/ThisOneIs-NOT-Mine

etu@nfsserver:~$ sudo ls -lh /ahome/etu-nfs/
total 4,0K
-rw-r--r-- 1 etu-nfs etu-nfs  0 25 août  16:00 CestAMoi
-rw-r--r-- 1 etu-nfs etu-nfs 18 25 août  15:48 textfile
-rw-r--r-- 1 etu-nfs etu-nfs  0 25 août  16:00 ThisOneIsMine
-rw-r--r-- 1    2000    2000  0 25 août  16:01 ThisOneIs-NOT-Mine

etu@client:~$ ls -l /ahome/etu-nfs/
total 4
-rw-r--r-- 1 etu-nfs etu-nfs 18 25 août  15:48 textfile
-rw-r--r-- 1 etu-nfs etu-nfs  0 25 août  16:00 ThisOneIsMine
-rw-r--r-- 1    2000    2000  0 25 août  16:01 ThisOneIs-NOT-Mine

```
