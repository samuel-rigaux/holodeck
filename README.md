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



