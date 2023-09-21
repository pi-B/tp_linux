Corellia 192.168.221.129/25
fe80:dd::1
2001:678:3fc:dd::1/64
VLAN : 221

[ ] Configuration serveur avec définition du domaine
[ ] Affichage du contenu de l'annuaire côté serveur
[ ] Accès aux objets de l'annuaire côté client (commande getent)
[ ] Service Web pages blanches avec photos côté client

# TP 3 - Introduction aux annuaires LDAP

### Date : 08/09/23
### Eleve : Pierre BAYLE


```
./startup_LDAP.sh -v 230 -s 425 -c 426
~> iSCSI lab VLAN identifier: 230
~> Target VM tap port number: 425
~> Initiator VM tap port number: 426
Configuring tap425 port...
Configuring tap426 port...
octogone \n sans regles
~> Virtual machine filename   : server_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6325
~> telnet console port number : 2725
~> MAC address                : b8:ad:ca:fe:01:a9
~> Switch port interface      : tap425, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1a9%vlan230
octogone \n sans regles
~> Virtual machine filename   : client_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6326
~> telnet console port number : 2726
~> MAC address                : b8:ad:ca:fe:01:aa
~> Switch port interface      : tap426, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1aa%vlan230
```

## Configuration serveur + définition domaine

On installe les paquets nécessaires et on vérifie que les processus s'exécutent bien sur les ports attendus.
Le serveur a pour nom de domaine "corellia.stri"
Mot de passe serveur LDAP : -etu-

```
etu@serveurLDAP:~$ ps aux | grep l[d]ap
openldap    1352  0.0  1.2 1159776 12336 ?       Ssl  16:47   0:00 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d

etu@serveurLDAP:~$ sudo lsof -i | grep l[d]ap
slapd     1352 openldap    8u  IPv4  20952      0t0  TCP *:ldap (LISTEN)
slapd     1352 openldap    9u  IPv6  20953      0t0  TCP *:ldap (LISTEN)

etu@serveurLDAP:~$ ss -tau | grep ldap
tcp   LISTEN 0      2048         0.0.0.0:ldap        0.0.0.0:*          
tcp   LISTEN 0      2048            [::]:ldap           [::]:*          

etu@serveurLDAP:~$ less /etc/services | grep ldap
ldap		389/tcp			# Lightweight Directory Access Protocol
ldap		389/udp
ldaps		636/tcp				# LDAP over SSL
ldaps		636/udp

```
### Configuration service

On effectue la configuration du serveur LDAP, en changeant le niveau de log

```
etu@serveurLDAP:~$ sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" \
> olcDatabase={1}mdb
[sudo] Mot de passe de etu : 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
# extended LDIF
#
# LDAPv3
# base <cn=config> with scope subtree
# filter: olcDatabase={1}mdb
# requesting: ALL
#

# {1}mdb, config
dn: olcDatabase={1}mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: {1}mdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=nodomain
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by * read
olcLastMod: TRUE
olcRootDN: cn=admin,dc=nodomain
olcRootPW: {SSHA}IM4XSW1ErHOWoW89P7DhokKzkhVAR5iN
olcDbCheckpoint: 512 30
olcDbIndex: objectClass eq
olcDbIndex: cn,uid eq
olcDbIndex: uidNumber,gidNumber eq
olcDbIndex: member,memberUid eq
olcDbMaxSize: 1073741824

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1

etu@serveurLDAP:~$ sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" olcDatabase={1}mdb \ olcSchemaConfig | grep ^cn
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0

```
```
etu@serveurLDAP:~$ sudo systemctl status slapd.service 
○ slapd.service - LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)
     Loaded: loaded (/etc/init.d/slapd; generated)
    Drop-In: /usr/lib/systemd/system/slapd.service.d
             └─slapd-remain-after-exit.conf
     Active: inactive (dead) since Fri 2023-09-08 17:35:59 CEST; 15s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 629 ExecStart=/etc/init.d/slapd start (code=exited, status=0/SUCCESS)
    Process: 877 ExecStop=/etc/init.d/slapd stop (code=exited, status=0/SUCCESS)
        CPU: 103ms

sept. 08 19:14:08 serveurLDAP slapd[669]: slapd starting
sept. 08 19:14:08 serveurLDAP slapd[629]: Starting OpenLDAP: slapd.
sept. 08 19:14:08 serveurLDAP systemd[1]: Started slapd.service - LSB: OpenLDAP standalone server (L>
sept. 08 17:35:59 serveurLDAP systemd[1]: Stopping slapd.service - LSB: OpenLDAP standalone server (>
sept. 08 17:35:59 serveurLDAP slapd[669]: daemon: shutdown requested and initiated.
sept. 08 17:35:59 serveurLDAP slapd[669]: slapd shutdown: waiting for 0 operations/tasks to finish
sept. 08 17:35:59 serveurLDAP slapd[669]: slapd stopped.
sept. 08 17:35:59 serveurLDAP slapd[877]: Stopping OpenLDAP: slapd.
sept. 08 17:35:59 serveurLDAP systemd[1]: slapd.service: Deactivated successfully.
sept. 08 17:35:59 serveurLDAP systemd[1]: Stopped slapd.service - LSB: OpenLDAP standalone server (L>

etu@serveurLDAP:~$ sudo rm -rf /var/lib/ldap/* /etc/ldap/slapd.d
etu@serveurLDAP:~$ sudo dpkg-reconfigure slapd

etu@serveurLDAP:~$ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" \
  olcSuffix | grep ^olcSuffix
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcSuffix: dc=corellia,dc=stri

```

### Création d'un nouvel annuaire

on doit ajouter les objets suivants.

    - Deux unités organisationnelles : people et groups.

    - Quatre compte utilisateurs : papa et maman Skywalker ainsi que leurs deux enfants

Nous allons créer ces objets via les fichier LDIF et les commandes du paquet ldap-utils ldapmodify/ldapadd

```
etu@serveurLDAP:~$ sudo ldapsearch -LLL -x -H ldap:/// -b "dc=corellia,dc=stri" \
  -D cn=admin,dc=corellia,dc=stri -W
Enter LDAP Password: 
dn: dc=corellia,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab.stri
dc: lab

etu@serveurLDAP:~$ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "dc=corellia,dc=stri" \
  -D cn=admin,dc=corellia,dc=stri -W
Enter LDAP Password: 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
dn: dc=corellia,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab.stri
dc: lab

etu@serveurLDAP:~$ mkdir ldif && cd ldif
etu@serveurLDAP:~/ldif$ cat > setolcLogLevel2stats.ldif << EOF
> # Set olcLogLevel to "stats"
> dn: cn=config
> changetype: modify
> replace: olcLoglevel
> olcLogLevel: stats
> EOF

etu@serveurLDAP:~/ldif$ sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f setolcLogLevel2stats.ldif 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

etu@serveurLDAP:~/ldif$ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config"   \  olcLogLevel | grep olcLogLevel
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcLogLevel: stats

etu@serveurLDAP:~/ldif$ journalctl -o cat -n 20 -u slapd --grep="conn"
conn=1025 fd=12 closed
conn=1025 op=2 UNBIND
conn=1025 op=1 SEARCH RESULT tag=101 err=0 qtime=0.000007 etime=0.000086 nentries=10 text=
conn=1025 op=1 SRCH attr=1.1 olcLogLevel
conn=1025 op=1 SRCH base="cn=config" scope=2 deref=0 filter="(objectClass=*)"
conn=1025 op=0 RESULT tag=97 err=0 qtime=0.000009 etime=0.000049 text=
conn=1025 op=0 BIND dn="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" mech=EXTERNAL bind_>
conn=1025 op=0 BIND authcid="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" authzid="gidNu>
conn=1025 op=0 BIND dn="" method=163
conn=1025 fd=12 ACCEPT from PATH=/var/run/slapd/ldapi (PATH=/var/run/slapd/ldapi)
conn=1024 fd=12 closed
conn=1024 op=2 UNBIND
conn=1024 op=1 SEARCH RESULT tag=101 err=0 qtime=0.000022 etime=0.000144 nentries=10 text=
conn=1024 op=1 SRCH attr=1.1
conn=1024 op=1 SRCH base="cn=config" scope=2 deref=0 filter="(objectClass=*)"
conn=1024 op=0 RESULT tag=97 err=0 qtime=0.000023 etime=0.000114 text=
conn=1024 op=0 BIND dn="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" mech=EXTERNAL bind_>
conn=1024 op=0 BIND authcid="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" authzid="gidNu>
conn=1024 op=0 BIND dn="" method=163
conn=1024 fd=12 ACCEPT from PATH=/var/run/slapd/ldapi (PATH=/var/run/slapd/ldapi)


```
### Affichage du contenu de l'annuaire côté serveur

On vérifie que l'on a bien accès au contenu ajouté puis on créé les comptes utilisateurs

```

etu@serveurLDAP:~/ldif$ sudo ldapsearch -LLL -x -H ldap:/// -b "dc=corellia,dc=stri" \  -D cn=admin,dc=corellia,dc=stri -W
Enter LDAP Password: 
dn: dc=corellia,dc=stri

dn: ou=groups,dc=corellia,dc=stri

dn: ou=people,dc=corellia,dc=stri

etu@serveurLDAP:~/ldif$ sudo slappasswd 
New password: 
Re-enter new password: 
{SSHA}qj/UNOfAfNJ32aZmLYYD/Nn6km6Edoun
etu@serveurLDAP:~/ldif$ openssl rand -base64 16 | tr -d '=' > user.passwd 
etu@serveurLDAP:~/ldif$ cat user.passwd 
IW+8CQq+SJKStXh21rfsGw
etu@serveurLDAP:~/ldif$ sudo slappasswd -v -h "{SSHA}" -s $(cat user.passwd)
{SSHA}VHhHCwkM3iUHoQsWtnX332IWNMHtxYKV


etu@serveurLDAP:~/ldif$ sudo ldapsearch -LLL -x -H ldap:/// -b "dc=corellia,dc=stri" \
  -D cn=admin,dc=corellia,dc=stri -W
Enter LDAP Password: 
dn: dc=corellia,dc=stri
objectClass: top
objectClass: dcObject
objectClass: organization
o: lab.stri
dc: lab

dn: ou=groups,dc=corellia,dc=stri
objectClass: organizationalUnit
ou: groups

dn: ou=people,dc=corellia,dc=stri
objectClass: organizationalUnit
ou: people

dn: uid=padme,ou=people,dc=corellia,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Padme
sn:: UGFkbcOpIEFtaWRhbGEgU2t5d2Fsa2Vy
uid: padme
uidNumber: 10000
gidNumber: 10000
loginShell: /bin/bash
homeDirectory: /ahome/padme
userPassword:: e1NTSEF9aEZHb3V1K3l0Zm5IMHFQeTd5OUcwTDBSYjZSNnNsWjQ=
gecos: Padme Amidala Skywalker

dn: uid=anakin,ou=people,dc=corellia,dc=stri
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
userPassword:: e1NTSEF9aEZHb3V1K3l0Zm5IMHFQeTd5OUcwTDBSYjZSNnNsWjQ=
gecos: Anakin Skywalker

dn: uid=leia,ou=people,dc=corellia,dc=stri
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
userPassword:: e1NTSEF9aEZHb3V1K3l0Zm5IMHFQeTd5OUcwTDBSYjZSNnNsWjQ=
gecos: Leia Organa Skywalker

dn: uid=luke,ou=people,dc=corellia,dc=stri
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
userPassword:: e1NTSEF9aEZHb3V1K3l0Zm5IMHFQeTd5OUcwTDBSYjZSNnNsWjQ=
gecos: Luke Skywalker
```

## Affichage du contenu de l'annuaire à distance

Nous passons sur la machine client et utilisons ldapsearch pour afficher le contenu d'un annuaire LDAP dont on a donné l'adresse IP :

```
etu@client:~$ sudo ldapsearch -LLL -H ldap://192.168.221.136   -b dc=corellia,dc=stri -D cn=admin,dc=corellia,dc=stri -W uid=padme
Enter LDAP Password: 
dn: uid=padme,ou=people,dc=corellia,dc=stri
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: posixAccount
cn: Padme
sn:: UGFkbcOpIEFtaWRhbGEgU2t5d2Fsa2Vy
uid: padme
uidNumber: 10000
gidNumber: 10000
loginShell: /bin/bash
homeDirectory: /ahome/padme
userPassword:: e1NTSEF9aEZHb3V1K3l0Zm5IMHFQeTd5OUcwTDBSYjZSNnNsWjQ=
gecos: Padme Amidala Skywalker

etu@client:~$ sudo ldappasswd -x -H ldap://192.168.221.136  -D cn=admin,dc=corellia,dc=stri -W -S uid=padme,ou=people,dc=corellia,dc=stri
New password: 
Re-enter new password: 
passwords do not match
etu@client:~$ sudo ldappasswd -x -H ldap://192.168.221.136  -D cn=admin,dc=corellia,dc=stri -W -S uid=padme,ou=people,dc=corellia,dc=stri
New password: 
Re-enter new password: 
Enter LDAP Password: 

-- nouveau mot de passe : motdepasse123

etu@client:~$ sudo ldapsearch -LLL -H ldap://192.168.221.136   -b dc=corellia,dc=stri -D cn=admin,dc=corellia,dc=stri -W uid=padme
Enter LDAP Password: 
dn: uid=padme,ou=people,dc=corellia,dc=stri
...
userPassword:: e1NTSEF9bWVmNC9GcjRTcElQZ1JTamhxaitCUndkY0xGcUdDYWM=

```

### Configuration Name Service Switch

```
 ┌───────────────────────────────────┤ Configuration de libnss-ldapd ├───────────────────────────────────┐
 │ Le fichier /etc/nsswitch.conf doit être modifié (afin d'utiliser la source de données « ldap ») pour  │
 │ rendre ce paquet fonctionnel.                                                                         │
 │                                                                                                       │
 │ Vous pouvez aussi choisir les services qui doivent être activés ou désactivés pour les requêtes       │
 │ LDAP. Les nouvelles requêtes LDAP seront ajoutées comme dernière source possible. Il est important    │
 │ de bien contrôler ces modifications.                                                                  │
 │                                                                                                       │
 │ Services de nom à configurer :                                                                        │
 │                                                                                                       │
 │  [*] passwd                                                                                           │
 │  [*] group                                                                                            │
 │  [*] shadow                                                                                           │
 │  [ ] hosts                                                                                            │
 │  [ ] networks                                                                                         │
 │  [ ] ethers                                                                                           │
 │  [ ] protocols                                                                                        │
 │  [ ] services                                                                                         │
 │  [ ] rpc                                                                                              │
 │  [ ] netgroup                                                                                         │
 │  [ ] aliases                                                                                          │
 │                                                                                                       │
 │                                                <Ok>                                                   │
 │                                                                                                       │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘


/etc/nsswitch.conf: enable LDAP lookups for group                                                          
/etc/nsswitch.conf: enable LDAP lookups for passwd
/etc/nsswitch.conf: enable LDAP lookups for shadow

  ┌─────────────────────────────────────┤ Configuration de nslcd ├──────────────────────────────────────┐
  │ Veuillez indiquer le nom distinctif (« DN ») de la base de recherche du serveur LDAP. Beaucoup de   │
  │ sites utilisent les éléments composant leur nom de domaine à cette fin. Par exemple, le domaine     │
  │ « example.net » utiliserait « dc=example,dc=net ».                                                  │
  │                                                                                                     │
  │ Base de recherche du serveur LDAP :                                                                 │
  │                                                                                                     │
  │ dc=corellia,dc=stri_____________________________________________________________________________________ │
  │                                                                                                     │
  │                            <Ok>                                <Annuler>                            │
  │                                                                                                     │
  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘


         ┌───────────────────────────────┤ Configuration de nslcd ├───────────────────────────────┐
         │ Veuillez choisir le type d'authentification que la base LDAP utilise (si nécessaire).  │
         │                                                                                        │
         │  - aucune : pas d'authentification;                                                    │
         │  - simple : authentification simple avec un identifiant (DN) et un                     │
         │             mot de passe;                                                              │
         │  - SASL   : mécanisme basé sur SASL (« Simple Authentication and                       │
         │             Security Layer ») : méthode simplifiée d'authentification                  │
         │             et de sécurité;                                                            │
         │                                                                                        │
         │ Authentification LDAP :                                                                │
         │                                                                                        │
         │                                        aucune                                          │
         │                                        simple                                          │
         │                                        SASL                                            │
         │                                                                                        │
         │                                                                                        │
         │                        <Ok>                            <Annuler>                       │
         │                                                                                        │
         └────────────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────┤ Configuration de nslcd ├──────────────────────────────────────┐
  │ Veuillez indiquer le compte à utiliser pour s'identifier sur la base LDAP. Cette valeur doit être   │
  │ indiquée sur la forme d'un nom distinctif (DN : « Distinguished Name »).                            │
  │                                                                                                     │
  │ Utilisateur de la base LDAP :                                                                       │
  │                                                                                                     │
  │ cn=admin,dc=corellia,dc=stri____________________________________________________________________________ │
  │                                                                                                     │
  │                            <Ok>                                <Annuler>                            │
  │                                                                                                     │
  └─────────────────────────────────────────────────────────────────────────────────────────────────────┘


          ┌─────────────────────────────┤ Configuration de nslcd ├──────────────────────────────┐
          │                                                                                     │
          │ Veuillez choisir si la connexion au serveur LDAP doit être chiffrée avec StartTLS.  │
          │                                                                                     │
          │ Faut-il utiliser StartTLS ?                                                         │
          │                                                                                     │
          │                        <Oui>                           <Non>                        │
          │                                                                                     │
          └─────────────────────────────────────────────────────────────────────────────────────┘

● nslcd.service - LSB: LDAP connection daemon
     Loaded: loaded (/etc/init.d/nslcd; generated)
     Active: active (running) since Sat 2023-09-09 13:25:28 CEST; 13s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 1726 ExecStart=/etc/init.d/nslcd start (code=exited, status=0/SUCCESS)
      Tasks: 6 (limit: 1082)
     Memory: 1.7M
        CPU: 37ms
     CGroup: /system.slice/nslcd.service
             └─1736 /usr/sbin/nslcd

sept. 09 13:25:28 client systemd[1]: Starting nslcd.service - LSB: LDAP connection daemon...
sept. 09 13:25:28 client nslcd[1736]: version 0.9.12 starting
sept. 09 13:25:28 client nslcd[1736]: accepting connections
sept. 09 13:25:28 client nslcd[1726]: Starting LDAP connection daemon: nslcd.
sept. 09 13:25:28 client systemd[1]: Started nslcd.service - LSB: LDAP connection daemon.
etu@client:~$ systemctl status nscd.service 
● nscd.service - Name Service Cache Daemon
     Loaded: loaded (/lib/systemd/system/nscd.service; enabled; preset: enabled)
     Active: active (running) since Sat 2023-09-09 13:25:33 CEST; 50s ago
    Process: 1749 ExecStart=/usr/sbin/nscd (code=exited, status=0/SUCCESS)
   Main PID: 1750 (nscd)
      Tasks: 11 (limit: 1082)
     Memory: 2.9M
        CPU: 10ms
     CGroup: /system.slice/nscd.service
             └─1750 /usr/sbin/nscd

sept. 09 13:25:33 client nscd[1750]: 1750 fichier de surveillance `/etc/nsswitch.conf` (7)
sept. 09 13:25:33 client nscd[1750]: 1750 répertoire de surveillance `/etc` (2)
sept. 09 13:25:33 client nscd[1750]: 1750 fichier de surveillance `/etc/nsswitch.conf` (7)
sept. 09 13:25:33 client nscd[1750]: 1750 répertoire de surveillance `/etc` (2)
sept. 09 13:25:33 client nscd[1750]: 1750 fichier de surveillance `/etc/nsswitch.conf` (7)
sept. 09 13:25:33 client nscd[1750]: 1750 répertoire de surveillance `/etc` (2)
sept. 09 13:25:33 client nscd[1750]: 1750 fichier de surveillance `/etc/nsswitch.conf` (7)
sept. 09 13:25:33 client nscd[1750]: 1750 répertoire de surveillance `/etc` (2)
sept. 09 13:25:33 client systemd[1]: Started nscd.service - Name Service Cache Daemon.
sept. 09 13:25:52 client nscd[1750]: 1750 recherche fichier surveillé `/etc/netgroup': Aucun fichier 

etu@client:~$ cat /etc/nsswitch.conf 
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files systemd ldap
group:          files systemd ldap
shadow:         files systemd ldap
gshadow:        files systemd

hosts:          files dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis

etu@client:~$ getent passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
etu:x:1000:1000:Etudiant.e,,,:/home/etu:/bin/bash
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
rdnssd:x:102:65534::/var/run/rdnssd:/usr/sbin/nologin
_rpc:x:103:65534::/run/rpcbind:/usr/sbin/nologin
tcpdump:x:104:109::/nonexistent:/usr/sbin/nologin
statd:x:105:65534::/var/lib/nfs:/usr/sbin/nologin
etu-nfs:x:1001:1001:,,,:/ahome/etu-nfs:/bin/bash
nslcd:x:106:110:nslcd name service LDAP connection daemon,,,:/run/nslcd:/usr/sbin/nologin
padme:x:10000:10000:Padme Amidala Skywalker:/ahome/padme:/bin/bash
anakin:x:10001:10001:Anakin Skywalker:/ahome/anakin:/bin/bash
leia:x:10002:10002:Leia Organa Skywalker:/ahome/leia:/bin/bash
luke:x:10003:10003:Luke Skywalker:/ahome/luke:/bin/bash


etu@client:~$ getent shadow
padme:*:::::::0
anakin:*:::::::0
leia:*:::::::0
luke:*:::::::0

```

### Optionnel : visualisation de l'authentification LDAP avec tshark

```
etu@serveurLDAP:~$ sudo tshark -i enp0s1 -f "port 389" > ldap.txt
Running as user "root" and group "root". This could be dangerous.
Capturing on 'enp0s1'
 ** (tshark:2372) 13:53:40.234609 [Main MESSAGE] -- Capture started.
 ** (tshark:2372) 13:53:40.234747 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s1X5Z4A2.pcapng"
64 ^C
tshark: 
etu@serveurLDAP:~$ cat ldap.txt 
    1 0.000000000 192.168.221.137 → 192.168.221.136 LDAP 242 searchRequest(8) "dc=corellia,dc=stri" wholeSubtree 
    2 0.000779518 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(8) success  [0 results]
    3 0.002456507 192.168.221.137 → 192.168.221.136 LDAP 242 searchRequest(9) "dc=corellia,dc=stri" wholeSubtree 
    4 0.003152365 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(9) success  [0 results]
    5 0.046655768 192.168.221.137 → 192.168.221.136 TCP 66 32848 → 389 [ACK] Seq=353 Ack=29 Win=501 Len=0 TSval=2410628322 TSecr=798459300
    6 6.604922953 192.168.221.137 → 192.168.221.136 LDAP 256 searchRequest(16) "dc=corellia,dc=stri" wholeSubtree 
    7 6.605658118 192.168.221.136 → 192.168.221.137 LDAP 127 searchResEntry(16) "uid=padme,ou=people,dc=corellia,dc=stri" 
    8 6.606050657 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(16) success  [1 result]
    9 6.606749614 192.168.221.137 → 192.168.221.136 TCP 66 53920 → 389 [ACK] Seq=191 Ack=62 Win=502 Len=0 TSval=2410634881 TSecr=798465903
   10 6.606894450 192.168.221.137 → 192.168.221.136 TCP 66 53920 → 389 [ACK] Seq=191 Ack=76 Win=502 Len=0 TSval=2410634882 TSecr=798465903
   11 10.379893806 192.168.221.137 → 192.168.221.136 LDAP 256 searchRequest(10) "dc=corellia,dc=stri" wholeSubtree 
   12 10.380587906 192.168.221.136 → 192.168.221.137 LDAP 127 searchResEntry(10) "uid=padme,ou=people,dc=corellia,dc=stri" 
   13 10.380974021 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(10) success  [1 result]
   14 10.381506225 192.168.221.137 → 192.168.221.136 TCP 66 32848 → 389 [ACK] Seq=543 Ack=90 Win=501 Len=0 TSval=2410638656 TSecr=798469678
   15 10.381506557 192.168.221.137 → 192.168.221.136 TCP 66 32848 → 389 [ACK] Seq=543 Ack=104 Win=501 Len=0 TSval=2410638656 TSecr=798469678
   16 10.382357286 192.168.221.137 → 192.168.221.136 LDAP 167 searchRequest(14) "dc=corellia,dc=stri" wholeSubtree 
   17 10.382973441 192.168.221.136 → 192.168.221.137 LDAP 149 searchResEntry(14) "uid=padme,ou=people,dc=corellia,dc=stri" 
   18 10.383161931 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(14) success  [1 result]
   19 10.383669491 192.168.221.137 → 192.168.221.136 TCP 66 42754 → 389 [ACK] Seq=102 Ack=84 Win=501 Len=0 TSval=2410638659 TSecr=798469680
   20 10.383669832 192.168.221.137 → 192.168.221.136 TCP 66 42754 → 389 [ACK] Seq=102 Ack=98 Win=501 Len=0 TSval=2410638659 TSecr=798469680
   21 10.384067811 192.168.221.137 → 192.168.221.136 TCP 74 55474 → 389 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=2410638659 TSecr=0 WS=128
   22 10.384106471 192.168.221.136 → 192.168.221.137 TCP 74 389 → 55474 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=798469681 TSecr=2410638659 WS=128
   23 10.384868434 192.168.221.137 → 192.168.221.136 TCP 66 55474 → 389 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=2410638660 TSecr=798469681
   24 10.384868807 192.168.221.137 → 192.168.221.136 LDAP 158 bindRequest(1) "uid=padme,ou=people,dc=corellia,dc=stri" simple 
   25 10.384942202 192.168.221.136 → 192.168.221.137 TCP 66 389 → 55474 [ACK] Seq=1 Ack=93 Win=65152 Len=0 TSval=798469682 TSecr=2410638660
   26 10.385643685 192.168.221.136 → 192.168.221.137 LDAP 80 bindResponse(1) success 
   27 10.386424537 192.168.221.137 → 192.168.221.136 TCP 66 55474 → 389 [ACK] Seq=93 Ack=15 Win=64256 Len=0 TSval=2410638661 TSecr=798469683
   28 10.386424870 192.168.221.137 → 192.168.221.136 LDAP 143 searchRequest(2) "uid=padme,ou=people,dc=corellia,dc=stri" baseObject 
   29 10.386653691 192.168.221.136 → 192.168.221.137 LDAP 111 searchResEntry(2) "uid=padme,ou=people,dc=corellia,dc=stri" 
   30 10.386813824 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(2) success  [1 result]
   31 10.387496966 192.168.221.137 → 192.168.221.136 LDAP 74 abandonRequest(2) 
   32 10.387497302 192.168.221.137 → 192.168.221.136 LDAP 73 unbindRequest(4) 
   33 10.387497412 192.168.221.137 → 192.168.221.136 TCP 66 55474 → 389 [RST, ACK] Seq=186 Ack=74 Win=64256 Len=0 TSval=2410638662 TSecr=798469684
   34 10.387711007 192.168.221.137 → 192.168.221.136 LDAP 256 searchRequest(15) "dc=corellia,dc=stri" wholeSubtree 
   35 10.388558086 192.168.221.136 → 192.168.221.137 LDAP 127 searchResEntry(15) "uid=padme,ou=people,dc=corellia,dc=stri" 
   36 10.388774197 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(15) success  [2 results]
   37 10.389344490 192.168.221.137 → 192.168.221.136 LDAP 74 abandonRequest(15) 
   38 10.391628270 192.168.221.137 → 192.168.221.136 LDAP 167 searchRequest(13) "dc=corellia,dc=stri" wholeSubtree 
   39 10.391659373 192.168.221.136 → 192.168.221.137 TCP 66 389 → 41350 [ACK] Seq=1 Ack=102 Win=502 Len=0 TSval=798469689 TSecr=2410638667
   40 10.392156800 192.168.221.136 → 192.168.221.137 LDAP 149 searchResEntry(13) "uid=padme,ou=people,dc=corellia,dc=stri" 
   41 10.392549093 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(13) success  [1 result]
   42 10.393166015 192.168.221.137 → 192.168.221.136 LDAP 220 searchRequest(14) "dc=corellia,dc=stri" wholeSubtree 
   43 10.393662357 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(14) success  [1 result]
   44 10.393895275 192.168.221.137 → 192.168.221.136 TCP 66 41350 → 389 [ACK] Seq=256 Ack=112 Win=501 Len=0 TSval=2410638669 TSecr=798469690
   45 10.396076100 192.168.221.137 → 192.168.221.136 LDAP 256 searchRequest(11) "dc=corellia,dc=stri" wholeSubtree 
   46 10.396508154 192.168.221.136 → 192.168.221.137 LDAP 127 searchResEntry(11) "uid=padme,ou=people,dc=corellia,dc=stri" 
   47 10.396608810 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(11) success  [2 results]
   48 10.397385922 192.168.221.137 → 192.168.221.136 TCP 66 32848 → 389 [ACK] Seq=733 Ack=179 Win=501 Len=0 TSval=2410638672 TSecr=798469694
   49 10.398013007 192.168.221.137 → 192.168.221.136 LDAP 167 searchRequest(12) "dc=corellia,dc=stri" wholeSubtree 
   50 10.398551611 192.168.221.136 → 192.168.221.137 LDAP 149 searchResEntry(12) "uid=padme,ou=people,dc=corellia,dc=stri" 
   51 10.398732696 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(12) success  [3 results]
   52 10.399307199 192.168.221.137 → 192.168.221.136 LDAP 256 searchRequest(13) "dc=corellia,dc=stri" wholeSubtree 
   53 10.399908854 192.168.221.136 → 192.168.221.137 LDAP 127 searchResEntry(13) "uid=padme,ou=people,dc=corellia,dc=stri" 
   54 10.400102127 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(13) success  [4 results]
   55 10.400762865 192.168.221.137 → 192.168.221.136 TCP 66 32848 → 389 [ACK] Seq=1024 Ack=337 Win=501 Len=0 TSval=2410638676 TSecr=798469696
   56 10.400763195 192.168.221.137 → 192.168.221.136 LDAP 74 abandonRequest(13) 
   57 10.403172147 192.168.221.137 → 192.168.221.136 LDAP 256 searchRequest(17) "dc=corellia,dc=stri" wholeSubtree 
   58 10.403509483 192.168.221.136 → 192.168.221.137 LDAP 127 searchResEntry(17) "uid=padme,ou=people,dc=corellia,dc=stri" 
   59 10.403672258 192.168.221.136 → 192.168.221.137 LDAP 80 searchResDone(17) success  [2 results]
   60 10.404161092 192.168.221.137 → 192.168.221.136 TCP 66 53920 → 389 [ACK] Seq=381 Ack=137 Win=502 Len=0 TSval=2410638679 TSecr=798469701
   61 10.404161380 192.168.221.137 → 192.168.221.136 TCP 66 53920 → 389 [ACK] Seq=381 Ack=151 Win=502 Len=0 TSval=2410638679 TSecr=798469701
   62 10.431391764 192.168.221.136 → 192.168.221.137 TCP 66 389 → 42754 [ACK] Seq=173 Ack=300 Win=501 Len=0 TSval=798469729 TSecr=2410638664
   63 10.443401808 192.168.221.136 → 192.168.221.137 TCP 66 389 → 32848 [ACK] Seq=351 Ack=1032 Win=501 Len=0 TSval=798469741 TSecr=2410638676
   64 10.446653694 192.168.221.137 → 192.168.221.136 TCP 66 32848 → 389 [ACK] Seq=1032 Ack=351 Win=501 Len=0 TSval=2410638722 TSecr=798469697

etu@client:~$ 
etu@client:~$ su - padme
Mot de passe : 
su: avertissement : impossible de changer le répertoire vers /ahome/padme: Aucun fichier ou dossier de ce type
padme@client:/home/etu$ exit
déconnexion
etu@client:~$ journalctl -o cat -n 20 --grep="pam_unix" | grep padme
pam_unix(su-l:session): session closed for user padme
pam_unix(su-l:session): session opened for user padme(uid=10000) by etu(uid=1000)
pam_unix(su-l:auth): authentication failure; logname=etu uid=1000 euid=0 tty=/dev/ttyS0 ruser=etu rhost=  user=padme
pam_unix(su-l:session): session closed for user padme
pam_unix(su-l:session): session opened for user padme(uid=10000) by etu(uid=1000)
pam_unix(su-l:auth): authentication failure; logname=etu uid=1000 euid=0 tty=/dev/ttyS0 ruser=etu rhost=  user=padme
pam_unix(su-l:session): session closed for user padme
pam_unix(su-l:session): session opened for user padme(uid=10000) by etu(uid=1000)
pam_unix(su-l:auth): authentication failure; logname=etu uid=1000 euid=0 tty=/dev/ttyS0 ruser=etu rhost=  user=padme
pam_unix(su-l:session): session closed for user padme
pam_unix(su-l:session): session opened for user padme(uid=10000) by etu(uid=1000)
pam_unix(su-l:auth): authentication failure; logname=etu uid=1000 euid=0 tty=/dev/ttyS0 ruser=etu rhost=  user=padme


etu@client:~$ journalctl -o cat -n 100 -u ssh | grep padme
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=::1  user=padme
Accepted password for padme from ::1 port 59184 ssh2
pam_unix(sshd:session): session opened for user padme(uid=10000) by (uid=0)

```

### Affichage du contenu de l'anuaire sur un site web

On installe un paquet "white-pages" qui permet d'afficher sur un site web le contenu d'un annuaire LDAP.
On peut aussi associer à nos utilisateurs des photos. Pour cela on transforme des fichier JPG en valeur alphanumérique et on les insere dans un fichier LDIF pour modifier les entrées des utilisateurs. 

```
etu@serveurLDAP:~$ wget https://ltb-project.org/archives/white-pages_0.4-2_all.deb
etu@serveurLDAP:~$ sudo dpkg -i white-pages_0.4-2_all.deb
etu@serveurLDAP:~$ sudo apt -y -f install
etu@serveurLDAP:~$ sudo apt install smarty3

etu@serveurLDAP:~$ sudo a2dissite 000-default
Site 000-default disabled.
To activate the new configuration, you need to run:
  systemctl reload apache2
etu@serveurLDAP:~$ sudo a2ensite white-pages
Enabling site white-pages.
To activate the new configuration, you need to run:
  systemctl reload apache2
etu@serveurLDAP:~$ sudo apachectl configtest
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Syntax OK
etu@serveurLDAP:~$ sudo systemctl reload apache2

```
On observe dans le fichier de configuration qu'il faut changer les informations de connrxion au serveur ldap

```etu@serveurLDAP:~$ cat /usr/share/white-pages/conf/config.inc.php 
<?php

# LDAP
$ldap_url = "ldap://localhost";
$ldap_starttls = false;
$ldap_binddn = "cn=manager,dc=example,dc=com";
$ldap_bindpw = "secret";
$ldap_base = "dc=example,dc=com";
$ldap_user_base = "ou=users,".$ldap_base;
$ldap_user_filter = "(objectClass=inetOrgPerson)";
#$ldap_user_regex = "/,ou=users,/i";
$ldap_group_base = "ou=groups,".$ldap_base;
$ldap_group_filter = "(|(objectClass=groupOfNames)(objectClass=groupOfUniqueNames))";
$ldap_size_limit = 100;
#$ldap_network_timeout = 10;
```

On crée le fichier LDIF puis on l'exécute avec ldapmodify pour modifier les entrées spécifiées :

```
etu@serveurLDAP:~$ cat change-photo.ldif 
dn: uid=leia,ou=people,dc=corellia,dc=stri
changetype: modify
add: jpegphoto
jpegphoto:<file:///home/etu/photos/leia.jpg

dn: uid=anakin,ou=people,dc=corellia,dc=stri
changetype: modify
add: jpegphoto
jpegphoto:<file:///home/etu/photos/anakin.jpg

dn: uid=padme,ou=people,dc=corellia,dc=stri
changetype: modify
add: jpegPhoto
jpegPhoto:<file:///home/etu/photos/padme.jpg

dn: uid=luke,ou=people,dc=corellia,dc=stri
changetype: modify
add: jpegPhoto
jpegPhoto:<file:///home/etu/photos/luke.jpg


etu@serveurLDAP:~$ sudo ldapmodify -x -H ldap:/// -D "cn=admin,dc=corellia,dc=stri" -W -f change-photo.ldif
Enter LDAP Password: 
modifying entry "uid=leia,ou=people,dc=corellia,dc=stri"

modifying entry "uid=anakin,ou=people,dc=corellia,dc=stri"

modifying entry "uid=padme,ou=people,dc=corellia,dc=stri"

modifying entry "uid=luke,ou=people,dc=corellia,dc=stri"

```

### Résultat

  