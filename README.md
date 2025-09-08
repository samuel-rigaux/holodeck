# Notice d’installation et d’utilisation du projet Holodeck

## Sommaire
- [Configuration des VMs](#configuration-des-vms)
- [DHCP](#dhcp)
- [DNS](#dns)
- 


## Configuration des VMs
Tout d'abord on crée et on configure les VMs sur VMWare de cette manière : 

 - Pour la VM Serveur : Debian 12 de base sans interface graphique – 2 Go
   RAM – 2vpcu – Disque 32 Go. Avec 2 cartes réseaux (une WAN et une
   LAN)
   
 - Pour la VM Cliente : une Debian avec GUI – 2 Go RAM – 2vcpu – Disque
   16 Go et connectée sur le «LAN» de la VM serveur et un navigateur
   web.
   
*Pour la suite du tutoriel, j'ai tout effectué en root pour simplifier les commandes*

# DHCP
- [Installation du serveur DHCP](#installation-du-serveur-dhcp)
- [Configuration du DHCP](#configuration-du-dhcp)
- [Affectation de l’interface réseau](#affectation-de-linterface-réseau)
- [Lancement et vérification du service](#lancement-et-vérification-du-service)
- [Dépannage](#dépannage)
- [Ressources](#ressources)
---
### Installation du serveur DHCP

    apt update  
    apt install isc-dhcp-server -y

### Configuration du DHCP

Édite le fichier principal de configuration :

    nano /etc/dhcp/dhcpd.conf


Exemple de configuration pour le réseau `172.16.0.0/16` :

    option domain-name "starfleet.lan";  
    option domain-name-servers 172.16.0.100; # IP du serveur DNS (modifier si besoin)
    
    default-lease-time 600;  
    max-lease-time 7200;  
    authoritative;
    
    subnet 172.17.0.100 netmask 255.255.0.0 {  
    range 172.17.0.5 172.17.0.10;  
    option routers 172.17.0.2; # Passerelle réseau  
    option subnet-mask 255.255.0.0;  
    option broadcast-address 172.17.255.255;  
    option domain-name-servers 172.16.0.100;  
    option domain-name "starfleet.lan";  
    }

**Aide** :
- le `range` (plage d’adresses attribuables)
- `option routers` (gateway de ton LAN)
- `option domain-name-servers` (IP de ton DNS)
- `domain-name` (nom de ton domaine interne)

---

### Affectation de l’interface réseau

Définir sur quelle interface DHCP doit écouter (exemple : `ens34` pour le LAN) :

    nano /etc/default/isc-dhcp-server


Renseigne la ligne suivante (sans # devant) :

    INTERFACESv4="ens34"
    
> Remplace `ens34` par le nom réel de ton interface LAN (vérifie avec `ip a`).


### Lancement et vérification du service

Démarrer le service DHCP et vérifier son statut :

    sudo systemctl restart isc-dhcp-server  
    sudo systemctl status isc-dhcp-server

**Surveille les logs** pour vérifier le bon fonctionnement :

    sudo journalctl -eu isc-dhcp-server  
    sudo tail -f /var/log/syslog

### Dépannage

- **Adresse du client en 169.254.x.x** : le client ne reçoit pas de bail DHCP. Vérifier la configuration, le service, le câblage/bridge réseau.
- **Erreur "No subnet declaration for [interface]"** : il faut ajouter un bloc `subnet` adapté au réseau de cette interface dans `dhcpd.conf`.
- **Erreur "multiple interfaces match the same subnet"** : ne pas mettre deux interfaces réseau sur le même subnet sur cette machine.
- **Vérifier la syntaxe de config** :
    ```
    sudo dhcpd -t
    ```
    > Affiche l’erreur et la ligne fautive s’il y a une coquille dans la conf.

---

### Ressources

- [Documentation Debian DHCP](https://wiki.debian.org/DHCP_Server)
- [Manuel ISC dhcpd.conf](https://linux.die.net/man/5/dhcpd.conf)
- [Dépannage DHCP sur Debian](https://wiki.debian.org/fr/DHCP_Server)

---

# DNS

## Sommaire

- [Installation de Bind9](#installation-de-bind9)
- [Configuration globale de Bind9](#configuration-globale-de-bind9)
- [Déclaration de la zone DNS](#déclaration-de-la-zone-dns)
- [Création du fichier de zone](#création-du-fichier-de-zone)
- [Vérification et redémarrage du service](#vérification-et-redémarrage-du-service)
- [Tester la résolution DNS](#tester-la-résolution-dns)
- [Dépannage](#dépannage)
- [Ressources](#ressources)

---

### Installation de Bind9

    sudo apt update  
    sudo apt install bind9 dnsutils


---

### Configuration globale de Bind9

#### 1. **Autoriser ton réseau local**

Édite le fichier `/etc/bind/named.conf.options` :

    nano /etc/bind/named.conf.options



Ajoute ces lignes dans `options { ... }` :

    acl "trusted" {  
    172.16.0.0/16;  
    localhost;  
    };
    
    options {  
    directory "/var/cache/bind";  
    recursion yes;  
    allow-query { trusted; };  
    forwarders {  
    8.8.8.8; # DNS Google (optionnel, tu peux en mettre plus ou d'autres)  
    };  
    dnssec-validation auto;  
    auth-nxdomain no; # recommandé pour éviter certains soucis  
    listen-on { any; };  
    listen-on-v6 { any; };  
    };

---

### Déclaration de la zone DNS

Édite `/etc/bind/named.conf.local` :

    nano /etc/bind/named.conf.local


Ajoute :

    zone "starfleet.lan" {  
    type master;  
    file "/etc/bind/db.starfleet.lan";  
    };
---

### Création du fichier de zone

Crée le fichier de zone à partir d’un modèle :

    cp /etc/bind/db.local /etc/bind/db.starfleet.lan  
    nano /etc/bind/db.starfleet.lan


Remplace le contenu ainsi :

    $TTL 604800  
    @ IN SOA ns.starfleet.lan. admin.starfleet.lan. (  
    1 ; Serial (à incrémenter à chaque modif)  
    604800 ; Refresh  
    86400 ; Retry  
    2419200 ; Expire  
    604800 ) ; Negative Cache TTL  
    ;  
    @ IN NS ns.starfleet.lan.  
    ns IN A 172.16.0.100 ; IP du serveur DNS  
    srv1 IN A 172.16.0.100 ; Exemple de client  
    client1 IN A 172.16.0.200 ; Autre client


> `admin.starfleet.lan.` est l’adresse email du contact technique, écris-la au format admin.starfleet.lan. (le @ est remplacé par un point)

**N’oublie PAS d’incrémenter le numéro "Serial" à chaque modification du fichier de zone !**

---

### Vérification et redémarrage du service

#### 1. **Vérifie la syntaxe de Bind9 et du fichier de zone**

    named-checkconf  
    named-checkzone starfleet.lan /etc/bind/db.starfleet.lan

(Aucune sortie = aucune erreur)

#### 2. **Redémarre Bind9**

    systemctl restart bind9  
    systemctl status bind9

---

### Tester la résolution DNS

Depuis le serveur (et aussi, idéalement, depuis un client avec DNS configuré) :

    nslookup srv1.starfleet.lan

(si le client utilise le bon DNS dans `/etc/resolv.conf`)

---

### Dépannage

- **Les clients ne résolvent rien ?**  
  Vérifie que :
  - leur `/etc/resolv.conf` pointe sur l’IP du serveur DNS
  - les règles firewall ne bloquent pas UDP/53
  
- **named-checkconf / named-checkzone retourne une erreur**  
  → Corrige le fichier signalé (souvent un espace oublié, ou point manquant).

- **Impossible d’atteindre le DNS ?**  
  → Ping l’IP du serveur, vérifie la connectivité.

- **Erreur "REFUSED"**  
  → Vérifie la section `allow-query` dans tes options de Bind (`acl trusted`).

- **Pour examiner les logs de Bind9 :**

    sudo journalctl -u bind9  
    sudo tail -f /var/log/syslog
---

### Ressources

- [Wiki Debian Bind9](https://wiki.debian.org/Bind9)
- [Doc Ubuntu Bind9](https://doc.ubuntu-fr.org/bind9)
- [Page de manuel BIND](https://www.isc.org/bind/)

---





