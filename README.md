# Réplication-Bookstack
##### Installation et configuration de bookstack

## guide complet pour installer et configurer BookStack en mode haute disponibilité (srv1/srv2) utilisant l'adresse IP virtuelle

# Mettez à jour tous les paquets logiciels des systèmes :

    apt update && sudo apt upgrade -y

# Installation de Apache, PHP, MySQL et leurs extensions nécessaires sur chacun des serveurs Web :
# Installer Apache HTTP Server :

    sudo apt install apache2 -y

# Activation module mod_rewrite pour Apache afin de gérer les URL amicales :

    sudo a2enmod rewrite

# Création de la structure du répertoire pour BookStack sous /var/www/html :

    sudo mkdir -p /var/www/html/bookstack/{app,public}

# Définir les propriétaires et groupes appropriés pour cette structure de répertoire :

    sudo chown -R $USER:$USER /var/www/html/bookstack

# être sur que les permissions sont sécurisées :

    find /var/www/html/bookstack -type d -exec chmod 750 {} \;
    find /var/www/html/bookstack -type f -exec chmod 640 {} \;


# l'installation de PHP et ses modules requis :

    sudo apt install php libapache2-mod-php php-{cli,common,fpm,mbstring,xml,zip,gd,mysql,curl,bcmath,json,ldap,gmp,opcache,redis,imagick,pear} -y

# Modification de la configuration globale de PHP pour inclure les directives supplémentaires nécessaires au bon fonctionnement de BookStack :

    sudo sed -ri 's!upload_max_filesize = .+!upload_max_filesize = 10M!g' /etc/php/*/fpm/php.ini
    sudo sed -ri 's!post_max_size = .+!post_max_size = 20M!g' /etc/php/*/fpm/php.ini
    sudo sed -ri 's!memory_limit = .+!memory_limit = 512M!g' /etc/php/*/fpm/php.ini

# Redémarrage de service FPM PHP après avoir effectué les modifications ci-dessus :

    sudo systemctl restart phpX.X-fpm
(Remplacez X.X par la version spécifique de PHP.)

# Pour autoriser les sites web à utiliser Composer (gestionnaire de packages PHP), il faut installer Git et Composer sur les deux serveurs (srv1-d12 et srv2-d12) :

    sudo apt install git composer -y

# Il faut créer ensuite un lien symbolique vers la dernière version stable de BookStack en tant que nouvelle application Laravel :

    cd /var/www/html/bookstack/app
    composer create-project --prefer-dist thujohn/pdf laravel-bookstack
    rm -rf public/*
    cp -r laravel-bookstack/public/. ./
    mv laravel-bookstack/vendor app/HQ > /dev/null

# Maintenant, créez une base de données MariaDB/MySQL pour BookStack et importez sa structure schématique initiale :
# Connexion à MariaDB/MySQL :

    sudo mariadb -u root -p

# Création d'une base de données et un nouvel utilisateur privilégié :

    CREATE DATABASE bookstack CHARACTER SET utf8 COLLATE utf8_unicode_ci;
    GRANT ALL PRIVILEGES ON bookstack.* TO 'bookstackuser'@'localhost' IDENTIFIED BY '<choose_a_password>';
    EXIT;

# Importation de la structure de schéma de BookStack :

    wget https://github.com/BookStackApp/BookStack/archive/refs/tags/v2.1.21.zip
    unzip v2.1.21.zip
    mysqldump -u root -p < bookstack-2.1.21/structure.sql | mysql -u root -p bookstack
    rm -rf v2.1.21.zip bookstack-2.1.21

# Configuration de l'environnement BookStack :
# Copier le fichier de configuration exemple :

    cp /var/www/html/bookstack/app/config.sample.env /var/www/html/bookstack/app/.env

# Éditer le nouveau fichier .env :

    nano /var/www/html/bookstack/app/.env

# Modifiez les éléments suivants selon vos besoins :

    APP_URL=http://<mon_virtual_ip> # Remplacer par votre IP virtuelle
    DB_*_HOST=localhost # Laisser tel quel
    DB_*_DATABASE=bookstack
    DB_*_USERNAME=bookstackuser
    DB_*_PASSWORD=<choose_a_password> # Mot de passe choisi précédemment
    CACHE_DRIVER=array
    SESSION_DRIVER=database
    QUEUE_CONNECTION=sync
    LOG_CHANNEL=single
    BROADCAST_DRIVER=log

# Activation de SSL pour assurer une connexion sécurisée (facultatif).
# d'une autorité de certification racine, générez automatiquement un certificat auto-signé pour 
# tester l'environnement. Sur chaque machine :

    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -subj '/CN=*.<your_domain>' -keyout /etc/ssl/private/nginx-selfsigned.key \
    -out /etc/ssl/certs/nginx-selfsigned.crt

# Il faut s'assurer que la pile complète LEMP est installée et configurée pour prendre en charge SSL :

    sudo apt install nginx -y
    
# Récupérez ou concevez notre propre bloc NGINX personnalisé prenant en charge SSL
# ...

# Une fois terminé, activez et redémarrez les services concernés :

    sudo systemctl enable apache2
    sudo systemctl restart apache2



