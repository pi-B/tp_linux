dans startup.sh changer "target" par "server" et "initiator" => "client"

# TP2 - NFS

### Date : 30/08/2023
### Eleve : Pierre Bayle

====================================================================

## Dialogue RPC

Gateway pour Corellia : 192.168.130.1/24

- client 192.168.130.40 sur le tap 425
- serveur 192.168.130.41 sur le tap 426

iface enp0s1 inet static
        address 192.168.130.40/24
        gateway 192.168.130.1

Changement du script startup.sh en startup_NFS.sh avec changement des noms de variable


```
#!/bin/bash

BLUE='\e[1;34m'
NC='\e[0m' # No Color

set -euo pipefail

RAM="1024"
VLAN=""
SERVER_TAP=""
CLIENT_TAP=""
MASTER_IMG_NAME="debian-testing-amd64"
LAB_NAME="iscsi_lab"
SECOND_DISK_SIZE="32G"

usage() {
>&2 cat << EOF
Usage: $0
   [ -v | --vlan <vlan id> ]
   [ -s | --server-port <target vm port number> ]
   [ -c | --client-port <initiator vm port number> ]
   [ -h | --help ]
EOF
exit 1
}

ARGS=$(getopt -a -o v:t:i:h --long vlan:,server-port:,client-port:,help -- "$@")

eval set -- "${ARGS}"
while :
do
    case $1 in
        -v | --vlan)
            VLAN=$2
            shift 2
            ;;
        -s | --server-port)
            SERVER_TAP=$2
            shift 2
            ;;
        -c | --client-port)
            CLIENT_TAP=$2
            shift 2
            ;;
        -h | --help)
            usage
            ;;
        # -- means the end of the arguments; drop this, and break out of the while loop
        --)
            shift
            break
            ;;
        *) >&2 echo Unsupported option: "$1"
           usage
           ;;
      esac
done

if [[ -z "$VLAN" ]] || [[ "$VLAN" =~ [^[:digit:]] ]]; then
    echo "VLAN identifier is required"
    usage
fi

if [[ -z "$SERVER_TAP" ]] || [[ "$SERVER_TAP" =~ [^[:digit:]] ]]; then
    echo "Target tap port number is required"
    usage
fi

if [[ -z "$CLIENT_TAP" ]] || [[ "$CLIENT_TAP" =~ [^[:digit:]] ]]; then
    echo "Initiator tap port number is required"
    usage
fi

echo -e "~> iSCSI lab VLAN identifier: ${BLUE}${VLAN}${NC}"
echo -e "~> Target VM tap port number: ${BLUE}${SERVER_TAP}${NC}"
echo -e "~> Initiator VM tap port number: ${BLUE}${CLIENT_TAP}${NC}"
tput sgr0

# Switch ports configuration
for p in ${SERVER_TAP} ${CLIENT_TAP}
do
    echo "Configuring tap${p} port..."
    sudo ovs-vsctl set port tap${p} tag=${VLAN} vlan_mode=access
done

# Copy iSCSI target and initiator VMs image files
mkdir -p $HOME/vm/${LAB_NAME}
cd $HOME/vm/${LAB_NAME}/

for f in target_img initiator_img
do
    if [[ ! -f ${f}.qcow2 ]]; then
        echo "Copying ${f}.qcow2 image file..."
        cp $HOME/masters/${MASTER_IMG_NAME}.qcow2 $HOME/vm/${LAB_NAME}/${f}.qcow2
        cp $HOME/masters/${MASTER_IMG_NAME}.qcow2_OVMF_VARS.fd $HOME/vm/${LAB_NAME}/${f}.qcow2_OVMF_VARS.fd
    fi
done

# Create second disk for each VM
for f in target_vol initiator_vol
do
    if [[ ! -e ${f}.qcow2 ]]; then
        echo "Creating ${f}.qcow2 disk..."
        qemu-img create -f qcow2 \
        -o lazy_refcounts=on,extended_l2=on,compression_type=zstd \
        $HOME/vm/${LAB_NAME}/${f}.qcow2 ${SECOND_DISK_SIZE}
    fi
done

for vm in server client
do
    # Launch iSCSI target VM
    tap=${vm^^}_TAP
    echo "octogone \n sans regles"
    $HOME/vm/scripts/ovs-startup.sh ${vm}_img.qcow2 ${RAM} ${!tap} \
        -drive if=none,id=${vm}_disk,format=qcow2,media=disk,file=${vm}_vol.qcow2 \
        -device virtio-blk,drive=${vm}_disk,scsi=off,config-wce=off
done

exit 0


~> Virtual machine filename   : server_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6326
~> telnet console port number : 2726
~> MAC address                : b8:ad:ca:fe:01:aa
~> Switch port interface      : tap426, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1aa%vlan130
~> Virtual machine filename   : client_img.qcow2
~> RAM size                   : 1024MB
~> SPICE VDI port number      : 6325
~> telnet console port number : 2725
~> MAC address                : b8:ad:ca:fe:01:a9
~> Switch port interface      : tap425, access mode
~> IPv6 LL address            : fe80::baad:caff:fefe:1a9%vlan130
```

Paramétrage des interfaces : 

```
etu@vm0:~$ ip a

2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:01:a9 brd ff:ff:ff:ff:ff:ff
    inet **192.168.130.40/24** brd 192.168.130.255 scope global enp0s1
       valid_lft forever preferred_lft forever
    inet6 2001:678:3fc:82:baad:caff:fefe:1a9/64 scope global tentative dynamic mngtmpaddr proto kernel_ra 
       valid_lft 2592000sec preferred_lft 604800sec
    inet6 fe80::baad:caff:fefe:1a9/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever

etu@vm0:~$ ip a 
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether b8:ad:ca:fe:01:aa brd ff:ff:ff:ff:ff:ff
    inet **192.168.130.41/24** brd 192.168.130.255 scope global enp0s1
       valid_lft forever preferred_lft forever
    inet6 2001:678:3fc:82:baad:caff:fefe:1aa/64 scope global dynamic mngtmpaddr proto kernel_ra 
       valid_lft 2591961sec preferred_lft 604761sec
    inet6 fe80::baad:caff:fefe:1aa/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever

etu@vm0:~$ host www.stri.fr
www.stri.fr is an alias for stri.fr.
stri.fr has address 109.234.161.150
stri.fr mail is handled by 0 stri.fr.
````

Installation des paquets pour l'execution des RPC : 
```
etu@vm0:~$ sudo apt search rpcbind
[sudo] Mot de passe de etu : 
En train de trier... Fait
Recherche en texte intégral... Fait
rpcbind/testing 1.2.6-6+b1 amd64
  conversion de numéros de programmes RPC en adresses universelles
```

RPC s'exécute derrière le port 111 : 
```
etu@vm0:~$ sudo lsof -i
rpcbind 1068 _rpc    4u  IPv4  19682      0t0  TCP *:sunrpc (LISTEN)
rpcbind 1068 _rpc    5u  IPv4  16094      0t0  UDP *:sunrpc 
rpcbind 1068 _rpc    6u  IPv6  19684      0t0  TCP *:sunrpc (LISTEN)
rpcbind 1068 _rpc    7u  IPv6  16096      0t0  UDP *:sunrpc 

etu@vm0:~$ cat /etc/services | grep rpc
sunrpc          111/tcp         portmapper      # RPC 4.0 portmapper
sunrpc          111/udp         portmapper
rpc2portmap     369/tcp
rpc2portmap     369/udp                         # Coda portmapper
```

on pouvait utiliser l'option -P de lsof pour afficher le port :
```
etu@vm0:~$ sudo lsof -i -P
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
rpcbind 1068 _rpc    4u  IPv4  19682      0t0  TCP *:111 (LISTEN)
rpcbind 1068 _rpc    5u  IPv4  16094      0t0  UDP *:111 
rpcbind 1068 _rpc    6u  IPv6  19684      0t0  TCP *:111 (LISTEN)
rpcbind 1068 _rpc    7u  IPv6  16096      0t0  UDP *:111 
```

On peut lister la liste des services disponible via RPC avec rpcinfo : 
```
etu@vm0:~$ rpcinfo
   program version netid     address                service    owner
    100000    4    tcp6      ::.0.111               portmapper superuser
    100000    3    tcp6      ::.0.111               portmapper superuser
    100000    4    udp6      ::.0.111               portmapper superuser
    100000    3    udp6      ::.0.111               portmapper superuser
    100000    4    tcp       0.0.0.0.0.111          portmapper superuser
    100000    3    tcp       0.0.0.0.0.111          portmapper superuser
    100000    2    tcp       0.0.0.0.0.111          portmapper superuser
    100000    4    udp       0.0.0.0.0.111          portmapper superuser
    100000    3    udp       0.0.0.0.0.111          portmapper superuser
    100000    2    udp       0.0.0.0.0.111          portmapper superuser
    100000    4    local     /run/rpcbind.sock      portmapper superuser
    100000    3    local     /run/rpcbind.sock      portmapper superuser

etu@vm0:~$ rpcinfo -s
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser
```

Pour afficher les services disponible sur une autre machine avec RPC on ajoute l'adresse ip de la machine après l'option -s ou -m

```
etu@vm0:~$ rpcinfo -s 192.168.130.40
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser

etu@vm0:~$ rpcinfo -s 192.168.130.41 
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser
```

Analyse du trafic lors de l'execution de tcpinfo depuis une autre machine
- On installe le paquet tcpdump
- on lance tcpdump sur .41
- on effectue le rpcinfo -s depuis .40 vers .41

```
etu@vm0:~$ sudo tcpdump 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:32:57.523262 LLDP, length 213: bob
18:33:04.383155 IP 192.168.130.40.36884 > 192.168.130.41.sunrpc: Flags [S], seq 3838077573, win 64240, options [mss 1460,sackOK,TS val 3888031539 ecr 0,nop,wscale 7], length 0
18:33:04.383354 IP 192.168.130.41.sunrpc > 192.168.130.40.36884: Flags [S.], seq 3705141908, ack 3838077574, win 65160, options [mss 1460,sackOK,TS val 512088350 ecr 3888031539,nop,wscale 7], length 0
18:33:04.384654 IP 192.168.130.40.36884 > 192.168.130.41.sunrpc: Flags [.], ack 1, win 502, options [nop,nop,TS val 3888031558 ecr 512088350], length 0
18:33:04.384885 IP 192.168.130.40.36884 > 192.168.130.41.sunrpc: Flags [P.], seq 1:45, ack 1, win 502, options [nop,nop,TS val 3888031559 ecr 512088350], length 44
18:33:04.384925 IP 192.168.130.41.sunrpc > 192.168.130.40.36884: Flags [.], ack 45, win 509, options [nop,nop,TS val 512088351 ecr 3888031559], length 0
18:33:04.385866 IP 192.168.130.41.sunrpc > 192.168.130.40.36884: Flags [P.], seq 1:689, ack 45, win 509, options [nop,nop,TS val 512088352 ecr 3888031559], length 688
18:33:04.386790 IP 192.168.130.40.36884 > 192.168.130.41.sunrpc: Flags [.], ack 689, win 501, options [nop,nop,TS val 3888031561 ecr 512088352], length 0
18:33:04.388490 IP 192.168.130.40.36884 > 192.168.130.41.sunrpc: Flags [F.], seq 45, ack 689, win 501, options [nop,nop,TS val 3888031562 ecr 512088352], length 0
18:33:04.388637 IP 192.168.130.41.sunrpc > 192.168.130.40.36884: Flags [F.], seq 689, ack 46, win 509, options [nop,nop,TS val 512088355 ecr 3888031562], length 0
18:33:04.389968 IP6 2001:678:3fc:82:baad:caff:fefe:1aa.38547 > 2001:678:3fc:3::2.domain: 9757+ PTR? 41.130.168.192.in-addr.arpa. (45)
18:33:04.390913 IP 192.168.130.40.36884 > 192.168.130.41.sunrpc: Flags [.], ack 690, win 501, options [nop,nop,TS val 3888031565 ecr 512088355], length 0
````

On voit que .40 fait un appel sur le port 111 qui est traduit par sunrpc en passant par TCP/IP, avec tshark on obtient une représentation légérement plus detaillé avec l'apparition du processus portmap

```
etu@vm0:~$ sudo tshark -i enp0s1
Running as user "root" and group "root". This could be dangerous.
Capturing on 'enp0s1'
 ** (tshark:1746) 18:41:51.186067 [Main MESSAGE] -- Capture started.
 ** (tshark:1746) 18:41:51.186204 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s18LHFA2.pcapng"
    1 0.000000000 192.168.130.40 → 192.168.130.41 TCP 74 51050 → 111 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM TSval=3888565337 TSecr=0 WS=128
    2 0.000142164 192.168.130.41 → 192.168.130.40 TCP 74 111 → 51050 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM TSval=512622177 TSecr=3888565337 WS=128
    3 0.001684839 192.168.130.40 → 192.168.130.41 TCP 66 51050 → 111 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3888565359 TSecr=512622177
    4 0.002299436 192.168.130.40 → 192.168.130.41 Portmap 110 V3 DUMP Call
    5 0.002337628 192.168.130.41 → 192.168.130.40 TCP 66 111 → 51050 [ACK] Seq=1 Ack=45 Win=65152 Len=0 TSval=512622179 TSecr=3888565360
    6 0.002796338 192.168.130.41 → 192.168.130.40 Portmap 754 V3 DUMP Reply (Call In 4)
    7 0.003685025 192.168.130.40 → 192.168.130.41 TCP 66 51050 → 111 [ACK] Seq=45 Ack=689 Win=64128 Len=0 TSval=3888565361 TSecr=512622179
    8 0.005058158 192.168.130.40 → 192.168.130.41 TCP 66 51050 → 111 [FIN, ACK] Seq=45 Ack=689 Win=64128 Len=0 TSval=3888565363 TSecr=512622179
    9 0.005356682 192.168.130.41 → 192.168.130.40 TCP 66 111 → 51050 [FIN, ACK] Seq=689 Ack=46 Win=65152 Len=0 TSval=512622182 TSecr=3888565363
   10 0.006732423 192.168.130.40 → 192.168.130.41 TCP 66 51050 → 111 [ACK] Seq=46 Ack=690 Win=64128 Len=0 TSval=3888565364 TSecr=512622182

etu@vm0:~$ sudo tshark -i enp0s1
Running as user "root" and group "root". This could be dangerous.
Capturing on 'enp0s1'
 ** (tshark:1782) 18:44:43.358880 [Main MESSAGE] -- Capture started.
 ** (tshark:1782) 18:44:43.359117 [Main MESSAGE] -- File: "/tmp/wireshark_enp0s1M0EAA2.pcapng"
    1 0.000000000 2001:678:3fc:82:baad:caff:fefe:1a9 → 2001:678:3fc:82:baad:caff:fefe:1aa TCP 94 56946 → 111 [SYN] Seq=0 Win=64800 Len=0 MSS=1440 SACK_PERM TSval=2397164737 TSecr=0 WS=128
    2 0.000090012 2001:678:3fc:82:baad:caff:fefe:1aa → 2001:678:3fc:82:baad:caff:fefe:1a9 TCP 94 111 → 56946 [SYN, ACK] Seq=0 Ack=1 Win=64260 Len=0 MSS=1440 SACK_PERM TSval=1835339782 TSecr=2397164737 WS=128
    3 0.001116023 2001:678:3fc:82:baad:caff:fefe:1a9 → 2001:678:3fc:82:baad:caff:fefe:1aa TCP 86 56946 → 111 [ACK] Seq=1 Ack=1 Win=64896 Len=0 TSval=2397164757 TSecr=1835339782
    4 0.001564487 2001:678:3fc:82:baad:caff:fefe:1a9 → 2001:678:3fc:82:baad:caff:fefe:1aa Portmap 130 V3 DUMP Call
    5 0.001594189 2001:678:3fc:82:baad:caff:fefe:1aa → 2001:678:3fc:82:baad:caff:fefe:1a9 TCP 86 111 → 56946 [ACK] Seq=1 Ack=45 Win=64256 Len=0 TSval=1835339783 TSecr=2397164758
    6 0.002015910 2001:678:3fc:82:baad:caff:fefe:1aa → 2001:678:3fc:82:baad:caff:fefe:1a9 Portmap 774 V3 DUMP Reply (Call In 4)
    7 0.002869780 2001:678:3fc:82:baad:caff:fefe:1a9 → 2001:678:3fc:82:baad:caff:fefe:1aa TCP 86 56946 → 111 [ACK] Seq=45 Ack=689 Win=64256 Len=0 TSval=2397164759 TSecr=1835339783
    8 0.005012042 2001:678:3fc:82:baad:caff:fefe:1a9 → 2001:678:3fc:82:baad:caff:fefe:1aa TCP 86 56946 → 111 [FIN, ACK] Seq=45 Ack=689 Win=64256 Len=0 TSval=2397164761 TSecr=1835339783
    9 0.005295076 2001:678:3fc:82:baad:caff:fefe:1aa → 2001:678:3fc:82:baad:caff:fefe:1a9 TCP 86 111 → 56946 [FIN, ACK] Seq=689 Ack=46 Win=64256 Len=0 TSval=1835339787 TSecr=2397164761
   10 0.005899443 2001:678:3fc:82:baad:caff:fefe:1a9 → 2001:678:3fc:82:baad:caff:fefe:1aa TCP 86 56946 → 111 [ACK] Seq=46 Ack=690 Win=64256 Len=0 TSval=2397164762 TSecr=1835339787
```

## Exportation depuis le serveur un serveur 

Sur la machine serveur installé le service de serveur nfs : 
```
etu@vm0:~$ sudo apt install nfs-kernel-server

etu@vm0:~$ sudo cat /etc/exports 
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/home/exports      192.168.130.40/24(rw,sync,no_subtree_check)
/home/exports/home 192.168.130.40/24(rw,sync,no_subtree_check)

etu@vm0:~$ sudo systemctl restart nfs-kernel-server
etu@vm0:~$ systemctl status nfs-kernel-server
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; preset: enabled)
     Active: active (exited) since Wed 2023-08-30 19:28:00 CEST; 3s ago
    Process: 3152 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 3153 ExecStart=/usr/sbin/rpc.nfsd (code=exited, status=0/SUCCESS)
   Main PID: 3153 (code=exited, status=0/SUCCESS)
        CPU: 7ms

août 30 19:28:00 vm0 systemd[1]: Starting nfs-server.service - NFS server and services...
août 30 19:28:00 vm0 systemd[1]: Finished nfs-server.service - NFS server and services.
etu@vm0:~$ sudo exportfs 
/home/exports   192.168.130.40/24
/home/exports/home
                192.168.130.40/24
```
```
etu@vm0:~$ sudo adduser --home /ahome/etu-nfs etu-nfs

etu@vm0:~$ id etu-nfs
uid=1001(etu-nfs) gid=1001(etu-nfs) groupes=1001(etu-nfs),100(users)
```

## Montage manuel depuis le client NFS

Depuis le client on peut observer les nouveaux services rpc disponibles sur le  serveur 

```
etu@vm0:~$ rpcinfo -s 192.168.130.41
   program version(s) netid(s)                         service     owner
    100000  2,3,4     local,udp,tcp,udp6,tcp6          portmapper  superuser
    100024  1         tcp6,udp6,tcp,udp                status      105
    100005  3,2,1     tcp6,udp6,tcp,udp                mountd      superuser
    100003  4,3       tcp6,tcp                         nfs         superuser
    100227  3         tcp6,tcp                         nfs_acl     superuser
    100021  4,3,1     tcp6,udp6,tcp,udp                nlockmgr    superuser

etu@vm0:~$ sudo umount ahome
/home/exports      192.168.130.40/24

etu@vm0:~$ ls -la
total 44
drwx------ 3 etu  etu  4096 30 août  17:37 .
drwxr-xr-x 3 root root 4096 26 mars  09:38 ..
-rw-r--r-- 1 etu  etu    20 26 mars  10:54 .bash_aliases
-rw------- 1 etu  etu  9136 30 août  17:22 .bash_history
-rw-r--r-- 1 etu  etu   220 26 mars  09:38 .bash_logout
-rw-r--r-- 1 etu  etu  3526 26 mars  10:52 .bashrc
-rw------- 1 etu  etu    34 27 mars  08:29 .lesshst
drwxr-xr-x 3 etu  etu  4096 30 août  17:37 .local
-rw-r--r-- 1 etu  etu   807 26 mars  09:38 .profile
-rw-r--r-- 1 etu  etu     0 26 mars  10:47 .sudo_as_admin_successful
etu@vm0:~$ mkdir ahome
etu@vm0:~$ sudo mount 192.168.130.41:/home/ /ahome

etu@vm0:~$ mount | grep nfs
192.168.130.41:/home on /home/etu/ahome type nfs4 (rw,relatime,vers=4.2,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.130.40,local_lock=none,addr=192.168.130.41)
etu@vm0:~$ 
```

On effectue la commande ```ls -laH``` pour lire le contenu du dossier ahome/home qui est monté sur le client NFS et on observe les paquets reçus sur le serveur

```
    1 0.000000000 192.168.130.40 → 192.168.130.41 NFS 278 V4 Call GETATTR FH: 0x93a73fea
    2 0.004473337 192.168.130.41 → 192.168.130.40 NFS 310 V4 Reply (Call In 1) GETATTR
    3 0.005794033 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=213 Ack=245 Win=501 Len=0 TSval=4048018524 TSecr=672075359
    4 5.974945103 2e:ae:37:1d:10:9a → LLDP_Multicast LLDP 227 MA/5c:6f:69:70:73:50 MA/2e:ae:37:1d:10:9a 120 SysN=bob SysD=Debian GNU/Linux trixie/sid Linux 6.4.0-3-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.4.11-1 (2023-08-17) x86_64 
    5 6.909780838 192.168.130.40 → 192.168.130.41 NFS 278 V4 Call GETATTR FH: 0x93a73fea
    6 6.910294993 192.168.130.41 → 192.168.130.40 NFS 310 V4 Reply (Call In 5) GETATTR
    7 6.911273923 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=425 Ack=489 Win=501 Len=0 TSval=4048025429 TSecr=672082265
    8 6.911694174 192.168.130.40 → 192.168.130.41 NFS 286 V4 Call ACCESS FH: 0x93a73fea, [Check: RD LU MD XT DL XAR XAW XAL]
    9 6.911899773 192.168.130.41 → 192.168.130.40 NFS 238 V4 Reply (Call In 8) ACCESS, [Access Denied: MD XT DL XAW], [Allowed: RD LU XAR XAL]
   10 6.913122704 192.168.130.40 → 192.168.130.41 NFS 302 V4 Call READDIR FH: 0x93a73fea
   11 6.944218557 192.168.130.41 → 192.168.130.40 NFS 386 V4 Reply (Call In 10) READDIR
   12 6.987664354 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=881 Ack=981 Win=501 Len=0 TSval=4048025506 TSecr=672082299
   13 14.739993086 192.168.130.40 → 192.168.130.41 NFS 278 V4 Call GETATTR FH: 0x93a73fea
   14 14.740408859 192.168.130.41 → 192.168.130.40 NFS 310 V4 Reply (Call In 13) GETATTR
   15 14.741305885 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=1093 Ack=1225 Win=501 Len=0 TSval=4048033259 TSecr=672090095
   16 16.488060592 192.168.130.40 → 192.168.130.41 NFS 278 V4 Call GETATTR FH: 0x93a73fea
   17 16.488439663 192.168.130.41 → 192.168.130.40 NFS 310 V4 Reply (Call In 16) GETATTR
   18 16.489386490 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=1305 Ack=1469 Win=501 Len=0 TSval=4048035007 TSecr=672091843
   19 16.973382875 192.168.130.40 → 192.168.130.41 NFS 298 V4 Call LOOKUP DH: 0x93a73fea/exports
   20 16.973781430 192.168.130.41 → 192.168.130.40 NFS 358 V4 Reply (Call In 19) LOOKUP
   21 16.974840692 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=1537 Ack=1761 Win=501 Len=0 TSval=4048035493 TSecr=672092328
   22 16.975831846 192.168.130.40 → 192.168.130.41 NFS 282 V4 Call GETATTR FH: 0x2311fbdc
   23 16.980877578 192.168.130.41 → 192.168.130.40 NFS 242 V4 Reply (Call In 22) GETATTR
   24 16.981931077 192.168.130.40 → 192.168.130.41 NFS 282 V4 Call GETATTR FH: 0x2311fbdc
   25 16.982089681 192.168.130.41 → 192.168.130.40 NFS 238 V4 Reply (Call In 24) GETATTR
   26 16.983473937 192.168.130.40 → 192.168.130.41 NFS 274 V4 Call GETATTR FH: 0x2311fbdc
   27 16.983808935 192.168.130.41 → 192.168.130.40 NFS 186 V4 Reply (Call In 26) GETATTR
   28 16.985468704 192.168.130.40 → 192.168.130.41 NFS 282 V4 Call GETATTR FH: 0x2311fbdc
   29 16.985663662 192.168.130.41 → 192.168.130.40 NFS 242 V4 Reply (Call In 28) GETATTR
   30 16.986626616 192.168.130.40 → 192.168.130.41 NFS 278 V4 Call GETATTR FH: 0x2311fbdc
   31 16.986921299 192.168.130.41 → 192.168.130.40 NFS 310 V4 Reply (Call In 30) GETATTR
   32 16.988953281 192.168.130.40 → 192.168.130.41 NFS 278 V4 Call GETATTR FH: 0x2311fbdc
   33 16.989259463 192.168.130.41 → 192.168.130.40 NFS 310 V4 Reply (Call In 32) GETATTR
   34 16.990254893 192.168.130.40 → 192.168.130.41 NFS 286 V4 Call ACCESS FH: 0x2311fbdc, [Check: RD LU MD XT DL XAR XAW XAL]
   35 16.990487206 192.168.130.41 → 192.168.130.40 NFS 238 V4 Reply (Call In 34) ACCESS, [Access Denied: MD XT DL XAW], [Allowed: RD LU XAR XAL]
   36 16.991326172 192.168.130.40 → 192.168.130.41 NFS 302 V4 Call READDIR FH: 0x2311fbdc
   37 16.991630353 192.168.130.41 → 192.168.130.40 NFS 390 V4 Reply (Call In 36) READDIR
   38 17.035730133 192.168.130.40 → 192.168.130.41 TCP 66 719 → 2049 [ACK] Seq=3273 Ack=3389 Win=501 Len=0 TSval=4048035554 TSecr=672092346
```
On observe que les paquets NFS effectuent la commande READDIR

Demontage depuis le client 

```
etu@vm0:~$ ls ahome/exports/home/
montoto.txt
etu@vm0:~$ sudo um
umask        umount       umount.nfs   umount.nfs4  
etu@vm0:~$ sudo umount 
/                           /boot/efi                   /home/etu/ahome             /run/user/1000              /sys/kernel/config 
./ahome                     /dev                        /home/etu/ahome/exports     /sys                        /sys/kernel/debug 
~/ahome                     /dev/hugepages              /proc                       /sys/firmware/efi/efivars   /sys/kernel/security 
ahome                       /dev/mqueue                 /proc/sys/fs/binfmt_misc    /sys/fs/bpf                 /sys/kernel/tracing 
./ahome/exports             /dev/pts                    /run                        /sys/fs/cgroup              
~/ahome/exports             /dev/shm                    /run/lock                   /sys/fs/fuse/connections    
ahome/exports               /home/etu                   /run/rpc_pipefs             /sys/fs/pstore              
etu@vm0:~$ sudo umount ahome
[sudo] Mot de passe de etu : 

etu@vm0:~$ ls ahome/
```
## Automatiser le montage sur le client

```
etu@vm0:~$ sudo apt install autofs
Lecture des listes de paquets... Fait
Construction de l'arbre des dépendances... Fait
Lecture des informations d'état... Fait      
autofs est déjà la version la plus récente (5.1.8-3.1).
0 mis à jour, 0 nouvellement installés, 0 à enlever et 0 non mis à jour.
etu@vm0:~$ 

etu@vm0:~$ sudo adduser --no-create-home --home /ahome/etu-nfs etu-nfs

etu@vm0:~$ id etu-nfs 
uid=1001(etu-nfs) gid=1001(etu-nfs) groupes=1001(etu-nfs),100(users)

etu@vm0:~$ echo "/ahome    /etc/auto.home" | \sudo tee -a /etc/auto.master.d/ahome.autofs
etu@vm0:~$ echo "* -fstype=nfs4 192.168.130.41:/home/exports/home/&" | \sudo tee -a /etc/auto.home
etu@vm0:~$ sudo systemctl restart autofs
etu@vm0:~$ systemctl status autofs
autofs.service - Automounts filesystems on demand
     Loaded: loaded (/lib/systemd/system/autofs.service; enabled; preset: enabled)
     Active: active (running) since Fri 2023-09-01 16:01:22 CEST; 14s ago
       Docs: man:autofs(8)
    Process: 8899 ExecStart=/usr/sbin/automount $OPTIONS --pid-file /var/run/autofs.pid (code=exited>
   Main PID: 8901 (automount)
      Tasks: 4 (limit: 1082)
     Memory: 3.0M
        CPU: 30ms
     CGroup: /system.slice/autofs.service
             └─8901 /usr/sbin/automount --pid-file /var/run/autofs.pid

sept. 01 16:01:22 vm0 systemd[1]: Starting autofs.service - Automounts filesystems on demand...
sept. 01 16:01:22 vm0 systemd[1]: Started autofs.service - Automounts filesystems on demand.

etu@vm0:~$ su - etu-nfs
Mot de passe : 
etu-nfs@vm0:~$ pwd
/ahome/etu-nfs

etu-nfs@vm0:~$ mount | grep nfs
192.168.130.41:/home/exports/home on /ahome/etu-nfs type nfs4 (rw,relatime,vers=4.2,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.130.40,local_lock=none,addr=192.168.130.41)

```

## Gestion des droits sur le dossier home de etu-nfs 

```
etu@vm0:~$ sudo touch /ahome/etu-nfs/fichierRoot.txt
etu@vm0:~$ sudo touch /ahome/etu-nfs/fichierEtu.txt
etu@vm0:~$ sudo chown etu-nfs /ahome/etu-nfs/fichierEtu.txt

etu@vm0:~$ sudo ls -la /ahome/etu-nfs/
total 24
drwx------ 2 etu-nfs etu-nfs 4096  1 sept. 16:25 .
drwxr-xr-x 3 root    root    4096  1 sept. 14:46 ..
-rw------- 1 etu-nfs etu-nfs   97  1 sept. 16:16 .bash_history
-rw-r--r-- 1 etu-nfs etu-nfs  220  1 sept. 14:46 .bash_logout
-rw-r--r-- 1 etu-nfs etu-nfs 3526  1 sept. 14:46 .bashrc
-rw-r--r-- 1 etu-nfs root       0  1 sept. 16:25 fichierEtu.txt
-rw-r--r-- 1 root    root       0  1 sept. 16:24 fichierRoot.txt
-rw-r--r-- 1 etu-nfs etu-nfs  807  1 sept. 14:46 .profile
```

sur la machine client on test :

```
etu@vm0:~$ su - etu-nfs
Mot de passe : 
etu-nfs@vm0:~$ ls -la
total 20
drwx------ 2 etu-nfs etu-nfs 4096  1 sept. 16:43 .
drwxr-xr-x 3 root    root       0  1 sept. 16:49 ..
-rw------- 1 etu-nfs etu-nfs   46  1 sept. 16:50 .bash_history
-rw-r--r-- 1 etu-nfs etu-nfs  220  1 sept. 16:42 .bash_logout
-rw-r--r-- 1 etu-nfs etu-nfs 3526  1 sept. 16:42 .bashrc
-rw-r--r-- 1 etu-nfs root       0  1 sept. 16:43 fichierEtu
-rw-r--r-- 1 root    root       0  1 sept. 16:43 fichierRoot
-rw-r--r-- 1 etu-nfs etu-nfs  807  1 sept. 16:42 .profile
```

On voit la présence des deux fichiers.
On test la création sur la machine serveur, l'"criture dans le fichier sur le client :
```
etu-nfs@SERVEUR:~$ touch monFich.txt
-----------------------------------
etu-nfs@CLIENT:~$ echo 'Je peux ecrire dans ce fichier' > monFich.txt 
-----------------------------------
etu-nfs@SERVEUR:~$ cat monFich.txt 
Je peux ecrire dans ce fichier
```

et dans l'autre sens 

```
etu-nfs@CLIENT:~$ mkdir dirEtu
etu-nfs@CLIENT:~$ echo "fichier dans dossier" > dirEtu/fichierDir

etu-nfs@SERVEUR~$ cat dirEtu/fichierDir 
fichier dans dossier
etu-nfs@SERVEUR:~$ ls -la dirEtu/
total 12
drwxr-xr-x 2 etu-nfs etu-nfs 4096  1 sept. 18:35 .
drwx------ 3 etu-nfs etu-nfs 4096  1 sept. 18:35 ..
-rw-r--r-- 1 etu-nfs etu-nfs   21  1 sept. 18:35 fichierDir
```


