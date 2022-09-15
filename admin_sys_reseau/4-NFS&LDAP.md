# TP4 NFS & LDAP - G2 Planète Teth

## Serveur

- Lancement de la VM server (ubuntu server)

```
castresamuel@oscar:~/vm/4-asso_ldap_nfsv4$ ./startup-server.sh 
~> Virtual machine filename   : ldapnfsv4-server-ubuntu-server.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 5962
~> telnet console port number : 2362
~> MAC address                : b8:ad:ca:fe:00:3e
~> Switch port interface      : tap62, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:3e%vlan305
qxl_send_events: spice-server bug: guest stopped, ignoring
```

- Renommage de la VM
```
sudo sed -i 's/vm0/server/g' /etc/hosts
sudo sed -i 's/vm0/server/g' /etc/hostname
```

- Configurationde l'interface réseau 
```
root@server:/home/etu# ip link set dev enp0s1 down
root@server:/home/etu# cat /etc/netplan/00-installer-config.yaml 
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s1:
      dhcp4: false
  version: 2
  renderer: networkd
  ethernets:
    enp0s1:
      addresses:
        - 10.0.49.194/27
      nameservers:
        addresses: [172.16.0.2]
      routes:
        - to: default
          via:  10.0.49.193

root@server:/home/etu# netplan apply
root@server:/home/etu# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:00:3e brd ff:ff:ff:ff:ff:ff
    inet 10.0.49.194/27 brd 10.0.49.223 scope global enp0s1
       valid_lft forever preferred_lft forever
    inet6 fe80::baad:caff:fefe:3e/64 scope link 
       valid_lft forever preferred_lft forever
```

### Serveur LDAP 

- Installation su service d'annuaire LDAP

```
┌─[etu][server][~]
└─▪ sudo aptitude install slapd ldap-utils
┌─[etu][server][~]
└─▪ sudo ps aux | grep l[d]ap
openldap    1200  0.0  0.5 1157924 5500 ?        Ssl  17:21   0:00 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d
┌─[etu][server][~]
└─▪ sudo lsof -i | grep l[d]ap
slapd     1200        openldap    8u  IPv4  22122      0t0  TCP *:ldap (LISTEN)
slapd     1200        openldap    9u  IPv6  22123      0t0  TCP *:ldap (LISTEN)
┌─[etu][server][~]
└─▪ sudo ss -tau | grep l[d]ap
tcp   LISTEN 0      2048         0.0.0.0:ldap        0.0.0.0:*          
tcp   LISTEN 0      2048            [::]:ldap           [::]:*          
┌─[etu][server][~]
└─▪ grep ldap /etc/services
ldap		389/tcp			# Lightweight Directory Access Protocol
ldap		389/udp
ldaps		636/tcp				# LDAP over SSL
ldaps		636/udp
```
- Reinitialisation de la base de l'annuaire 
```

┌─[etu][server][~]
└─▪ systemctl list-units | grep slapd
  slapd.service                                                                       loaded active running   LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)
┌─[etu][server][~]
└─▪ systemctl stop slapd
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'slapd.service'.
Authenticating as: Etudiant (etu)
Password: 
==== AUTHENTICATION COMPLETE ===
┌─[etu][server][~]
└─▪ sudo rm /var/lib/ldap/*
rm: cannot remove '/var/lib/ldap/*': No such file or directory
┌─[etu][server][~]
└─▪ sudo rm -rf /etc/ldap/slapd.d
┌─[etu][server][~]
└─▪ sudo dpkg-reconfigure slapd
```

- Analyse de la configuration du service LDAP
```
┌─[etu][server][~]
└─▪ sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config"
...
# {1}mdb, config
dn: olcDatabase={1}mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: {1}mdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=lab,dc=stri
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by * read
olcLastMod: TRUE
olcRootDN: cn=admin,dc=lab,dc=stri
olcRootPW: {SSHA}c+70qgez9/g/KTORnSGL6rmrKKlMwzFA
olcDbCheckpoint: 512 30
olcDbIndex: objectClass eq
olcDbIndex: cn,uid eq
olcDbIndex: uidNumber,gidNumber eq
olcDbIndex: member,memberUid eq
olcDbMaxSize: 1073741824

# search result
search: 2
result: 0 Success

# numResponses: 11
# numEntries: 10
```

- Activation de la journalisation
```
root@server:/home/etu# cat setolcLogLevel2stats.ldif
# Set olcLogLevel 2 stats
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats

root@server:/home/etu#  ldapmodify -Y EXTERNAL -H ldapi:/// -f setolcLogLevel2stats.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

root@server:/home/etu#  ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" olcLogLevel |\
 grep ^olcLogLevel
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcLogLevel: stats

root@server:/home/etu# grep -5 olcLogLevel /var/log/syslog
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=0 BIND dn="" method=163
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=0 BIND authcid="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" authzid="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=0 BIND dn="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" mech=EXTERNAL bind_ssf=0 ssf=71
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=0 RESULT tag=97 err=0 qtime=0.000035 etime=0.000465 text=
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=1 MOD dn="cn=config"
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=1 MOD attr=olcLogLevel
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=1 RESULT tag=103 err=0 qtime=0.000015 etime=0.000572 text=
Sep 14 17:52:17 server slapd[1744]: conn=1005 op=2 UNBIND
Sep 14 17:52:17 server slapd[1744]: conn=1005 fd=12 closed
Sep 14 17:52:51 server slapd[1744]: conn=1006 fd=12 ACCEPT from PATH=/var/run/slapd/ldapi (PATH=/var/run/slapd/ldapi)
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=0 BIND dn="" method=163
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=0 BIND authcid="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" authzid="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=0 BIND dn="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" mech=EXTERNAL bind_ssf=0 ssf=71
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=0 RESULT tag=97 err=0 qtime=0.000020 etime=0.000153 text=
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=1 MOD dn="cn=config"
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=1 MOD attr=olcLogLevel
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=1 RESULT tag=103 err=0 qtime=0.000018 etime=0.000578 text=
Sep 14 17:52:51 server slapd[1744]: conn=1006 op=2 UNBIND
Sep 14 17:52:51 server slapd[1744]: conn=1006 fd=12 closed
Sep 14 17:53:41 server slapd[1744]: conn=1007 fd=12 ACCEPT from PATH=/var/run/slapd/ldapi (PATH=/var/run/slapd/ldapi)
Sep 14 17:53:41 server slapd[1744]: conn=1007 op=0 BIND dn="" method=163

```
-- Ajout de deux unités organisationnelles
```
root@server:/home/etu# vi ou.ldif
root@server:/home/etu# cat ou.ldif
dn: ou=people,dc=lab,dc=stri
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=lab,dc=stri
objectClass: organizationalUnit
ou: groups
root@server:/home/etu# ldapadd -cxWD cn=admin,dc=lab,dc=stri -f ou.ldif
Enter LDAP Password: 
adding new entry "ou=people,dc=lab,dc=stri"

adding new entry "ou=groups,dc=lab,dc=stri"
root@server:/home/etu#  ldapsearch -LLL -x -H ldap:/// -b "dc=lab,dc=stri" -D cn=admin,dc=lab,dc=stri -W
Enter LDAP Password: 
dn: dc=lab,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab.stri
dc: lab

dn: ou=groups,dc=lab,dc=stri
objectClass: organizationalUnit
ou: groups

dn: ou=people,dc=lab,dc=stri
objectClass: organizationalUnit
ou: people
```

- Ajout de 4 comptes utilisateurs dans l'annuaire

```
root@server:/home/etu# slappasswd -v -h "{SSHA}" -s etu
{SSHA}PMQU0LzG3WgMsSu9tr1YQeMb7I7CepP3
root@server:/home/etu# vi users.ldif
root@server:/home/etu# cat users.ldif 
# Padmé Amidala
dn: uid=padme,ou=people,dc=lab,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Padme
sn: Padmé Amidala Skywalker
uid: padme
uidNumber: 10000
gidNumber: 10000
loginShell: /bin/bash
homeDirectory: /ahome/padme
userPassword: {SSHA}v+FjDFxekTrOP4e/ghW84bA9ll6+SDBA
gecos: Padme Amidala Skywalker

# Anakin Skywalker
dn: uid=anakin,ou=people,dc=lab,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Anakin
sn: Anakin Skywalker
uid: anakin
uidNumber: 10001
gidNumber: 10001
loginShell: /bin/bash
homeDirectory: /ahome/anakin
userPassword: {SSHA}v+FjDFxekTrOP4e/ghW84bA9ll6+SDBA
gecos: Anakin Skywalker

# Leia Organa Skywalker
dn: uid=leia,ou=people,dc=lab,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Leia
sn: Leia Organa
uid: leia
uidNumber: 10002
gidNumber: 10002
loginShell: /bin/bash
homeDirectory: /ahome/leia
userPassword: {SSHA}v+FjDFxekTrOP4e/ghW84bA9ll6+SDBA
gecos: Leia Organa Skywalker

# Luke Skywalker
dn: uid=luke,ou=people,dc=lab,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Luke
sn: Luke Skywalker
uid: luke
uidNumber: 10003
gidNumber: 10003
loginShell: /bin/bash
homeDirectory: /ahome/luke
userPassword: {SSHA}v+FjDFxekTrOP4e/ghW84bA9ll6+SDBA
gecos: Luke Skywalker

root@server:/home/etu# ldapadd -cxWD cn=admin,dc=lab,dc=stri -f users.ldif
Enter LDAP Password: 
adding new entry "uid=padme,ou=people,dc=lab,dc=stri"

adding new entry "uid=anakin,ou=people,dc=lab,dc=stri"

adding new entry "uid=leia,ou=people,dc=lab,dc=stri"

adding new entry "uid=luke,ou=people,dc=lab,dc=stri"

root@server:/home/etu# ldapsearch -LLL -x -H ldap:/// -b "dc=lab,dc=stri" -D cn=admin,dc=lab,dc=stri -W
Enter LDAP Password: 
dn: dc=lab,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab.stri
dc: lab

dn: ou=groups,dc=lab,dc=stri
objectClass: organizationalUnit
ou: groups

dn: ou=people,dc=lab,dc=stri
objectClass: organizationalUnit
ou: people

dn: uid=leia,ou=people,dc=lab,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Leia
sn: Leia Organa
uid: leia
uidNumber: 10002
gidNumber: 10002
loginShell: /bin/bash
homeDirectory: /ahome/leia
userPassword:: e1NTSEF9d

```

### Serveur NFS 

- Configuration du serveur 
```
root@server:/home/etu#  sudo apt install rpcbind
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait      
Les NOUVEAUX paquets suivants seront installés :
  rpcbind
0 mis à jour, 1 nouvellement installés, 0 à enlever et 7 non mis à jour.
Il est nécessaire de prendre 46,6 ko dans les archives.
Après cette opération, 160 ko d'espace disque supplémentaires seront utilisés.
Réception de :1 http://fr.archive.ubuntu.com/ubuntu jammy/main amd64 rpcbind amd64 1.2.6-2build1 [46,6 kB]
46,6 ko réceptionnés en 1s (91,4 ko/s)
Sélection du paquet rpcbind précédemment désélectionné.
(Lecture de la base de données... 111086 fichiers et répertoires déjà installés.)
Préparation du dépaquetage de .../rpcbind_1.2.6-2build1_amd64.deb ...
Dépaquetage de rpcbind (1.2.6-2build1) ...
Paramétrage de rpcbind (1.2.6-2build1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/rpcbind.service → /lib/systemd/system/rpcbind.service.
Created symlink /etc/systemd/system/sockets.target.wants/rpcbind.socket → /lib/systemd/system/rpcbind.socket.
Traitement des actions différées (« triggers ») pour man-db (2.10.2-1) ...
Scanning processes...                                                                                                                                                                                        
Scanning linux images...                                                                                                                                                                                     

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
root@server:/home/etu# ps aux | grep rpc[b]ind
_rpc        3666  0.0  0.4   8104  4312 ?        Ss   18:33   0:00 /sbin/rpcbind -f -w
root@server:/home/etu#  lsof -i | grep rpc[b]ind
rpcbind   3666            _rpc    4u  IPv4  35223      0t0  TCP *:sunrpc (LISTEN)
rpcbind   3666            _rpc    5u  IPv4  38090      0t0  UDP *:sunrpc 
rpcbind   3666            _rpc    6u  IPv6  38990      0t0  TCP *:sunrpc (LISTEN)
rpcbind   3666            _rpc    7u  IPv6  40001      0t0  UDP *:sunrpc 
root@server:/home/etu# grep sunrpc /etc/services
sunrpc		111/tcp		portmapper	# RPC 4.0 portmapper
sunrpc		111/udp		portmapper

root@server:/home/etu# apt install nfs-common
...
root@server:/home/etu# apt install nfs-kernel-server
...
root@server:/home/etu# mkdir -p /home/exports/home

root@server:/home/etu# cat << EOF | sudo tee -a /etc/exports
/home/exports 10.0.49.195/27(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home 10.0.49.195/27(rw,sync,no_subtree_check)
EOF
/home/exports 10.0.49.195/27(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home 10.0.49.195/27(rw,sync,no_subtree_check)

root@server:/home/etu# cat << EOF | sudo tee -a /etc/exports
/home/exports fe80::baad:caff:fefe:3f/64(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home fe80::baad:caff:fefe:3f/64(rw,sync,no_subtree_check)
EOF
/home/exports fe80::baad:caff:fefe:3f/64(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home fe80::baad:caff:fefe:3f/64(rw,sync,no_subtree_check)

root@server:/home/etu# grep -v ^# /etc/exports
/home/exports 10.0.49.195/27(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home 10.0.49.195/27(rw,sync,no_subtree_check)
/home/exports fe80::baad:caff:fefe:3f/64(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home fe80::baad:caff:fefe:3f/64(rw,sync,no_subtree_check)

root@server:/home/etu# systemctl restart nfs-kernel-server
root@server:/home/etu# systemctl status nfs-kernel-server
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
     Active: active (exited) since Wed 2022-09-14 19:47:45 UTC; 8s ago
    Process: 4124 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 4125 ExecStart=/usr/sbin/rpc.nfsd (code=exited, status=0/SUCCESS)
   Main PID: 4125 (code=exited, status=0/SUCCESS)
        CPU: 13ms

sept. 14 19:47:44 server systemd[1]: Starting NFS server and services...
sept. 14 19:47:45 server systemd[1]: Finished NFS server and services.
root@server:/home/etu# exportfs
/home/exports 	10.0.49.195/27
/home/exports 	fe80::baad:caff:fefe:3f/64
/home/exports/home
		10.0.49.195/27
/home/exports/home
		fe80::baad:caff:fefe:3f/64

root@server:/home/etu# mkdir /ahome
root@server:/home/etu# mount --bind /home/exports/home /ahome

┌─[etu][server][~]
└─▪ echo "/home/exports/home /ahome none defaults,bind 0 0" | \
sudo tee -a /etc/fstab
[sudo] password for etu: 
/home/exports/home /ahome none defaults,bind 0 0
```
-Montage automatique 

```
┌─[etu][server][~]
└─▪ sudo umount /ahome

┌─[etu][server][~]
└─▪ grep -v ^# /etc/fstab
/dev/disk/by-uuid/ae79c94e-beec-4141-a682-7bd8db447f50 / btrfs defaults 0 1
/dev/disk/by-uuid/D723-795D /boot/efi vfat defaults 0 1
/swap.img	none	swap	sw	0	0
/home/exports/home /ahome none defaults,bind 0 0
```
- Création automatique du répertoire utilisateur 
```
┌─[etu][server][~]
└─▪ sudo  aptitude install oddjob-mkhomedir -y
<snip>

┌─[etu][server][~]
└─▪ systemctl status oddjobd
● oddjobd.service - privileged operations for unprivileged applications
     Loaded: loaded (/lib/systemd/system/oddjobd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-09-14 20:15:22 UTC; 55s ago
   Main PID: 4370 (oddjobd)
      Tasks: 1 (limit: 1021)
     Memory: 720.0K
        CPU: 10ms
     CGroup: /system.slice/oddjobd.service
             └─4370 /usr/sbin/oddjobd -n -p /run/oddjobd.pid -t 300

sept. 14 20:15:22 server systemd[1]: Started privileged operations for unprivileged applications

┌─[etu][server][~]
└─▪ cat /etc/pam.d/common-session
#
# /etc/pam.d/common-session - session-related modules common to all services
#
# This file is included from other service-specific PAM config files,
# and should contain a list of modules that define tasks to be performed
# at the start and end of interactive sessions.
#
# As of pam 1.0.1-6, this file is managed by pam-auth-update by default.
# To take advantage of this, it is recommended that you configure any
# local modules either before or after the default block, and use
# pam-auth-update to manage selection of other modules.  See
# pam-auth-update(8) for details.

# here are the per-package modules (the "Primary" block)
session	[default=1]			pam_permit.so
# here's the fallback if no module succeeds
session	requisite			pam_deny.so
# prime the stack with a positive return value if there isn't one already;
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
session	required			pam_permit.so
# The pam_umask module will set the umask according to the system default in
# /etc/login.defs and user settings, solving the problem of different
# umask settings with different shells, display managers, remote sessions etc.
# See "man pam_umask".
session optional			pam_umask.so
# and here are more per-package modules (the "Additional" block)
session	required	pam_unix.so 
session	optional	pam_systemd.so 
# end of pam-auth-update config
session optional pam_oddjob_mkhomedir.so skel=/etc/skel/ umask=0022

┌─[etu][server][~]
└─▪ sudo  aptitude install libnss-ldapd -y
<snip>
                                                        
┌─[etu][server][~]
└─▪ sudo apt install libpam-ldapd -y
<snip>
┌─[etu][server][~]
└─▪ sudo dpkg-reconfigure libpam-ldapd
┌─[etu][server][~]
└─▪ sudo  dpkg-reconfigure nslcd
┌─[etu][server][~]
└─▪ sudo systemctl restart nscd
┌─[etu][server][~]
└─▪ sudo systemctl status nscd
● nscd.service - Name Service Cache Daemon
     Loaded: loaded (/lib/systemd/system/nscd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-09-14 20:38:34 UTC; 5s ago
    Process: 5629 ExecStart=/usr/sbin/nscd (code=exited, status=0/SUCCESS)
   Main PID: 5630 (nscd)
      Tasks: 10 (limit: 1021)
     Memory: 792.0K
        CPU: 8ms
     CGroup: /system.slice/nscd.service
             └─5630 /usr/sbin/nscd

```

### Configuration de l'automontage avec le service LDAP
```
┌─[etu][server][~]
└─▪ sudo mv autofs.schema  /etc/ldap/schema/

─[etu][server][~]
└─▪ mkdir schema-convert
mkdir: created directory 'schema-convert'

┌─[etu][server][~]
└─▪ vi schema-convert.conf

┌─[etu][server][~]
└─▪ cat schema-convert.conf 
include /etc/ldap/schema/core.schema
include /etc/ldap/schema/cosine.schema
include /etc/ldap/schema/inetorgperson.schema
include /etc/ldap/schema/autofs.schema

┌─[etu][server][~]
└─▪ slaptest -f schema-convert.conf -F schema-convert
config file testing succeeded

┌─[etu][server][~]
└─▪ cat schema-convert/cn\=config/cn\=schema/cn\=\{3\}autofs.ldif | \
egrep -v structuralObjectClass\|entryUUID\|creatorsName | \
egrep -v createTimestamp\|entryCSN\|modifiersName\|modifyTimestamp | \
sed 's/dn: cn={.}autofs/dn: cn=autofs,cn=schema,cn=config/g' | \
sed 's/{.}autofs/autofs/' > autofs.ldif

┌─[etu][server][~]
└─▪  sudo ldapadd -Y EXTERNAL -H ldapi:/// -f autofs.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=autofs,cn=schema,cn=config"

┌─[etu][server][~]
└─▪ vi ou-autofs.ldif
       
┌─[etu][server][~]
└─▪ cat ou-autofs.ldif 
dn: ou=automount,dc=lab,dc=stri
ou: automount
objectClass: top
objectClass: organizationalUnit

dn: ou=auto.master,ou=automount,dc=lab,dc=stri
ou: auto.master
objectClass: top
objectClass: automountMap

dn: cn=/ahome,ou=auto.master,ou=automount,dc=lab,dc=stri
cn: /ahome
objectClass: top
objectClass: automount
automountInformation: ldap:ou=auto.home,ou=automount,dc=lab,dc=stri

dn: ou=auto.home,ou=automount,dc=lab,dc=stri
ou: auto.home
objectClass: top
objectClass: automountMap

dn: cn=*,ou=auto.home,ou=automount,dc=lab,dc=stri
cn: *
objectClass: top
objectClass: automount
automountInformation: -fstype=nfs4 192.0.2.12:/home/&
┌─[etu][server][~]
└─▪ sudo ldapadd -cxWD cn=admin,dc=lab,dc=stri -f ou-autofs.ldif
Enter LDAP Password: 
adding new entry "ou=automount,dc=lab,dc=stri"

adding new entry "ou=auto.master,ou=automount,dc=lab,dc=stri"

adding new entry "cn=/ahome,ou=auto.master,ou=automount,dc=lab,dc=stri"

adding new entry "ou=auto.home,ou=automount,dc=lab,dc=stri"

adding new entry "cn=*,ou=auto.home,ou=automount,dc=lab,dc=stri"

```