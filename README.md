# Notice d’installation et d’utilisation du projet Holodeck

## Sommaire
- [Configuration des VMs](#configuration-des-vms)
- [DHCP](#dhcp)
- [DNS](#dns)
- [NGINX](#nginx)


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
## Installation du serveur DHCP

    apt update  
    apt install isc-dhcp-server -y

## Configuration du DHCP

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

## Affectation de l’interface réseau

Définir sur quelle interface DHCP doit écouter (exemple : `ens34` pour le LAN) :

    nano /etc/default/isc-dhcp-server


Renseigne la ligne suivante (sans # devant) :

    INTERFACESv4="ens34"
    
> Remplace `ens34` par le nom réel de ton interface LAN (vérifie avec `ip a`).


## Lancement et vérification du service

Démarrer le service DHCP et vérifier son statut :

    sudo systemctl restart isc-dhcp-server  
    sudo systemctl status isc-dhcp-server

**Surveille les logs** pour vérifier le bon fonctionnement :

    sudo journalctl -eu isc-dhcp-server  
    sudo tail -f /var/log/syslog

## Dépannage

- **Adresse du client en 169.254.x.x** : le client ne reçoit pas de bail DHCP. Vérifier la configuration, le service, le câblage/bridge réseau.
- **Erreur "No subnet declaration for [interface]"** : il faut ajouter un bloc `subnet` adapté au réseau de cette interface dans `dhcpd.conf`.
- **Erreur "multiple interfaces match the same subnet"** : ne pas mettre deux interfaces réseau sur le même subnet sur cette machine.
- **Vérifier la syntaxe de config** :
    ```
    sudo dhcpd -t
    ```
    > Affiche l’erreur et la ligne fautive s’il y a une coquille dans la conf.

---

## Ressources

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

## Installation de Bind9

    sudo apt update  
    sudo apt install bind9 dnsutils


---

## Configuration globale de Bind9

### 1. **Autoriser ton réseau local**

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

## Déclaration de la zone DNS

Édite `/etc/bind/named.conf.local` :

    nano /etc/bind/named.conf.local


Ajoute :

    zone "starfleet.lan" {  
    type master;  
    file "/etc/bind/db.starfleet.lan";  
    };
---

## Création du fichier de zone

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

## Vérification et redémarrage du service

### 1. **Vérifie la syntaxe de Bind9 et du fichier de zone**

    named-checkconf  
    named-checkzone starfleet.lan /etc/bind/db.starfleet.lan

(Aucune sortie = aucune erreur)

### 2. **Redémarre Bind9**

    systemctl restart bind9  
    systemctl status bind9

---

## Tester la résolution DNS

Depuis le serveur (et aussi, idéalement, depuis un client avec DNS configuré) :

    nslookup srv1.starfleet.lan

(si le client utilise le bon DNS dans `/etc/resolv.conf`)

---

## Dépannage

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

## Ressources

- [Wiki Debian Bind9](https://wiki.debian.org/Bind9)
- [Doc Ubuntu Bind9](https://doc.ubuntu-fr.org/bind9)
- [Page de manuel BIND](https://www.isc.org/bind/)

---


# NGINX

## Sommaire

- [Installation de NGINX depuis le dépôt officiel](#installation-de-nginx-depuis-le-dépôt-officiel)
- [Configuration de base de NGINX](#configuration-de-base-de-nginx)
- [Test du serveur web](#test-du-serveur-web)
- [Dépannage (erreurs fréquentes)](#dépannage-erreurs-fréquentes)
- [Ressources](#ressources)

---

## Installation de NGINX depuis le dépôt officiel

### 1. Ajouter la clé GPG officielle et le dépôt NGINX.ORG

    apt update  
    
    apt install curl gnupg2 ca-certificates lsb-release -y  
    
    curl [https://nginx.org/keys/nginx_signing.key]
    (https://nginx.org/keys/nginx_signing.key) | gpg --dearmor | sudo tee 
    /etc/apt/trusted.gpg.d/nginx.gpg > /dev/null  
    
    echo "deb [http://nginx.org/packages/debian]
    (http://nginx.org/packages/debian) $(lsb_release -cs) nginx" | sudo tee 
    /etc/apt/sources.list.d/nginx.list  
    
    echo -e "Package: *\nPin: origin nginx.org\nPin-Priority: 900\n" | sudo 
    tee /etc/apt/preferences.d/99nginx  
    
    apt update


### 2. Installer NGINX (toute dernière version)

    apt install nginx  
    nginx -v

Doit afficher : nginx version: nginx/1.28.0


---

## Configuration de base de NGINX

### 1. **Vérifier que le service est actif**

    sudo systemctl status nginx

Tu dois voir **active (running)**.  

Sinon, démarre le :

    sudo systemctl start nginx


### 2. **Configuration minimale du site par défaut**

Modifie `/etc/nginx/conf.d/default.conf` (fichier par défaut du dépôt nginx.org) :

    server {  
    listen 80;  
    server_name _;
    
    root /usr/share/nginx/html;
    index index.html index.htm;
    
    location / {
        try_files $uri $uri/ =404;
    }


### 3. **Personnaliser la page d'accueil**


    nano /usr/share/nginx/html/index.html

Ajoute, par exemple :

    Bienvenue sur le serveur NGINX 1.28 (starfleet.lan)

### 4.  **Tester et recharger la configuration**

    sudo nginx -t
    sudo systemctl restart nginx
    
## Test du serveur web

### 1. Test en local :

    curl -I http://localhost

Tu dois avoir une réponse 200 OK.

### 2. Test depuis une autre machine du réseau (navigateur ou console) :

-   URL :  `http://172.16.0.100`
    
-   Ou :
- 

`curl -I http://172.16.0.100`

_Modifie l’IP si nécessaire selon ta config réseau !_


## Dépannage (erreurs fréquentes)

### 1.  **Port 80 déjà utilisé**

**Erreur vue** :  
`bind() to 0.0.0.0:80 failed (98: Address already in use)`

-   Cause : un autre serveur web (ex. Apache2) écoute déjà.
    
-   Solution :
  `systemctl stop apache2
  systemctl disable apache2
  systemctl restart nginx`

### 2.  **Nginx démarre, mais page inaccessible**

-   Vérifie firewall local :

  `ufw allow 80/tcp`

-   Vérifie la config de l’interface réseau de la VM et du bridge dans VMware.

### 3.  **Erreur SSL/certificat (si HTTPS activé)**

-   Message :  `cannot load certificate ...`
    
-   Solution : vérifie les chemins et droits du certificat & clé dans la conf (voir doc SSL de NGINX).  
    Génère un certificat auto-signé en cas de doute :

   

    `sudo openssl req -x509 -days 365 -newkey rsa:2048 -nodes \  
     -keyout /etc/ssl/private/nginx-selfsigned.key \  
     -out /etc/ssl/certs/nginx-selfsigned.crt \  -subj "/CN=starfleet.lan"`


### 4.  **La configuration est correcte (`nginx -t`  OK) mais le service ne démarre pas**

-   Toujours vérifier le log détaillé :

  `journalctl -xeu nginx.service`
    

## Ressources

-   [Docs Officielles NGINX](https://nginx.org/en/docs/)
    
-   [Q&A Nginx nginx.org vs Debian](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)
    
-   [Page de manuel Debian](https://wiki.debian.org/Nginx)






