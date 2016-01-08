# LIN2 - Création d'un serveur WEB

Auteur :

* Alain Pichonnat
* Christophe Kalman
* Sébastien Martin

## OBJECTIFS & INFORMATIONS

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
  6. iptables

## Configuration matériel

Configurtaion de la machine virtuelle :

* Distribution : Debian v8.0.0 64bits
* Processeur : 2 CPU de 3.30 Ghz
* Mémoire vive : 2 Gb
* Disque dur : 20 Gb
* Carte réseau

Concernant l'installation de debian, nous laisson tous les paramètres par défaut.  

## INSTALLATION DES SERVICES

### OpenSSH-Server

Après avoir configurer la machine virtuelle sous Linux Debian 8, installer SSH Server à l'aide de la commande suivante:
```sh
apt-get install openssh-server
```
#### Configuration

Nous allons maintenant sécuriser est configurer le serveur ssh.

Editer le fichier de configuration :

```sh
nano /etc/ssh/sshd_config
```

```sh
# Si vous ne trouvez pas le paramètre créer le.
# Rechercher le aramètre PermitRootLogin est mettez-le a no
PermitRootLogin no

# Permet à ssh de rechercher la clef public dans le dossier home de l'utilisateur
AuthorizedKeysFile      %h/.ssh/authorized_keys

# Autoriser  la connexion avec les clefs publique/privé
PubkeyAuthentication yes
RSAAuthentication yes
```

#### Génération d'un jeu de clef RSA.

Cette partie permet de générer le jeu de clef public / privée.

```sh
mkdir ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t rsa

# Autoriser la clef génée
# Exemple toto@debian
ssh-copy-id <username>@<host>
```

#### Récupération de la clef depuis windows

Afficher votre clef est copier la depuis putty :

```sh
cat ~/.ssh/id_rsa
```

Enregistrer cette clef privée non convertie dans un document de texte.

Convertissez cette clef avec l'outil PuTTYgen :
http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html

```
Conversion -> ImportKey -> votre clef privée importée.
```

Ensuite sauvegarder votre clef convertie : save private key.

Une fois la clef convertie vous pouvez l'utiliser dans putty:

```
Category -> Connection -> SSH -> Auth : Private key file for authentication.
```
### PuTTY - Client SSH

Maintenant il est temps d'installer un client SSH sur toutes les machines avec lesquels on désire communiquer avec le serveur. Pour ce faire nous allons installer le service le plus utiliser: PuTTY.

Le logiciel est téléchargeable à l'adresse suivante:

```sh
http://www.putty.org/
```
Pour cette marche à suivre, nous allons executer toutes les commandes en tant que "root".

### NGINX - Serveur

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

### PHP-FPM
À présent, nous allons installer PHP-FPM. NGINX n'a pas de module PHP intégré pour réduire son empreinte mémoire. Voilà pourquoi il faut installer PHP Fast Process Manager.
```sh
apt-get install php5-fpm php5-mysqlnd
```
#### CONFIGURATION

Premiérement nous commençons par crée et configurer un fichier qui se trouve dans */etc/php5/fpm/pool.d*.

Pour la configuration je l'est nome *site_site1_php.conf* mais donné lui un nom qui se rapport au ficher de conf de nginx car il y a aussi un fichier de conf par utilisateurs.

Cette configuration va servire à séparer chaque service de chaque utilisateur, donc chaque user aura son service php-fpm.

Voici les config:

*site_site1_php.conf*

```sh
[site_site1]

user = site1
group = site1

# faire attention à la ligne suivant bien rajouter le "-site1"
listen = /var/run/php5-fpm-site1.sock
listen.owner = www-data
listen.group = www-data
php_admin_value[disable_functions] = exec,passthru,shell_exec,system
php_admin_flag[allow_url_fopen] = off
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
chdir = /
```

L'utilisateur site1 doit bien évidement être créé.

### MariaDB - MySQL

Désormais, il est temps d'installer MariaDB.
> MariaDB est une alternative à Oracle MySQL. Elle propose toutes les caractéristiques et options de la solution Oracle. On peut donc la considérer comme un remplacement complet de Oracle MySQL mais sans la marque et avec des particularités en plus.

```sh
apt-get install mariadb-server
```

**IMPORTANT! :** Vous avez absolument besoin d'un mot de passe root particulièrement solide pour MariaDB. Sauvez le quelque part car il faudra l'entré 2 fois lors de l'installation de MariaDB

### PHP-mcrypt - Module de hashage PHP

L'installation de ce module est obligatoire en terme de sécurité. C'est une librairie compilée en C qui permet d'appeler des fonctions en PHP pour chiffrer/hasher des mots de passe.

```sh
apt-get install php5-gd php5-mcrypt
```

## CONFIGURATION

### Configuration NGINX

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

Resultat final pour le fichier **nginx.conf** :

```sh
user  www-data;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log on;
    client_max_body_size 12m;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/sites-available/*.conf;
}

```
Déplacez-vous dans le dossier **conf.d** :

```sh
cd /etc/nginx/conf.d
```

Dans ce dossier vous trouvez par default deux fichier *default.conf* qui vous permet d'avoir un fichier de referance pour les future configuration de vos domaines.

Le deuxième fichier est *example_ssl.conf* qui vous montre une possible configuration via le protocole ssl.

Vous devez pour le bon fonctionnement de votre serveur, déplacer/renomer/supprimer ces 2 fichiers. Pour ensuite en crée un que vous nomerer comme bon vous semble, il doit juste se terminer par *.conf*, il vas servire à configurer votre serveur web.

Chaque fichier que vous metterez dans ce dossier sera un nouveau serveur/domaine.

Voici un exemple de fichier : **domaine1.conf**.

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

### Configuration MariaDB

Nous allons maintenant configurer est sécurisé notre base de données. Tout d’abord nous allons exécuté le scripte **mysql_secure_installation**. Ce scripte vas nous guider à travers certaines procédures qui élimineront certaines valeurs par défaut qui sont dangereux à utiliser dans un environnement de production.

```sh
sudo mysql_secure_installation
```

Suivez est lisez scrupuleusement les étapes du scripte.

1. Entrer le mot de passe pour l'utilisateur root
2. Changer le mot de passe root si celui-ci n'est pas un mot de passe fort.
3. Supprimer tous les utilisateur anonyme -> Y
4. Interdire l'accès de root à distance ? -> Y
5. Supprimer la base de donnée de test -> Y
6. Recharger tous les privilèges -> Y

Voici un aperçus de l’exécution du scripte :

```sh
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from "localhost".  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named "test" that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you ve completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

Ensuite nous allons éditer le fichier de configuration de mysql :

```sh
nano /etc/mysql/my.cnf
```

Ajoutez dans la section [mysqld] l'adresse du fichier de log

```sh
log=/var/log/mysql-logfile
```

Puis nous allons désactiver la fonction qui permet d'accéder au système de fichiers sous-jacents au sein de MySQL toujours dans la section [mysqld] :

```sh
local-infile=0
```

### Ajout d'un nouvel utilisateur

Voici la commande pour créer un nouvel utilisateur **newuser**:

```sql
CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'password';
```

voici la commande pour créer la base de donnée **testdatabase**

```sql
CREATE DATABASE testdatabase;
```

voici la commande pour ajouter les permission à l'utilisateur **newuser**:

```sql
GRANT SELECT,UPDATE,CREATE,DELETE,ALTER,DROP ON TABLE testdatabase.* TO 'newuser'@'localhost';
```

### GESTION DES UTILISATEURS

L'ajout des utilisateurs se ferra pour l'instant manuellement. Il sera donc nécessaire aux utilisateurs de nous contacter pour créer un compte.

Pour ajouter un utilisateurs il faut se rendre dans le répertoire *home* en tant que *root*.

```sh
cd /home
```

Puis créer un utilisateur à l'aide de la commande suivante:

```sh
adduser lenomdutilisateur
```

> A noter qu'il faut faire attention aux règles RegEx pour l'ajout d'utilisateur. En effet, il est par exemple impossible d'utiliser des lettres majuscule.

Une fois la commande passé, le système vous demandera de choisir un mot de passe pour cette utilisateur ainsi que d'entrer certaines données facultative comme le numéro de téléphone.

Afin de garantir un accès exclusif à l'utilisateur à son répertoire personnel, faites un *chmod 750*. Cette commande de permission permettra aussi au groupe, auquel appartient le répertoire *home* de l'utilisateur, d'accéder aux fichiers de ce derniersans toutefois pouvoir les modifier.

```sh
chmod 750 nomrepertoirehomeutilisateur
```

Les autres utilisateurs ne pourront même pas voir le répertoire. Ils n'auront donc aucune connaissance de son existence.

Désormais, ajoutez le groupe *www-data* au répertoire home de votre utilisateur:

```sh
chgrp www-data nomrepertoirehomeutilisateur
```

> A noter que le nom du répertoire home est le même que le nom de l'utilisateur.

Félicitations! Vous avez créé et gérer les droits de votre premier utilisateur :neckbeard:

### Configuration iptables

Nous allons configurer iptables afin d’avoir une sécurité forte. Certaines parties ont été volontairement commentées. Toutes les commandes seront exécutées en root.

Si vous souhaitez empêcher toutes communication entre les services via l’adresse de bouclage ( localhost ) ainsi que d’empêcher l’usurpation d’identité. Dé commenter la variable

```sh
SPOOFIP
```

Ainsi que toutes la section du code :

```sh
# Log and block spoofed ips
```

```sh
#!/bin/bash
IPT="/sbin/iptables"

#### IPS ######
# Get server public ip
SERVER_IP=$(ifconfig eth0 | grep 'inet addr:' | awk -F'inet addr:' '{ print $2}' | awk '{ print $1}')
#LB1_IP="172.17.103.231"
#LB2_IP="204.54.1.2"

# Do some smart logic so that we can use damm script on LB2 too
#OTHER_LB=""
#SERVER_IP=""
#[[ "$SERVER_IP" == "$LB1_IP" ]] && OTHER_LB="$LB2_IP" || OTHER_LB="$LB1_IP"
#[[ "$OTHER_LB" == "$LB2_IP" ]] && OPP_LB="$LB1_IP" || OPP_LB="$LB2_IP"

#### FILES #####
BLOCKED_IP_TDB=/root/.fw/blocked.ip.txt
#SPOOFIP="127.0.0.0/8 192.168.0.0/16 172.16.0.0/12 10.0.0.0/8 169.254.0.0/16 0.0.0.0/8 240.0.0.0/4 255.255.255.255/32 168.254.0.0/16 224.0.0.0/4 240.0.0.0/5 248.0.0.0/5 192.0.2.0/24"
BADIPS=$( [[ -f ${BLOCKED_IP_TDB} ]] && egrep -v "^#|^$" ${BLOCKED_IP_TDB})

### Interfaces ###
PUB_IF="eth0"   # public interface
LO_IF="lo"      # loopback

### start firewall ###
echo "Setting LB1 $(hostname) Firewall..."

# DROP and close everything
$IPT -P INPUT DROP
$IPT -P OUTPUT DROP
$IPT -P FORWARD DROP

# Unlimited lo access
$IPT -A INPUT -i ${LO_IF} -j ACCEPT
$IPT -A OUTPUT -o ${LO_IF} -j ACCEPT


# Drop sync
$IPT -A INPUT -i ${PUB_IF} -p tcp ! --syn -m state --state NEW -j DROP

# Drop Fragments
$IPT -A INPUT -i ${PUB_IF} -f -j DROP

$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags ALL FIN,URG,PSH -j DROP
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags ALL ALL -j DROP

# Drop NULL packets
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags ALL NONE -m limit --limit 5/m --limit-burst 7 -j LOG --log-prefix " NULL Packets "
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags ALL NONE -j DROP

$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags SYN,RST SYN,RST -j DROP

# Drop XMAS
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags SYN,FIN SYN,FIN -m limit --limit 5/m --limit-burst 7 -j LOG --log-prefix " XMAS Packets "
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP

# Drop FIN packet scans
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags FIN,ACK FIN -m limit --limit 5/m --limit-burst 7 -j LOG --log-prefix " Fin Packets Scan "
$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags FIN,ACK FIN -j DROP

$IPT  -A INPUT -i ${PUB_IF} -p tcp --tcp-flags ALL SYN,RST,ACK,FIN,URG -j DROP

# Log and get rid of broadcast / multicast and invalid
$IPT  -A INPUT -i ${PUB_IF} -m pkttype --pkt-type broadcast -j LOG --log-prefix " Broadcast "
$IPT  -A INPUT -i ${PUB_IF} -m pkttype --pkt-type broadcast -j DROP

$IPT  -A INPUT -i ${PUB_IF} -m pkttype --pkt-type multicast -j LOG --log-prefix " Multicast "
$IPT  -A INPUT -i ${PUB_IF} -m pkttype --pkt-type multicast -j DROP

$IPT  -A INPUT -i ${PUB_IF} -m state --state INVALID -j LOG --log-prefix " Invalid "
$IPT  -A INPUT -i ${PUB_IF} -m state --state INVALID -j DROP

# Log and block spoofed ips
#$IPT -N spooflist
#for ipblock in $SPOOFIP
#do
#$IPT -A spooflist -i ${PUB_IF} -s $ipblock -j LOG --log-prefix " SPOOF List Block "
#$IPT -A spooflist -i ${PUB_IF} -s $ipblock -j DROP
#done
#$IPT -I INPUT -j spooflist
#$IPT -I OUTPUT -j spooflist
#$IPT -I FORWARD -j spooflist
# Log and block spoofed ips END

# SSH
$IPT -A INPUT -i ${PUB_IF}  -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
$IPT -A OUTPUT -o ${PUB_IF}  -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT


# allow incoming ICMP ping pong stuff
$IPT -A INPUT -i ${PUB_IF} -p icmp --icmp-type 8 -s 0/0 -m state --state NEW,ESTABLISHED,RELATED -m limit --limit 30/sec  -j ACCEPT
$IPT -A OUTPUT -o ${PUB_IF} -p icmp --icmp-type 0 -d 0/0 -m state --state ESTABLISHED,RELATED -j ACCEPT

# allow incoming HTTP port 80
$IPT -A INPUT -i ${PUB_IF} -p tcp -s 0/0 --sport 1024:65535 --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
$IPT -A OUTPUT -o ${PUB_IF} -p tcp --sport 80 -d 0/0 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT


# allow outgoing ntp
$IPT -A OUTPUT -o ${PUB_IF} -p udp --dport 123 -m state --state NEW,ESTABLISHED -j ACCEPT
$IPT -A INPUT -i ${PUB_IF} -p udp --sport 123 -m state --state ESTABLISHED -j ACCEPT

# allow outgoing smtp
$IPT -A OUTPUT -o ${PUB_IF} -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
$IPT -A INPUT -i ${PUB_IF} -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT

### add your other rules here ####

#######################
# drop and log everything else
$IPT -A INPUT -m limit --limit 5/m --limit-burst 7 -j LOG --log-prefix " DEFAULT DROP "
$IPT -A INPUT -j DROP

#######################
# Allow user
#######################

$IPT -A INPUT -i ${PUB_IF} -p tcp -s 0/0 --sport 1024:65535 --dport 90 -m state --state NEW,ESTABLISHED -j ACCEPT
$IPT -A OUTPUT -o ${PUB_IF} -p tcp --sport 80 -d 0/0 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT


exit 0
```

Rendez le scripte exécutable :

```sh
# Rendre exécutable
Chmod u+x ./nom_du_scripte

# Exécuter le scripte
./nom_du_scripte
```


Règle pour autoriser un nouveau client :

```sh
# Ajouter un nouveau client pour le port : XXXXX
# Remplacer XXXXX par le port du client exemple 80.
iptables -A INPUT -i eth0 -p tcp -s 0/0 --sport 1024:65535 --dport XXXXX -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport XXXXX -d 0/0 --dport 1024:65535 -m state --state ESTABLISHED -j ACCEPT
```

#### Sauvegarder la configuration iptables afin de la rendre persistante

Installer le paquet :

```sh
apt-get install iptables-persistent
```

Ensuite pour sauvegarder les régles enregistrer les dans le fichier *rules* :

```sh
iptables-save > /etc/iptables/rules
```

Recharger le parefeu au suite a un démarage :

```sh
iptables-restore < /etc/iptables/rules
```

Créer un scripte qui se lancera automatiquement au démarrage du serveur :

```sh
#! /bin/sh
iptables-restore < /etc/iptables/rules
```

Enregistrer le dans le dossier sous le nom de **iptables-persistent.sh**

```sh
/etc/init.d
```
#### Ajouter les droits d’exécution au fichier

```sh
chmod 755 /etc/init.d/iptables-persistent.sh
```

Maintenant à chaque démarrage iptables sera configuré en fonction du fichier iptables-persistent.sh
