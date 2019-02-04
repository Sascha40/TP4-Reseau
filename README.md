# TP4-Reseau

Cette fois ci j'ai troqué mon mac contre un bon vieux windows, bon c'est toujours pas linux mais sa dépanne.

## Préparation d'une VM "patron" :

J'ai crée un VM sous VirtualBox avec les conditions que tu nous as recommandé, cad:
* 512 Mo RAM
* 1 coeur pour le CPU
* Réseau
  * une carte NAT
* Stockage
  * disque de 8Go

**Configuration de la VMs:**
* J'ai desactiver Setenforce, puis mis à jour les dépots avec `yum -update -y`
* Il m'a répondu `Complete!`
* Installation des dépots additionnels
* Il me répond `Complete!`
* Même chose avec les paquets réseau dont on se sert souvent
* Réponse: `Complete!`
* **Ensuite:**
* J'ai mis le `ONBOOT` à `NO`

## Mise en place du lab :

### 1) Création des réseaux:
J'ai crée les réseaux host-only sur mon ordi.

### 2) Création des VMs  

***CHECKLIST complétée***

* Définition des IP statiques :  
La commande qui suit : `sudo nano etc/sysconfig/network-scripts/ifcfg-<NOM_CARTE_RESEAU>` permet de définir/modifier une ip sur chaque VM

Ci joint une copie du paramètrage:

  + *client1.tp4* avec **10.1.0.10/24** 
  
        ```
        NAME=enp0s3
        DEVICE=enp0s3
        BOOTPROTO=static
        ONBOOT=yes
        IPADDR=10.1.0.10
        NETMASK=255.255.255.0
        ```  
    +  On fait pareil pour le *serveur1.tp4* avec **10.2.0.10/24** 
    
        ```
        NAME=enp0s3
        DEVICE=enp0s3
        BOOTPROTO=static
        ONBOOT=yes
        IPADDR=10.2.0.10
        NETMASK=255.255.255.0
        ```  
    
    + Passons au routeur, le *routeur1.tp4*, on modifie les 2 cartes réseaux.  
   Sois **10.1.0.254/24** et **10.2.0.254/24**  

**Connexion SSH**

Pour se connecter via ssh à chaque VM depuis l'ordinateur hôte on tape la commande : `ssh root@<IP>`  
Désormais mes 3 VMs client, serveur, routeur sont connectées entre elles.  

+ Nom de domaine  
Pour changer le nom de domaine, je tape sur mes terminals(aux?) la commande:
`echo '<NOM_DOMAINE>' | sudo tee /etc/hostname`  


+ Fichier /etc/hosts

Pour modifier les hosts on tape la commande: `sudo nano /etc/hosts`
Ici l'exemple est sur le terminal **client**, ce qui explique que les 3 dernieres lignes varient.
En effet si on est sur le routeur on ajoute client et serveur, et si on est sur serveur on ajoute client et routeur.

Ce qui nous donne: 
    ```
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    10.2.0.10   serveur1.tp4
    10.1.0.254  routeur1.net1.tp4
    10.2.0.254  routeur1.net2.tp4
    ```


+ Le famous Ping  

    **client1 ping router1.tp4 sur l'IP 10.1.0.254  
    On tape la commande `ping routeur1.net1.tp4` qui nous donne:
    
        ```
        PING routeur1.net1.tp4 (10.1.0.254) 56(84) bytes of data.
        64 bytes from routeur1.net1.tp4 (10.1.0.254): icmp_seq=1 ttl=64 time=0.459 ms
        64 bytes from routeur1.net1.tp4 (10.1.0.254): icmp_seq=2 ttl=64 time=0.664 ms
        ^C
        --- routeur1.net1.tp4 ping statistics ---
        2 packets transmitted, 2 received, 0% packet loss, time 1001ms
        rtt min/avg/max/mdev = 0.459/0.561/0.664/0.105 ms
        ```  
     **server1 ping router1.tp4 sur l'IP 10.2.0.254 
     On tape la commande `ping routeur1.net2.tp4` qui nous donne:
     
        ```
        PING routeur1.net2.tp4 (10.2.0.254) 56(84) bytes of data.
        64 bytes from routeur1.net2.tp4 (10.2.0.254): icmp_seq=1 ttl=64 time=0.382 ms
        64 bytes from routeur1.net2.tp4 (10.2.0.254): icmp_seq=2 ttl=64 time=0.363 ms
        ^C
        --- routeur1.net2.tp4 ping statistics ---
        2 packets transmitted, 2 received, 0% packet loss, time 1003ms
        rtt min/avg/max/mdev = 0.363/0.372/0.382/0.021 ms
        ```  

###  3) Mise en place du routage statique  

* sur `routeur1` :  

    * activer l'IPv4 Forwarding (= transformer la machine en routeur)  
        ``` 
        [root@routeur1 ~]# sudo sysctl -w net.ipv4.conf.all.forwarding=1
        net.ipv4.conf.all.forwarding = 1
        ```
    * désactiver le firewall (pour éviter certaines actions non voulues)
   
   Je vais pas te mentir ça n'a pas marché au départ, et part l'opération du St Esprit ça à enfin fonctionné.. au bout   d'une heure, c'est scandaleux.
   
        ```
        [root@routeur1 ~]# sudo systemctl disable firewalld
        Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
        Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
        ```  
    * vérifier qu'il a déjà des routes pour aller vers net1 et net2
        ```
        [root@routeur1 ~]# ip route show
        10.1.0.0/24 dev enp0s3 proto kernel scope link src 10.1.0.254 metric 100
        10.2.0.0/24 dev enp0s8 proto kernel scope link src 10.2.0.254 metric 101
        ```  
* sur `client1` :  
    * faire en sorte que la machine ait une route vers net1 et net2  
    * Modifier le fichier `route-enp0s3`dans `/etc/sysconfig/network-scripts`  
    * Ajouter les routes vers `net1` et `net2`  
    
* sur `serveur1` :  
    * faire en sorte que la machine ait une route vers net1 et net2  
    * Modifier le fichier `route-enp0s3`dans `/etc/sysconfig/network-scripts`  
    * Ajouter les routes vers `net1` et `net2`  

* Test  
    Client1 doit pouvoir ping server1
    
    * Ping vers `serveur1.tp4` :  
    
        ```
        [root@client1 ~]# ping serveur1.tp4
        PING serveur1.tp4 (10.2.0.10) 56(84) bytes of data.
        64 bytes from serveur1.tp4 (10.2.0.10): icmp_seq=1 ttl=63 time=0.963 ms
        64 bytes from serveur1.tp4 (10.2.0.10): icmp_seq=2 ttl=63 time=1.44 ms
        ```  
 
    Server1 doit pouvoir ping client1
 
    + Ping vers `client1.tp4` :
    
        ```  
        [root@serveur1 ~]# ping client1.tp4
        PING client1.tp4 (10.1.0.10) 56(84) bytes of data.
        64 bytes from client1.tp4 (10.1.0.10): icmp_seq=1 ttl=63 time=0.578 ms
        64 bytes from client1.tp4 (10.1.0.10): icmp_seq=2 ttl=63 time=1.43 ms
        ```  
        
        
## II. Spéléologie réseau  

### 1. ARP 

#### A. Manipulation 1  

1. Vider la table ARP (à faire sur toutes les machines) : 
Sur client1 on fait:  `[root@client1 ~]# sudo ip neigh flush all`  

2. Sur `client1`  
    + table ARP : `10.1.0.1 dev enp0s3 lladdr 0a:00:27:00:00:0c DELAY`  
    `10.1.0.1` : adresse réseau dans lequel ou nous sommes, ici `net1`  
    `enp0s3` : nom de la carte réseau du `client1`  
    `0a:00:27:00:00:0c` : adresse MAC de la carte réseau  
    
3. Sur `serveur1`  
    + table ARP : `10.2.0.1 dev enp0s3 lladdr 0a:00:27:00:00:12 REACHABLE`  
    `10.2.0.1` : adresse réseau dans lequel ou nous sommes, ici `net2`  
    `enp0s3` : nom de la carte réseau du `serveur1`  
    `0a:00:27:00:00:12` : adresse MAC de la carte réseau  
    
4. Sur `client1`  
    + ping `serveur1`  
    + table ARP :  
    
        ```
        10.1.0.1 dev enp0s3 lladdr 0a:00:27:00:00:0c REACHABLE
        10.1.0.254 dev enp0s3 lladdr 08:00:27:cb:84:f7 REACHABLE
        ```
 L'IP de notre `routeur1.net1`, son nom de carte réseau et sa MAC sont affichés.  
 
5) Sur `serveur1`  
    + ping `client1`  
    + table ARP : 
    
        ```
        10.2.0.1 dev enp0s3 lladdr 0a:00:27:00:00:12 REACHABLE
        10.2.0.254 dev enp0s3 lladdr 08:00:27:67:6e:d9 STALE
        ```
    L'IP de notre `routeur1.net2`, son nom de carte réseau et sa MAC sont affichés.   

#### B. Manipulation 2  

1. Vider la table ARP (à faire sur toutes les machines) :  
Sur client1 on fait: `[root@client1 ~]# sudo ip neigh flush all`  

2. Sur `routeur1`  
    + table ARP : `10.1.0.1 dev enp0s3 lladdr 0a:00:27:00:00:0c DELAY`  
    `10.1.0.1` : adresse du réseau dans lequel on se trouve, `net1`  
    `enp0s3` : nom de la carte réseau du `routeur1`  
    `0a:00:27:00:00:0c` : adresse MAC de la carte réseau 
    
3. `client1` ping `serveur1`  

4. Sur `routeur1`  
    + table ARP :  
        ```
        10.1.0.10 dev enp0s3 lladdr 08:00:27:1b:64:47 REACHABLE
        10.1.0.1 dev enp0s3 lladdr 0a:00:27:00:00:0c REACHABLE
        10.2.0.10 dev enp0s8 lladdr 08:00:27:55:03:c7 REACHABLE
        ```  
    Plusieurs changements visibles : la première ligne et la dernière.  
    Cela montre que pour *ping* notre **serveur** depuis notre **client**, le *ping* est passé par notre **routeur**, qui a ensuite transmit le message au **serveur**.  

### C. Manipulation 3  

1) Même chose pour vider la table ARP : `[root@client1 ~]# sudo ip neigh flush all` 

2) Sur `PC hôte`  
    + table ARP :  
        ```
        C:\WINDOWS\system32>arp -a

        Interface : 192.168.56.1 --- 0x3
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        230.0.0.1             01-00-5e-00-00-01     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique

        Interface : 10.1.0.1 --- 0xc
        Adresse Internet      Adresse physique      Type
        10.1.0.10             08-00-27-1b-64-47     dynamique
        224.0.0.22            01-00-5e-00-00-16     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique

        Interface : 10.2.0.1 --- 0x12
        Adresse Internet      Adresse physique      Type
        10.2.0.10             08-00-27-55-03-c7     dynamique
        224.0.0.22            01-00-5e-00-00-16     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique
        ```  
    + Vider la table ARP : `arp -d`  
    + Afficher de nouveau la table ARP : 
        ```
        C:\WINDOWS\system32>arp -a

        Interface : 192.168.56.1 --- 0x3
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        230.0.0.1             01-00-5e-00-00-01     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique

        Interface : 10.1.0.1 --- 0xc
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique

        Interface : 10.2.0.1 --- 0x12
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique
        ```  
    + Attendre un peu (ou beaucoup ... j'aurais du rester sur mac mdr) et réafficher la table ARP :
    
        ```
        C:\WINDOWS\system32>arp -a

        Interface : 192.168.56.1 --- 0x3
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        230.0.0.1             01-00-5e-00-00-01     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique

        Interface : 10.1.0.1 --- 0xc
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique

        Interface : 10.2.0.1 --- 0x12
        Adresse Internet      Adresse physique      Type
        224.0.0.22            01-00-5e-00-00-16     statique
        239.255.255.250       01-00-5e-7f-ff-fa     statique
        ```  
  
 Si on attend un peu avant de réafficher la table ARP,
 `255.255.255.255       ff-ff-ff-ff-ff-ff     statique` apparait.

#### D. Manip 4 

1) Vider la table ARP (à faire sur toutes les machines) :  
Comme d'hab on tape: `[root@client1 ~]# sudo ip neigh flush all`  

2) Sur `client1`   
    + table ARP : `10.1.0.1 dev enp0s3 lladdr 0a:00:27:00:00:0c DELAY`  

### 2. Wireshark  

#### A. Interception d'ARP et `ping`  
1) Sur `routeur1`  
    + lancer Wireshark pour enregistrer le trafic qui passer par l'interface choisie et enregistrer le trafic dans un fichier ping.pcap :  
    `sudo tcpdump -i enp0s9 -w ping.pcap` 
    
2) Sur `client1`  
    + Vider la table ARP : `[root@client1 ~]# sudo ip neigh flush all`  
    + Envoyer 4 pings à `server1`  
3) Sur `router1`  
    + quitter la capture (CTRL + C)  
    + vérifier la présence du fichier **ping.pcap** avec un `ls`  
        ```
        [root@routeur1 ~]# ls
        anaconda-ks.cfg  ping.pcap
        ```  
Sur l'hôte:  

```
C:\Users\saschasalles>scp root@10.1.0.254:/root/ping.pcap C:/Users/saschasalles/Desktop
root@10.1.0.254's password:
ping.pcap                                                                             100%   16KB  17.5KB/s   00:00
```

#### B. Interception d'une communication netcat  

* Fichier `netcat-ok.pcap` :
    * le client envoie `SYN`  
    * le serveur répond `SYN,ACK`  
    * le client répond `ACK`  
 
 J'ai du mal à finir le TP pour la partie C, je ne parvient pas a lancer le serveur, deplus il me calle une erreur avec l'install de nginx
