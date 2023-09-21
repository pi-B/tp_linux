# TP 4 - LDAP et NFS

### Date 15/09/2023
### Eleve : Pierre Bayle

## Mise en place du serveur NFS

On installe les paquets nécessaire et on vérifie leur exécution derrière les ports attendus : 

```
etu@server:~/ldif$ sudo apt install rpcbind
etu@server:~/ldif$ ss -tau | grep rpc
udp   UNCONN    0      0                                   0.0.0.0:sunrpc             0.0.0.0:*          
udp   UNCONN    0      0                                      [::]:sunrpc                [::]:*          
tcp   LISTEN    0      4096                                0.0.0.0:sunrpc             0.0.0.0:*          
tcp   LISTEN    0      4096                                   [::]:sunrpc                [::]:*

etu@server:~/ldif$ sudo apt install nfs-kernel-server
etu@server:~/ldif$ sudo apt install nfs-common
```

Une fois le service installé on configure les dossiers à exporter dans /etc/exports :

```
etu@server:~/ldif$ sudo mkdir -p /home/exports/home

etu@server:~/ldif$ cat /etc/exports 
# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

/home/exports		192.168.9.11/28(rw,sync,fsid=0,crossmnt,no_subtree_check)
/home/exports/home	192.168.9.11/28(rw,sync,no_subtree_check)

etu@server:~/ldif$ sudo systemctl restart nfs-kernel-server
etu@server:~/ldif$ systemctl status nfs-kernel-server.service 
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; preset: enabled)
     Active: active (exited) since Sat 2023-09-16 14:51:29 CEST; 9s ago
    Process: 7759 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 7760 ExecStart=/usr/sbin/rpc.nfsd (code=exited, status=0/SUCCESS)
   Main PID: 7760 (code=exited, status=0/SUCCESS)
        CPU: 6ms

sept. 16 14:51:29 server systemd[1]: Starting nfs-server.service - NFS server and services...
sept. 16 14:51:29 server systemd[1]: Finished nfs-server.service - NFS server and services.

etu@server:~/ldif$ sudo exportfs 
/home/exports 	192.168.9.1/28
/home/exports/home
		192.168.9.1/28
```
Cote client on effectue la vérification des dossiers disponibles à l'export : 
```
etu@client:~$ sudo showmount -e 192.168.9.11
Export list for 192.168.9.11:
/home/exports/home 192.168.9.1/28
/home/exports      192.168.9.1/28
etu@client:~$ 
```

On met en place sur le serveur l'arborescence qui permettra de garantir la cohérence du schéma de nommage entre le serveur et le client.

```
etu@server:~/ldif$ sudo mount --bind /home/exports/home /ahome/
etu@server:~/ldif$ ls -la /home/exports/home/
``` 

## Mise en oeuvre de l'annuaire LDAP

Création des machines :

```
tp_nsf_ldap → ./startup.sh -v 230 -c 425 -s 426
~> iSCSI lab VLAN identifier: 230
~> server VM tap port number: 426
~> client VM tap port number: 425
Configuring tap426 port...
Configuring tap425 port...
Copying server_img.qcow2 image file...
Copying client_img.qcow2 image file...
Creating server_vol.qcow2 disk...
Formatting '/home/pierrebayle/vm/ldapNFS_lab/server_vol.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=on compression_type=zstd size=34359738368 lazy_refcounts=on refcount_bits=16
Creating client_vol.qcow2 disk...
Formatting '/home/pierrebayle/vm/ldapNFS_lab/client_vol.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=on compression_type=zstd size=34359738368 lazy_refcounts=on refcount_bits=16
~> Virtual machine filename   : client_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6325
~> telnet console port number : 2725
~> MAC address                : b8:ad:ca:fe:01:a9
~> Switch port interface      : tap425, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1a9%vlan230
~> Virtual machine filename   : server_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6326
~> telnet console port number : 2726
~> MAC address                : b8:ad:ca:fe:01:aa
~> Switch port interface      : tap426, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1aa%vlan230
```

```
etu@server:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s1
iface enp0s1 inet static
	address     
	gateway 192.168.9.1
# This is an autoconfigured IPv6 interface
iface enp0s1 inet6 auto

etu@vm0:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s1
iface enp0s1 inet static
	address 192.168.9.12/28
	gateway 192.168.9.1
# This is an autoconfigured IPv6 interface
iface enp0s1 inet6 auto

```
Mot de passe serveur ldap : ldappassword

On installe les paquets necéssaires à la mise en route du serveur ldap, puis on vérifie dans les processus s'exécutant que openldap est présent :

```
etu@server:~$ ps aux | grep  l[d]ap
openldap    1410  0.0  1.0 1159640 10028 ?       Ssl  19:45   0:00 /usr/sbin/slapd -h ldap:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d

etu@server:~$ ss -tau 
Netid    State     Recv-Q    Send-Q       Local Address:Port         Peer Address:Port    Process    
tcp      LISTEN    0         128                0.0.0.0:ssh               0.0.0.0:*                  
tcp      LISTEN    0         128                0.0.0.0:2222              0.0.0.0:*                  
tcp      LISTEN    0         2048               0.0.0.0:ldap              0.0.0.0:*                  
tcp      LISTEN    0         128                   [::]:ssh                  [::]:*                  
tcp      LISTEN    0         128                   [::]:2222                 [::]:*                  
tcp      LISTEN    0         2048                  [::]:ldap                 [::]:*    

```

Mise en service d'un nouvel annuaire :

```
etu@server:~$ sudo systemctl stop slapd
etu@server:~$ sudo rm -rf /var/lib/ldap/* /etc/ldap/slapd.d
etu@server:~$ sudo dpkg-reconfigure slapd

  Creating initial configuration... done.                                                           
  Creating LDAP directory... done.

etu@server:~$ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" \
  olcSuffix | grep ^olcSuffix
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcSuffix: dc=corelia,dc=stri

```

La base LDAP a bien été configuré, avec le nom d'organisation corellia.stri. On utilise un LDIF pour changer la valeur de l'attribut olcLogLevel :

``` 
etu@server:~/ldif$ sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f setolcLogLevel2stats.ldif 
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

etu@server:~/ldif$ sudo ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config"   \  olcLogLevel | grep olcLogLevel
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
olcLogLevel: stats
```

On ajoute les deux OU people et group avec un autre fichier LDIF :
``` 
etu@server:~/ldif$ cat createorganisationunit.ldif 
# Add people and groups OU
dn: ou=people,dc=corellia,dc=stri
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=corellia,dc=stri
objectClass: organizationalUnit
ou: groups

etu@server:~/ldif$ sudo ldapadd -cxWD cn=admin,dc=corellia,dc=stri -f createorganisationunit.ldif
adding new entry "ou=people,dc=corellia,dc=stri"

adding new entry "ou=groups,dc=corellia,dc=stri"
```

Puis on créé les quatres comptes utilisateurs, en commencant par créer un mot de passe commun : 

```
etu@server:~/ldif$ sudo slappasswd
New password: 
Re-enter new password: 
{SSHA}EnbUMgHEuwQ3S77njAvsjFw1wjyTif9Q

etu@server:~/ldif$ cat createAccounts.ldif 
# Create 4 accounts for Padme, Anakin, Luke and Leila

dn: uid=padme,ou=people,dc=corellia,dc=stri
objectClass : inetOrgPerson
objectClass : shadowAccount
objectClass : posixAccount
cn : Padme
sn : Padme Amidala Skywalker
uidNumber : 10000
gidNumber : 10000
loginShell : /bin/bash
homeDirectory : /ahome/padme
userPassword : {SSHA}EnbUMgHEuwQ3S77njAvsjFw1wjyTif9Q
gecos: Padme Amidala Skywalker

dn: uid=anakin,ou=people,dc=corellia,dc=stri
objectClass : inetOrgPerson
objectClass : shadowAccount
objectClass : posixAccount
cn : Anakin
sn : Anakin Skywalker
uid : anakin
uidNumber : 10001
gidNumber : 10001
loginShell : /bin/bash
homeDirectory : /ahome/anakin
userPassword : {SSHA}EnbUMgHEuwQ3S77njAvsjFw1wjyTif9Q
gecos: Anakin Skywalker

dn: uid=leila,ou=people,dc=corellia,dc=stri
objectClass : inetOrgPerson
objectClass : shadowAccount
objectClass : posixAccount
cn : Leila
sn : Leila Organa Skywalker
uid: leila
uidNumber : 10002
gidNumber : 10002
loginShell : /bin/bash
homeDirectory : /ahome/leila
userPassword : {SSHA}EnbUMgHEuwQ3S77njAvsjFw1wjyTif9Q
gecos: Leila Organa Skywalker

dn: uid=luke,ou=people,dc=corellia,dc=stri
objectClass : inetOrgPerson
objectClass : shadowAccount
objectClass : posixAccount
cn : Luke
sn : Luke Skywalker
uid : luke
uidNumber : 10003
gidNumber : 10003
loginShell : /bin/bash
homeDirectory : /ahome/luke
userPassword : {SSHA}EnbUMgHEuwQ3S77njAvsjFw1wjyTif9Q
gecos: Luke Amidala Skywalker

etu@server:~/ldif$ 

sudo ldapsearch -LLL -x -H ldap:/// -b "dc=corellia,dc=stri" \
  -D cn=admin,dc=corellia,dc=stri -W

  etu@server:~/ldif$ sudo ldapadd -cxWD cn=admin,dc=corellia,dc=stri -f createAccounts.ldif 
Enter LDAP Password: 
adding new entry "uid=padme,ou=people,dc=corellia,dc=stri"

adding new entry "uid=anakin,ou=people,dc=corellia,dc=stri"

adding new entry "uid=luke,ou=people,dc=corellia,dc=stri"

adding new entry "uid=leila,ou=people,dc=corellia,dc=stri"

```


## Automatisation de la creation des homedir pour les utilisateurs de l'annuaire 

```
etu@server:~$ sudo apt install libpam-ldapd
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait      
● oddjobd.service - privileged operations for unprivileged applications
     Loaded: loaded (/lib/systemd/system/oddjobd.service; enabled; preset: enabled)
     Active: active (running) since Sat 2023-09-16 15:38:22 CEST; 3min 8s ago
   Main PID: 8603 (oddjobd)
      Tasks: 1 (limit: 1082)
     Memory: 760.0K
        CPU: 10ms
     CGroup: /system.slice/oddjobd.service
             └─8603 /usr/sbin/oddjobd -n -p /run/oddjobd.pid -t 300

sept. 16 15:38:22 server systemd[1]: Started oddjobd.service - privileged operations for unprivileged appli>
~
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
session [default=1]                     pam_permit.so
# here's the fallback if no module succeeds
session requisite                       pam_deny.so
# prime the stack with a positive return value if there isn't one already;
# this avoids us returning an error just because nothing sets a success code
# since the modules above will each just jump around
session required                        pam_permit.so
# and here are more per-package modules (the "Additional" block)
session required        pam_unix.so
session [success=ok default=ignore]     pam_ldap.so minimum_uid=1000
session optional        pam_systemd.so
# end of pam-auth-update config
session optional pam_oddjob_mkhomedir.so skel=/etc/skel/ umask=0022

```
On teste de se connecter sur un compte de l'annuaire depuis le client

```
etu@client:~$ ssh padme@192.168.9.11
The authenticity of host '192.168.9.11 (192.168.9.11)' can't be established.
ED25519 key fingerprint is SHA256:yFLaZk+OfY7z7bHyHPXgjowRS4KMHjfoMQxracRdG9M.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.9.11' (ED25519) to the list of known hosts.
padme@192.168.9.11's password: 
Creating home directory for padme.
Linux server 6.4.0-4-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.4.13-1 (2023-08-31) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.

padme@server:~$ pwd
/ahome/padme
padme@server:~$ who
etu      ttyS0        2023-09-15 19:39
padme    pts/0        2023-09-16 15:49 (192.168.9.12)
padme@server:~$ whoami
padme
padme@server:~$ ls -la
total 20
drwxr-xr-x 2 padme 10000 4096 16 sept. 15:49 .
drwxr-xr-x 3 root  root  4096 16 sept. 15:49 ..
-rw-r--r-- 1 padme 10000  220 16 sept. 15:49 .bash_logout
-rw-r--r-- 1 padme 10000 3526 16 sept. 15:49 .bashrc
-rw-r--r-- 1 padme 10000  807 16 sept. 15:49 .profile

``` 
On constate que le dossier /ahome de l'utilisateur padme a bien été crée dans notre dossier exporté  :

```
etu@server:~$ ls -la /ahome/
total 12
drwxr-xr-x  3 root  root  4096 16 sept. 15:49 .
drwxr-xr-x 19 root  root  4096 16 sept. 15:18 ..
drwxr-xr-x  3 padme 10000 4096 16 sept. 15:51 padme

etu@server:~$ ls -la /home/exports/home/
total 12
drwxr-xr-x 3 root  root  4096 16 sept. 15:49 .
drwxr-xr-x 3 root  root  4096 16 sept. 14:41 ..
drwxr-xr-x 3 padme 10000 4096 16 sept. 15:51 padme

```

### Mise en place de l'automontage LDAP/NFS

``` 
etu@client:~$ sed -i 's/caseExactMatch/caseExactIA5Match/g' autofs.schema
etu@client:~$ cat autofs.schema 
# Depends upon core.schema and cosine.schema

# OID Base is 1.3.6.1.4.1.2312.4
#
# Attribute types are under 1.3.6.1.4.1.2312.4.1
# Object classes are under 1.3.6.1.4.1.2312.4.2
# Syntaxes are under 1.3.6.1.4.1.2312.4.3

# Attribute Type Definitions

attributetype ( 1.3.6.1.4.1.2312.4.1.2 NAME 'automountInformation'
	DESC 'Information used by the autofs automounter'
	EQUALITY caseExactIA5Match
	SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 SINGLE-VALUE )

objectclass ( 1.3.6.1.4.1.2312.4.2.3 NAME 'automount' SUP top STRUCTURAL
	DESC 'An entry in an automounter map'
	MUST ( cn $ automountInformation $ objectclass )
	MAY ( description ) )

objectclass ( 1.3.6.1.4.1.2312.4.2.2 NAME 'automountMap' SUP top STRUCTURAL
	DESC 'An group of related automount objects'
	MUST ( ou ) )


etu@client:~$ scp autofs.schema etu@192.168.9.11:~
etu@192.168.9.11's password: 
autofs.schema                                                             100%  774   154.9KB/s   00:00    


```

```
etu@server:~$ sudo slaptest -f schema-convert.conf -F schema-convert
config file testing succeeded

sudo cat schema-convert/cn\=config.ldif 
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 a133bea0
dn: cn=config
objectClass: olcGlobal
cn: config
olcConfigFile: schema-convert.conf
olcConfigDir: schema-convert
olcAttributeOptions: lang-
olcAuthzPolicy: none
olcConcurrency: 0
olcConnMaxPending: 100
olcConnMaxPendingAuth: 1000
olcGentleHUP: FALSE
olcIdleTimeout: 0
olcIndexSubstrIfMaxLen: 4
olcIndexSubstrIfMinLen: 2
olcIndexSubstrAnyLen: 4
olcIndexSubstrAnyStep: 2
olcIndexHash64: FALSE
olcIndexIntLen: 4
olcListenerThreads: 1
olcLocalSSF: 71
olcLogLevel: 0
olcMaxFilterDepth: 1000
olcReadOnly: FALSE
olcReverseLookup: FALSE
olcSaslAuxpropsDontUseCopyIgnore: FALSE
olcSaslSecProps: noplain,noanonymous
olcSockbufMaxIncoming: 262143
olcSockbufMaxIncomingAuth: 16777215
olcThreads: 16
olcThreadQueues: 1
olcTLSVerifyClient: never
olcTLSProtocolMin: 0.0
olcToolThreads: 1
olcWriteTimeout: 0
structuralObjectClass: olcGlobal
entryUUID: dd353da4-e9a2-103d-93ee-1dcd5b908083
creatorsName: cn=config
createTimestamp: 20230917123835Z
entryCSN: 20230917123835.957723Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20230917123835Z

etu@server:~$ sudo ldapadd -Y EXTERNAL -H ldapi:/// -f autofs.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=autofs,cn=schema,cn=config"

etu@server:~$ cat ou-autofs.ldif 
dn: ou=automount,dc=corellia,dc=stri
ou: automount
objectClass: top
objectClass: organizationalUnit

dn: ou=auto.master,ou=automount,dc=corellia,dc=stri
ou: auto.master
objectClass: top
objectClass: automountMap

dn: cn=/ahome,ou=auto.master,ou=automount,dc=corellia,dc=stri
cn: /ahome
objectClass: top
objectClass: automount
automountInformation: ldap:ou=auto.home,ou=automount,dc=corellia,dc=stri

dn: ou=auto.home,ou=automount,dc=corellia,dc=stri
ou: auto.home
objectClass: top
objectClass: automountMap

dn: cn=*,ou=auto.home,ou=automount,dc=corellia,dc=stri
cn: *
objectClass: top
objectClass: automount
automountInformation: -fstype=nfs4 192.168.9.11:/home/&


etu@server:~$ sudo ldapadd -cxWD cn=admin,dc=corellia,dc=stri -f ou-autofs.ldif 
Enter LDAP Password: 
adding new entry "ou=automount,dc=corellia,dc=stri"

adding new entry "ou=auto.master,ou=automount,dc=corellia,dc=stri"

adding new entry "cn=/ahome,ou=auto.master,ou=automount,dc=corellia,dc=stri"

adding new entry "ou=auto.home,ou=automount,dc=corellia,dc=stri"

adding new entry "cn=*,ou=auto.home,ou=automount,dc=corellia,dc=stri"

```

## Accès aux ressources LDAP et NFS depuis le client

### Connexion LDAP depus le client

On met en place le Name Switch Service et PAM et on configure une de leur dépendance : nslcd : 

```
tu@client:~$ systemctl status nslcd.service 
● nslcd.service - LSB: LDAP connection daemon
     Loaded: loaded (/etc/init.d/nslcd; generated)
     Active: active (running) since Wed 2023-09-20 15:04:54 CEST; 47s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 2665 ExecStart=/etc/init.d/nslcd start (code=exited, status=0/SUCCESS)
      Tasks: 6 (limit: 1082)
     Memory: 3.6M
        CPU: 36ms
     CGroup: /system.slice/nslcd.service
             └─2676 /usr/sbin/nslcd

sept. 20 15:04:54 client systemd[1]: Starting nslcd.service - LSB: LDAP connection daemon...
sept. 20 15:04:54 client nslcd[2676]: version 0.9.12 starting
sept. 20 15:04:54 client nslcd[2676]: accepting connections
sept. 20 15:04:54 client nslcd[2665]: Starting LDAP connection daemon: nslcd.
sept. 20 15:04:54 client systemd[1]: Started nslcd.service - LSB: LDAP connection daemon.

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
statd:x:104:65534::/var/lib/nfs:/usr/sbin/nologin
nslcd:x:105:109:nslcd name service LDAP connection daemon,,,:/run/nslcd:/usr/sbin/nologin
padme:x:10000:10000:Padme Amidala Skywalker:/ahome/padme:/bin/bash
anakin:x:10001:10001:Anakin Skywalker:/ahome/anakin:/bin/bash
leila:x:10002:10002:Leila Organa Skywalker:/ahome/leila:/bin/bash
luke:x:10003:10003:Luke Skywalker:/ahome/luke:/bin/bash

``` 

On constate qu'on peut récupérer les informations du fichier /etc/.passwd grâce à getent et NSS.
Nous testons maintenant le changement d'identité avec le compte padme, nous constatons que le montage du répertoire home ne se fait pas encore 

```
etu@client:~$ su - padme
Mot de passe : 
su: avertissement : impossible de changer le répertoire vers /ahome/padme: Aucun fichier ou dossier de ce type
padme@client:/home/etu$ 
```

### Configuration NFS automontage

On modifie le contenue du fichier de configuration de autofs (/etc/default/autofs)

Une fois le service relancé on teste de se connecter avec LDAP depuis le client et on vérifie quel est notre répertoire de connexion avec pwd.

```
etu@client:~$ sudo systemctl status autofs.service 
● autofs.service - Automounts filesystems on demand
     Loaded: loaded (/lib/systemd/system/autofs.service; enabled; preset: enabled)
     Active: active (running) since Wed 2023-09-20 16:12:00 CEST; 2s ago
       Docs: man:autofs(8)
    Process: 3142 ExecStart=/usr/sbin/automount $OPTIONS --pid-file /var/run/autofs.pid (code=exited>
   Main PID: 3143 (automount)
      Tasks: 4 (limit: 1082)
     Memory: 4.2M
        CPU: 49ms
     CGroup: /system.slice/autofs.service
             └─3143 /usr/sbin/automount --pid-file /var/run/autofs.pid

sept. 20 16:12:00 client systemd[1]: Starting autofs.service - Automounts filesystems on demand...
sept. 20 16:12:00 client (utomount)[3142]: autofs.service: Referenced but unset environment variable>
sept. 20 16:12:00 client automount[3143]: Starting automounter version 5.1.8, master map ou=auto.mas>
sept. 20 16:12:00 client automount[3143]: using kernel protocol version 5.05
sept. 20 16:12:00 client automount[3143]: connected to uri ldap://192.168.9.11
sept. 20 16:12:00 client automount[3143]: mounted indirect on /ahome with timeout 300, freq 75 secon>
sept. 20 16:12:00 client systemd[1]: Started autofs.service - Automounts filesystems on demand.
etu@client:~$ su - luke
Mot de passe : 
luke@client:~$ pwd
/ahome/luke
```

On peut voir dans les logs de autofs le resultat de nos actions : 

```
 autofs.service - Automounts filesystems on demand
     Loaded: loaded (/lib/systemd/system/autofs.service; enabled; preset: enabled)
     Active: active (running) since Wed 2023-09-20 16:12:00 CEST; 7min ago
       Docs: man:autofs(8)
    Process: 3142 ExecStart=/usr/sbin/automount $OPTIONS --pid-file /var/run/autofs.pid (code=exited>
   Main PID: 3143 (automount)
      Tasks: 4 (limit: 1082)
     Memory: 6.4M
        CPU: 99ms
     CGroup: /system.slice/autofs.service
             └─3143 /usr/sbin/automount --pid-file /var/run/autofs.pid

sept. 20 16:12:00 client automount[3143]: using kernel protocol version 5.05
sept. 20 16:12:00 client automount[3143]: connected to uri ldap://192.168.9.11
sept. 20 16:12:00 client automount[3143]: mounted indirect on /ahome with timeout 300, freq 75 secon>
sept. 20 16:12:00 client systemd[1]: Started autofs.service - Automounts filesystems on demand.
sept. 20 16:12:17 client automount[3143]: attempting to mount entry /ahome/luke
sept. 20 16:12:17 client automount[3143]: set_tsd_user_vars: failed to get group info from getgrgid_r
sept. 20 16:12:17 client automount[3143]: connected to uri ldap://192.168.9.11
sept. 20 16:12:17 client automount[3143]: mounted /ahome/luke
sept. 20 16:18:32 client automount[3143]: expiring path /ahome/luke
sept. 20 16:18:32 client automount[3143]: expired /ahome/luke
~

luke@client:~$ mount | egrep '(ldap|nfs)'
ldap:ou=auto.home,ou=automount,dc=corellia,dc=stri on /ahome type autofs (rw,relatime,fd=7,pgrp=3143,timeout
=300,minproto=5,maxproto=5,indirect,pipe_ino=26391)
192.168.9.11:/home/luke on /ahome/luke type nfs4 (rw,relatime,vers=4.2,rsize=131072,wsize=131072,namlen=255,
hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.9.12,local_lock=none,addr=192.168.9.11)
```