# TP4 NFS & LDAP - G2 Planète Teth

## Client 

### Client LDAP

- Lancement de la VM client (ubuntu client)
```
lekienemery@oscar:~/vm/4-asso_ldap_nfsv4$ ./startup-client.sh 
~> Virtual machine filename   : ldapnfsv4-client-ubuntu-desktop.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 5963
~> telnet console port number : 2363
~> MAC address                : b8:ad:ca:fe:00:3f
~> Switch port interface      : tap63, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:3f%vlan305
qxl_send_events: spice-server bug: guest stopped, ignoring

```
- Renommage de la VM
```
etu@vm0:~$ sudo sed -i 's/vm0/client/g' /etc/hosts
[sudo] Mot de passe de etu : 
etu@vm0:~$ sudo sed -i 's/vm0/client/g' /etc/hostname
```

- Configuration de l'interface réseau 
```
etu@client:~$ cat /etc/netplan/01-network-manager-all.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s1:
      dhcp4: false
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s1:
      addresses:
        - 10.0.49.195/27
      nameservers:
        addresses: [172.16.0.2]
      routes:
        - to: default
          via:  10.0.49.193

etu@client:~$ sudo netplan apply
etu@client:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:00:3f brd ff:ff:ff:ff:ff:ff
    inet 10.0.49.195/27 brd 10.0.49.223 scope global noprefixroute enp0s1
       valid_lft forever preferred_lft forever
    inet6 fe80::baad:caff:fefe:3f/64 scope link 
       valid_lft forever preferred_lft forever
```

- Configuration NFS client 

```
etu@client:~$ sudo apt install rpcbind -y
...
etu@client:~$ sudo apt install nfs-common -y
...
etu@client:~$ sudo showmount -e 10.0.49.194
Export list for 10.0.49.194:
/home/exports/home 10.0.49.195/27
/home/exports      10.0.49.195/27

etu@client:~$ sudo mkdir /ahome

etu@client:~$ sudo mount 10.0.49.194:/home /ahome

etu@client:~$ mount | grep nfs
10.0.49.194:/home on /ahome type nfs4 (rw,relatime,vers=4.2,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.0.49.195,local_lock=none,addr=10.0.49.194)

```
### Configuration client LDAP 

```
etu@client:~$ sudo  aptitude install libnss-ldapd -y
...
etu@client:~$ sudo apt install libpam-ldapd -y
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait      
Les NOUVEAUX paquets suivants seront installés :
  libpam-ldapd
0 mis à jour, 1 nouvellement installés, 0 à enlever et 3 non mis à jour.
Il est nécessaire de prendre 16,6 ko dans les archives.
Après cette opération, 89,1 ko d'espace disque supplémentaires seront utilisés.
Réception de :1 http://fr.archive.ubuntu.com/ubuntu jammy/universe amd64 libpam-ldapd amd64 0.9.12-2 [16,6 kB]
16,6 ko réceptionnés en 0s (144 ko/s)   
Sélection du paquet libpam-ldapd:amd64 précédemment désélectionné.
(Lecture de la base de données... 171776 fichiers et répertoires déjà installés.)
Préparation du dépaquetage de .../libpam-ldapd_0.9.12-2_amd64.deb ...
Dépaquetage de libpam-ldapd:amd64 (0.9.12-2) ...
Paramétrage de libpam-ldapd:amd64 (0.9.12-2) ...
Traitement des actions différées (« triggers ») pour man-db (2.10.2-1) ...
etu@client:~$ sudo dpkg-reconfigure libpam-ldapd
etu@client:~$ sudo  dpkg-reconfigure nslcd
etu@client:~$ sudo systemctl restart nscd
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentification requise pour arrêter « nscd.service ».
Authenticating as: Etudiant,,, (etu)
Password: 
==== AUTHENTICATION COMPLETE ===

```

 ### Configuration de l'automontage avec le service LDAP

```
etu@client:~$ sudo aptitude install autofs-ldap -y
<snip>

etu@client:~$ cp /etc/ldap/schema/autofs.schema .
etu@client:~$ sed -i 's/caseExactMatch/caseExactIA5Match/g' autofs.schema
etu@client:~$ scp /etc/ldap/schema/autofs.schema etu@10.0.49.194:~
The authenticity of host '10.0.49.194 (10.0.49.194)' can't be established.
ED25519 key fingerprint is SHA256:yuX6IKxtZwa0265VzGVMtksRhWFfWl9T/HK7B/2u98U.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.49.194' (ED25519) to the list of known hosts.
etu@10.0.49.194's password: 
autofs.schema                                                      100%  774   405.1KB/s   00:00 

```

### Accès aux ressources 
```
etu@client:~$  getent passwd
<snip>
nslcd:x:115:136:nslcd name service LDAP connection daemon,,,:/run/nslcd:/usr/sbin/nologin
padme:x:10000:10000:Padme Amidala Skywalker:/ahome/padme:/bin/bash
anakin:x:10001:10001:Anakin Skywalker:/ahome/anakin:/bin/bash
leia:x:10002:10002:Leia Organa Skywalker:/ahome/leia:/bin/bash
luke:x:10003:10003:Luke Skywalker:/ahome/luke:/bin/bash

etu@client:~$  echo -e "\nautomount: ldap" | sudo tee -a /etc/nsswitch.conf
[sudo] Mot de passe de etu : 

automount: ldap

etu@client:~$ sudo vi /etc/default/autofs 
etu@client:~$ sudo grep -v ^# /etc/default/autofs
USE_MISC_DEVICE="yes"
MASTER_MAP_NAME="ou=auto.master,ou=automount,dc=lab,dc=stri"
TIMEOUT=300
BROWSE_MODE="no"
LOGGING="verbose"
LDAP_URI="ldap://10.0.49.194"
SEARCH_BASE="ou=automount,dc=lab,dc=stri"

etu@client:~$ echo "/ahome /etc/auto.home" | \
sudo tee -a /etc/auto.master.d/ahome.autofs
/ahome /etc/auto.home
etu@client:~$  echo "* -fstype=nfs4 [10.0.49.194]:/home/&" | \
sudo tee -a /etc/auto.home
* -fstype=nfs4 [10.0.49.194]:/home/&


etu@client:~$ sudo systemctl status autofs
● autofs.service - Automounts filesystems on demand
     Loaded: loaded (/lib/systemd/system/autofs.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-09-14 22:19:24 CEST; 44min ago
       Docs: man:autofs(8)
    Process: 7670 ExecStart=/usr/sbin/automount $OPTIONS --pid-file /var/run/autofs.pid (code=exited>
   Main PID: 7671 (automount)
      Tasks: 3 (limit: 1071)
     Memory: 844.0K
        CPU: 23ms
     CGroup: /system.slice/autofs.service
             └─7671 /usr/sbin/automount --pid-file /var/run/autofs.pid

sept. 14 22:19:24 client systemd[1]: Starting Automounts filesystems on demand...
sept. 14 22:19:24 client systemd[1]: Started Automounts filesystems on demand.

etu@client:~$ su - padme
Mot de passe : 
groups: impossible de trouver le nom pour le GID 10000
padme@client:~$

```