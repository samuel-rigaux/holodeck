# Notice d’installation et d’utilisation du projet Holodeck

## Sommaire
- [Configuration des VMs](#configuration-des-vms)
- [DHCP](#dhcp)
- [DNS](#dns)
- [NGINX](#nginx)
- [PHP](#php)


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



# PHP

>  🌌 Installation et Configuration multi-PHP (7.x & 8.x) sur NGINX +
> Debian 12

## Sommaire

- [Objectif](#objectif)
- [Pré-requis](#pré-requis)
- [Étape 1 - Installation de NGINX depuis le dépôt officiel](#étape-1---installation-de-nginx-depuis-le-dépôt-officiel)
- [Étape 2 - Ajout de PHP 7.x & PHP 8.x depuis le dépôt Sury.org](#étape-2---ajout-de-php-7x--php-8x-depuis-le-dépôt-suryorg)
- [Étape 3 - Configuration PHP-FPM (sockets)](#étape-3---configuration-php-fpm-sockets)
- [Étape 4 - Création des virtual hosts NGINX](#étape-4---création-des-virtual-hosts-nginx)
- [Étape 5 - Préparation des dossiers et pages de test](#étape-5---préparation-des-dossiers-et-pages-de-test)
- [Étape 6 - Installation de phpMyAdmin](#étape-6---installation-de-phpmyadmin)
- [Étape 7 - Test d'accès & vérification PHP](#étape-7---test-daccès--vérification-php)
- [Dépannages et difficultés rencontrées](#dépannages-et-difficultés-rencontrées)
- [Ressources utiles](#ressources-utiles)

---

## Objectif

Installer un environnement web sous Debian 12 permettant :
- De faire cohabiter **PHP 7.x** et **PHP 8.x** en parallèle avec NGINX (versions récentes hors dépôts Debian).
- Héberger plusieurs sites sur différents virtual hosts :
  - **www8.starfleet.lan** → site web avec PHP 8
  - **www7.starfleet.lan** → site web avec PHP 7
  - **php.starfleet.lan** → interface phpMyAdmin
  - **admin.starfleet.lan** → interface d’administration statique

---

## Pré-requis

- Deux machines/VM sur le même réseau LAN (ex : 172.17.0.100 serveur, 172.17.0.5 client)
- Accès root/sudo sur le serveur Debian 12
- DNS LAN ou modifications du fichier `/etc/hosts` côté client pour les sous-domaines
- Pas d’Apache ou autre serveur web actif occupant le port 80

---

## Étape 1 - Installation de NGINX depuis le dépôt officiel

    sudo apt update  
    sudo apt install curl gnupg2 ca-certificates lsb-release -y  
    curl [https://nginx.org/keys/nginx_signing.key](https://nginx.org/keys/nginx_signing.key) | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/nginx.gpg > /dev/null  
    echo "deb [http://nginx.org/packages/debian](http://nginx.org/packages/debian) $(lsb_release -cs) nginx" | sudo tee /etc/apt/sources.list.d/nginx.list  
    echo -e "Package: *\nPin: origin nginx.org\nPin-Priority: 900\n" | sudo tee /etc/apt/preferences.d/99nginx  
    sudo apt update  
    sudo apt install nginx

*Vérifie la version :*

    nginx -v

> nginx version: nginx/1.28.x


---

## Étape 2 - Ajout de PHP 7.x & PHP 8.x depuis le dépôt Sury.org


    apt install -y ca-certificates apt-transport-https software-properties-common  
    wget -qO /etc/apt/trusted.gpg.d/php.gpg [https://packages.sury.org/php/apt.gpg](https://packages.sury.org/php/apt.gpg)  
    echo "deb [https://packages.sury.org/php/](https://packages.sury.org/php/) $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list  
    apt update  
    apt install php7.4-fpm php7.4-mysql php7.4-curl php7.4-mbstring  
    apt install php8.1-fpm php8.1-mysql php8.1-curl php8.1-mbstring


---

## Étape 3 - Configuration PHP-FPM (sockets)

Vérifie la présence des sockets via :

    ls -l /run/php/
    
Les fichiers suivants doivent apparaître :
- `/run/php/php7.4-fpm.sock`
- `/run/php/php8.1-fpm.sock`

Les services doivent être actifs :

    systemctl status php7.4-fpm  
    systemctl status php8.1-fpm

---

## Étape 4 - Création des virtual hosts NGINX

Crée un fichier de conf par site dans `/etc/nginx/conf.d/` :

### www8.starfleet.lan (PHP 8)

    server {  
    listen 80;  
    server_name www8.starfleet.lan;  
    root /var/www/www8;  
    index index.php index.html;  
    location / {  
    try_files $uri $uri/ =404;  
    }  
    location ~ .php$ {  
    include fastcgi_params;  
    fastcgi_pass unix:/run/php/php8.1-fpm.sock;  
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
    }  
    }


### www7.starfleet.lan (PHP 7)


    server {  
    listen 80;  
    server_name www7.starfleet.lan;  
    root /var/www/www7;  
    index index.php index.html;  
    location / {  
    try_files $uri $uri/ =404;  
    }  
    location ~ .php$ {  
    include fastcgi_params;  
    fastcgi_pass unix:/run/php/php7.4-fpm.sock;  
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
    }  
    }


### php.starfleet.lan (phpMyAdmin, via PHP 8)

    server {  
    listen 80;  
    server_name php.starfleet.lan;  
    root /usr/share/phpmyadmin;  
    index index.php;  
    location / {  
    try_files $uri $uri/ =404;  
    }  
    location ~ .php$ {  
    include fastcgi_params;  
    fastcgi_pass unix:/run/php/php8.1-fpm.sock;  
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
    }  
    }


### admin.starfleet.lan (statique)

    server {  
    listen 80;  
    server_name admin.starfleet.lan;  
    root /var/www/admin;  
    index index.html;  
    location / {  
    try_files $uri $uri/ =404;  
    }  
    }


⚠️ **Attention à bien nommer les `server_name` et à retirer tout site par défaut gênant dans les confs !**

---

## Étape 5 - Préparation des dossiers et pages de test

    mkdir -p /var/www/www8 /var/www/www7 /var/www/admin  
    echo "<?php phpinfo(); ?>" | sudo tee /var/www/www8/phpinfo.php  
    echo "<?php phpinfo(); ?>" | sudo tee /var/www/www7/phpinfo.php  
    echo "<h1>Administration VM</h1>" | sudo tee /var/www/admin/index.html  
    chown -R www-data:www-data /var/www  
    chmod -R 755 /var/www

---

## Étape 6 - Installation de phpMyAdmin

    apt install phpmyadmin

Lors de l'installation, choisis "apache2" (même si tu utilises NGINX, on sert quand même les fichiers).

La racine de phpMyAdmin devient `/usr/share/phpmyadmin`; modifie ton vhost en conséquence.

---

## Étape 7 - Test d'accès & vérification PHP

Sur ta **VM cliente** (après avoir ajouté dans son `/etc/hosts` : tous les sous-domaines pointant sur l’IP serveur) :

    curl -H "Host: www8.starfleet.lan"  [http://172.17.0.100/phpinfo.php](http://172.17.0.100/phpinfo.php)  | grep "PHP Version"  
    curl -H "Host: www7.starfleet.lan"  [http://172.17.0.100/phpinfo.php](http://172.17.0.100/phpinfo.php)  | grep "PHP Version"
    
- Tu dois voir la version de PHP correspondante à chaque site.

Test phpMyAdmin dans un navigateur :  
`http://php.starfleet.lan/`

Test page admin statique :  
`http://admin.starfleet.lan/`

---

## Dépannages et difficultés rencontrées

**Problèmes fréquents et résolus lors du projet :**

- **Erreur 502 Bad Gateway** :  
  - PHP-FPM éteint ou chemin du socket incorrect (corrigé en vérifiant `/run/php/` et la config NGINX).
  - Mauvais utilisateur/config user dans NGINX (relu `user www-data;` dans nginx.conf).

- **Page NGINX de base sur tous les domaines** :
  - Vhosts/mauvais server_name ou DNS/hosts non configuré côté client.
  - Solution : configurer `/etc/hosts` et vérifier les server_name dans chaque vhost.

- **Résolution DNS/LAN** :
  - Impossible d’accéder aux domaines sans résolution réseau adaptée. Résolu avec `/etc/hosts` côté VM cliente.
- **Erreur "http_auth_ldap: Could not connect"** :
  - Un bloc LDAP resté dans la conf (obligeant NGINX à authentifier sur un serveur LDAP inexistant).  
  - Solution : commenter/supprimer toute la conf ldap_server et toute directive d’auth_ldap.
- **Problème d’accès LAN sans Internet** :
  - Le LAN VM n’a pas besoin d’internet pour communiquer. Il faut uniquement que les VMs soient sur le même réseau virtuel.
- **phpMyAdmin : mot de passe oublié** :
  - Les identifiants sont ceux des utilisateurs MySQL/MariaDB, à régénérer via `sudo mysql` si besoin.

---

## Ressources utiles

- [NGINX.org Linux packages](https://nginx.org/en/linux_packages.html)
- [PHP Sury.org & multi-versions](https://deb.sury.org/)
- [Documentation NGINX + PHP-FPM](https://www.nginx.com/resources/wiki/start/topics/tutorials/phpfastcgionnginx/)
- [phpMyAdmin](https://www.phpmyadmin.net/)

---

# Maria DB

> Installation et Configuration de la dernière version MariaDB sur
> Debian 12

## Sommaire

- [Objectif](#objectif)
- [Pré-requis](#pré-requis)
- [Étape 1 - Ajout du dépôt officiel MariaDB](#étape-1---ajout-du-dépôt-officiel-mariadb)
- [Étape 2 - Installation de MariaDB](#étape-2---installation-de-mariadb)
- [Étape 3 - Sécurisation de l’installation MariaDB](#étape-3---sécurisation-de-linstallation-mariadb)
- [Étape 4 - Vérification de l’installation](#étape-4---vérification-de-linstallation)
- [Étape 5 - Création de bases de données et utilisateurs](#étape-5---création-de-bases-de-données-et-utilisateurs)
- [Étape 6 - Configuration de l’accès distant (optionnel)](#étape-6---configuration-de-laccès-distant-optionnel)
- [Dépannages fréquents](#dépannages-fréquents)
- [Ressources utiles](#ressources-utiles)

---

## Objectif

Ce guide explique comment installer la version la plus récente de MariaDB sur Debian 12 en utilisant le dépôt officiel MariaDB (hors dépôts Debian standards).  
Il couvre l’installation, la sécurisation, la création de bases et utilisateurs, ainsi que l’accès distant optionnel.

---

## Pré-requis

- Machine Debian 12 avec accès root ou sudo
- Connexion internet active (au moins pendant l’installation)
- Connaissances basiques en ligne de commande Linux

---

## Étape 1 - Ajout du dépôt officiel MariaDB

1. Installe les outils nécessaires pour ajouter des dépôts HTTPS :

```
sudo apt update
```

```
sudo apt install software-properties-common dirmngr apt-transport-https ca-certificates -y
```


4. Importer la clé GPG officielle MariaDB :
```
sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
```


5. Ajouter le dépôt MariaDB adapté à Debian 12 (Bookworm) :

```
echo "deb [arch=amd64,arm64,ppc64el] https://mirror.mariadb.org/repo/11.1/debian bookworm main" | sudo tee /etc/apt/sources.list.d/mariadb.list
```


*Note : `11.1` correspond à la version majeure actuelle. Vérifie la dernière version stable sur le site officiel MariaDB.*

---

## Étape 2 - Installation de MariaDB

Met à jour la liste des paquets avec le nouveau dépôt :

    sudo apt update


Installe MariaDB serveur et client :

    sudo apt install mariadb-server mariadb-client

---

## Étape 3 - Sécurisation de l’installation MariaDB

Lance l’outil interactif pour sécuriser MariaDB :

    sudo mysql_secure_installation


Tu seras amené à :

- Définir ou modifier le mot de passe root MariaDB
- Supprimer les utilisateurs anonymes
- Désactiver la connexion root distante (recommandé)
- Supprimer la base de test non sécurisée
- Recharger les privilèges

---

## Étape 4 - Vérification de l’installation

Vérifie que le service est actif :

    sudo systemctl status mariadb


Connecte-toi à MariaDB en root :

sudo mysql -u root -p


Vérifie la liste des bases de données :

    SHOW DATABASES;


---

## Étape 5 - Création de bases de données et utilisateurs

Pour créer une base et un utilisateur avec tous les privilèges :


    CREATE DATABASE starfleet_db;  
    CREATE USER 'starfleet_user'@'localhost' IDENTIFIED BY 'MotDePasseTrèsSûr!';  
    GRANT ALL PRIVILEGES ON starfleet_db.* TO 'starfleet_user'@'localhost';  
    FLUSH PRIVILEGES;  
    EXIT;

>  Dans mon cas le mot de passe sera 'root'


---

## Étape 6 - Configuration de l’accès distant (optionnel)

Si tu souhaites accéder à MariaDB depuis d’autres machines partageant le réseau :

1. Édite le fichier de configuration `/etc/mysql/mariadb.conf.d/50-server.cnf`.

2. Modifie la ligne `bind-address` de :

```
bind-address = 127.0.0.1
```
en

    bind-address = 0.0.0.0



3. Crée un utilisateur MariaDB avec accès distant :

```
CREATE USER 'starfleet_user'@'172.17.0.%' IDENTIFIED BY 'MotDePasseTrèsSûr!';  
GRANT ALL PRIVILEGES ON starfleet_db.* TO 'starfleet_user'@'172.17.0.%';  
FLUSH PRIVILEGES;
```

4. Redémarre MariaDB pour prendre en compte ce changement :

```
sudo systemctl restart mariadb
```

---

## Dépannages fréquents

- **Impossible de démarrer MariaDB** : vois les logs dans `/var/log/mysql/error.log` ou utilise `journalctl -xeu mariadb.service`.
- **Erreur d’accès refusé** : vérifier la définition des utilisateurs et privilèges dans MariaDB.
- **MariaDB n’écoute pas sur le port 3306 distant** : vérifier `bind-address` et firewall (ports ouverts).
- **Problèmes de clés GPG** : assure-toi que la clé est bien importée et que le système est à jour.

---

## Ressources utiles

- [Site officiel MariaDB](https://mariadb.org/)
- [Dépôt officiel MariaDB Linux](https://mariadb.org/download/)
- [Guide MariaDB Debian](https://mariadb.com/kb/en/installing-mariadb-debian/)

---

















