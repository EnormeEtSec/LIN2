# LIN2 - Création d'un serveur WEB
###### Alain Pichonnat, Christophe Kalmann & Sébastien Martin
___

### OBJECTIFS & INFORMATIONS

Le but de ce projet et la mise en place d'une machine Linux "from scratch". C'est à dire sans interface graphique. Le serveur doit permettre d'héberger plusieurs sites web en isolant les utilisateurs des autres afin de rendre impossible l'accès aux fichiers et aux bases de données des autres utilisateurs.

Chaque utilisateur du système aura à sa disposition:

  - Un accès SSH
  - Un site internet (un seul et unique par compte)
  - Une base de donnée

Les services et logiciels utilisés pour la réalisation de ce projet sont les suivants:

  1. Linux Debian 8
  2. Service SSH (client PuTTy)
  3. Serveur Web Nginx
  4. PHP-FPM
  5. MariaDB

___

### INSTALLATION DES SERVICES

#### OpenSSH-Server

Après avoir configurer la machine virtuelle sous Linux Debian 8, installer SSH Server à l'aide de la commande suivante:
```sh
apt-get install OpenSSH-server
```

#### PuTTY - Client SSH
Maintenant il est temps d'installer un client SSH sur toutes les machines avec lesquels on désire communiquer avec le serveur. Pour ce faire nous allons installer le service le plus utiliser: PuTTY.

Le logiciel est téléchargeable à l'adresse suivante:
```sh
http://www.putty.org/
```
Pour cette marche à suivre, nous allons executer toutes les commandes en tant que "root".

#### NGINX - Serveur
Désormais, nous allons installer la dernière version d'NGINX directement depuis de répertoire Debian d'NGINX officiel.
```sh
wget http://nginx.org/keys/nginx_signing.key
apt-key add nginx_signing.key
echo 'deb http://nginx.org/packages/debian/ jessie nginx' >> /etc/apt/sources.list
echo 'deb-src http://nginx.org/packages/debian/ jessie nginx' >> /etc/apt/sources.list
apt-get update && apt-get install nginx
```

>NGINX est un serveur asynchrone. Chaque requête est traitée par un processus dédié.
Ce logiciel de serveur Web est utilisé pour les sites à fort trafic. En effet, son architecture permet de gérer plusieurs connections en même temps. Chaque requête est découpée en plusieurs mini-tâches. Cette architecture se traduit par des perforamces très élevées et une charge mémoire particulièrement faible comparé aux serveurs HTTP classiques comme Apache.    Source: Wikipedia

#### PHP-FPM
À présent, nous allons installer PHP-FPM. NGINX n'a pas de module PHP intégré pour réduire son empreinte mémoire. Voilà pourquoi il faut installer PHP Fast Process Manager.
```sh
apt-get install php5-fpm php5-mysqlnd
```

#### MariaDB - MySQL
Désormais, il est temps d'installer MariaDB.
>MariaDB est une alternative à Oracle MySQL. Elle propose toutes les caractéristiques et options de la solution Oracle. On peut donc la considérer comme un remplacement complet de Oracle MySQL mais sans la marque et avec des particularités en plus.
```sh
apt-get install mariadb-server
```
**IMPORTANT! :** Vous avez absolument besoin d'un mot de passe root particulièrement solide pour MariaDB. Sauvez le quelque part car il faudra l'entré 2 fois lors de l'installation de MariaDB

#### PHP-mcrypt - Module de hashage PHP
L'installation de ce module est obligatoire en terme de sécurité. C'est une librairie compilée en C qui permet d'appeler des fonctions en PHP pour chiffrer/hasher des mots de passe.
```sh
apt-get install php5-gd php5-mcrypt
```

___

### CONFIGURATION

#### Configuration NGINX

Pour débuter la configuration de NGINX, ouvrez le fichier *nginx.conf* de la façon suivante:
```sh
cd /etc/nginx/
nano nginx.conf
```
Nous allons, comme indiqué dans la ligne de commande ci-dessus, modifier le fichier à l'aide de *nano*.

À présent, modifier les lignes où se situe le *user* ainsi:
```sh
user www-data;
worker_processes 1;
```
La commande *worker_process* définit le nombre de processeur que le serveur va utiliser. Dans notre cas 1 seul suffira.

Toujours dans nginx.conf, modifier la ligne:
```sh
access_log on;
# par:
access_log off;
```
Puis rajoutez la ligne suivante au code qui permet de déterminer la taille maximale des fichiers uploadé. La valeur par défault est 10m.
```sh
client_max_body_size 12m;
```

Ensuite vous devez vous rendre dans le dossier */etc/nginx/conf.d*

Dans ce dossier vous trouvez par default deux fichier *default.conf* qui vous permet d'avoir un fichier de referance pour les future configuration de vos domaines.

le deuxième fichier est *example_ssl.conf* qui vous montre une possible configuration via le protocole ssl.

Vous devez pour le bon fonctionnement de votre serveur, déplacer/renomer/supprimer ces 2 fichiers. Pour ensuite en crée un que vous nomerer comme bon vous semble, il doit juste se terminer par *.conf*, il vas servire à configurer votre serveur web.

Chaque fichier que vous metterez dans ce dossier sera un nouveau serveur/domaine.

*domaine1.conf*
```sh
server {
        listen 80;
        listen [::]:80 default_server ipv6only=on;

        #chemin d'accee à votre dossier www
        root /home/alain/www;
        index index.html index.htm index.php;

        # nom de votre domaine
        server_name www.alaindomain.ch;

        location / {
                try_files $uri $uri/ /index.html;
        }

        #localisation de votre serveur php
        location ~ \.php$
        {       
                try_files $uri =404;
                fastcgi_pass   unix:/var/run/php5-fpm.sock;
                fastcgi_param SCRIPT_FILENAME /home/alain/www$fastcgi_script_name;
                include fastcgi_params;
        }
}
```
