# TP3 LDAP - G2 Planète Teth

## Serveur LDAP

- Installation des paquets relatif au serveur LDAP 

```
aptitude search '?description(OpenLDAP)'
┌─[etu][server][~]
└─▪ sudo apt install slapd ldap-utils

┌─[etu][server][~]
└─▪ ps aux | grep l[d]ap
openldap    2844  0.0  0.5 1157924 5408 ?        Ssl  14:33   0:00 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d

┌─[etu][server][~]
└─▪ lsof -i | grep l[d]ap

┌─[etu][server][~]
└─▪ sudo lsof -i | grep ldap
[sudo] password for etu: 
slapd     2844        openldap    8u  IPv4  25453      0t0  TCP *:ldap (LISTEN)
slapd     2844        openldap    9u  IPv6  25454      0t0  TCP *:ldap (LISTEN)

┌─[etu][server][~]
└─▪ grep ldap /etc/services
ldap		389/tcp			# Lightweight Directory Access Protocol
ldap		389/udp
ldaps		636/tcp				# LDAP over SSL
ldaps		636/udp

┌─[etu][server][~]
└─▪ ss -tanp
State           Recv-Q           Send-Q                     Local Address:Port                     Peer Address:Port           Process          
LISTEN          0                2048                             0.0.0.0:389                           0.0.0.0:*                               
LISTEN          0                128                              0.0.0.0:2222                          0.0.0.0:*                               
LISTEN          0                4096                       127.0.0.53%lo:53                            0.0.0.0:*                               
LISTEN          0                128                              0.0.0.0:22                            0.0.0.0:*                               
ESTAB           0                0                            10.0.72.227:22                         172.16.0.7:34284                           
LISTEN          0                2048                                [::]:389                              [::]:*                               
LISTEN          0                128                                 [::]:2222                             [::]:*                               
LISTEN          0                128                                 [::]:22                               [::]:*                 

┌─[etu][server][~]
└─▪ sudo ss -tanp
State      Recv-Q     Send-Q         Local Address:Port         Peer Address:Port     Process                                                   
LISTEN     0          2048                 0.0.0.0:389               0.0.0.0:*         users (("slapd",pid=2844,fd=8))                          
LISTEN     0          128                  0.0.0.0:2222              0.0.0.0:*         users:(("sshd",pid=812,fd=3))                            
LISTEN     0          4096           127.0.0.53%lo:53                0.0.0.0:*         users:(("systemd-resolve",pid=546,fd=14))                
LISTEN     0          128                  0.0.0.0:22                0.0.0.0:*         users:(("sshd",pid=812,fd=5))                            
ESTAB      0          0                10.0.72.227:22             172.16.0.7:34284     users:(("sshd",pid=2991,fd=4),("sshd",pid=2936,fd=4))    
LISTEN     0          2048                    [::]:389                  [::]:*         users:(("slapd",pid=2844,fd=9))                          
LISTEN     0          128                     [::]:2222                 [::]:*         users:(("sshd",pid=812,fd=4))                            
LISTEN     0          128                     [::]:22                   [::]:*         users:(("sshd",pid=812,fd=6))
```

- Supression des fichiers et répertoires
```
┌─[etu][server][~]
└─▪ sudo rm /var/lib/ldap/*
┌─[etu][server][~]
└─▪ sudo rm -rf /etc/ldap/slapd.d/
```

- Configuration et redémarrage du daemon
```
┌─[etu][server][~]
└─▪ dpkg-reconfigure slapd

┌─[etu][server][~]
└─▪ systemctl restart slapd.service 
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to restart 'slapd.service'.
Authenticating as: Etudiant (etu)
Password: 
==== AUTHENTICATION COMPLETE ===
┌─[etu][server][~]
└─▪ systemctl status slapd.service 
● slapd.service - LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)
     Loaded: loaded (/etc/init.d/slapd; generated)
    Drop-In: /usr/lib/systemd/system/slapd.service.d
             └─slapd-remain-after-exit.conf
     Active: active (running) since Fri 2022-09-02 15:08:12 UTC; 9s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 4482 ExecStart=/etc/init.d/slapd start (code=exited, status=0/SUCCESS)
      Tasks: 3 (limit: 1021)
     Memory: 3.8M
        CPU: 50ms
     CGroup: /system.slice/slapd.service
             └─4489 /usr/sbin/slapd -h "ldap:/// ldapi:///" -g openldap -u openldap -F /etc/ldap/slapd.d

sept. 02 15:08:12 server systemd[1]: Starting LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)...
sept. 02 15:08:12 server slapd[4482]:  * Starting OpenLDAP slapd
sept. 02 15:08:12 server slapd[4488]: @(#) $OpenLDAP: slapd 2.5.13+dfsg-0ubuntu0.22.04.1 (Aug  5 2022 14:51:52) $
                                           Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
sept. 02 15:08:12 server slapd[4489]: slapd starting
sept. 02 15:08:12 server slapd[4482]:    ...done.
sept. 02 15:08:12 server systemd[1]: Started LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol).
```

- Validation de la configuration
```
┌─[etu][server][~]
└─▪ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" olcSuffix | grep ^olcSuffix
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcSuffix: dc=teth,dc=stri
┌─[etu][server][~]
└─▪ ldapsearch -LLL -x -H ldap:/// -b "dc=teth,dc=stri" -D cn=admin,dc=teth,dc=stri -W
Enter LDAP Password: 
dn: dc=teth,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: nodomaiqqqqqq
dc: teth
└─▪ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" olcLogLevel | grep ^olcLogLevel
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcLogLevel: none
```

- Activation de la journalisation
```
┌─[etu][server][~]
└─▪ vi setolcLogLevel2stats.ldif 

┌─[etu][server][~]
└─▪ sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f setolcLogLevel2stats.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

┌─[etu][server][~]
└─▪ grep -5 olcLogLevel /var/log/syslog
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=0 BIND dn="" method=163
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=0 BIND authcid="gidNumber=1000+uidNumber=1000,cn=peercred,cn=external,cn=auth" authzid="gidNumber=1000+uidNumber=1000,cn=peercred,cn=external,cn=auth"
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=0 BIND dn="gidNumber=1000+uidNumber=1000,cn=peercred,cn=external,cn=auth" mech=EXTERNAL bind_ssf=0 ssf=71
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=0 RESULT tag=97 err=0 qtime=0.000039 etime=0.000238 text=
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=1 SRCH base="cn=config" scope=2 deref=0 filter="(objectClass=*)"
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=1 SRCH attr=olcLogLevel
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=1 SEARCH RESULT tag=101 err=32 qtime=0.000035 etime=0.000177 nentries=0 text=
Sep  2 15:29:22 server slapd[4684]: conn=1022 op=2 UNBIND
Sep  2 15:29:22 server slapd[4684]: conn=1022 fd=12 closed
```

- Ajout de groupes organisationnels
```
┌─[etu][server][~]
└─▪ vi ou.ldif

┌─[etu][server][~]
└─▪ ldapadd -cxWD cn=admin,dc=teth,dc=stri -f ou.ldif
Enter LDAP Password: 
ldapadd: attributeDescription "dn": (possible missing newline after line 5, entry "ou=people,dc=teth,dc=stri"?)
adding new entry "ou=people,dc=teth,dc=stri"
ldap_add: Type or value exists (20)
	additional info: objectClass: value #0 provided more than once
└─▪ sudo ldapadd -cxWD cn=admin,dc=teth,dc=stri -f ou.ldif
Enter LDAP Password: 
adding new entry "ou=people,dc=teth,dc=stri"

adding new entry "ou=groups,dc=teth,dc=stri"

```

```
┌─[etu][server][~]
└─▪ ldapsearch -LLL -x -H ldap:/// -b "dc=teth,dc=stri" -D cn=admin,dc=teth,dc=stri -W
Enter LDAP Password: 
dn: dc=teth,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: nodomaiqqqqqq
dc: teth

dn: ou=groups,dc=teth,dc=stri
objectClass: organizationalUnit
ou: groups

dn: ou=people,dc=teth,dc=stri
objectClass: organizationalUnit
ou: people
```
- Ajout d'utilisateurs
```
┌─[etu][server][~]
└─▪ vi users.ldif 
┌─[etu][server][~]
└─▪ sudo ldapadd -cxWD cn=admin,dc=teth,dc=stri -f users.ldif
Enter LDAP Password: 
adding new entry "uid=padme,ou=groups,dc=teth,dc=stri"

adding new entry "uid=anakin,ou=groups,dc=teth,dc=stri"

adding new entry "uid=leia,ou=people,dc=teth,dc=stri"

adding new entry "uid=luke,ou=people,dc=teth,dc=stri"
```
