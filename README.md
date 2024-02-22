# Partie 1 : Prérequis
# Installez les prérequis minimaux sur les deux serveurs, Srv1-D12 et Srv2-D12, 
# incluant Apache, PHP, MariaDB, Git et Composer 
# Serveur 1 et Serveur 2

# Mettre à jour le système :

    sudo apt update && sudo apt upgrade -y

# Installer Apache, PHP et MariaDB :

    sudo apt install apache2 php libapache2-mod-php mariadb-server -y

# Activer les modules PHP requis :

    sudo apt install php-{cli,common,fpm,mbstring,xml,zip,gd,mysql,curl,bcmath,json,ldap,gmp,opcache,redis,imagick,pear} -y

# Configurer PHP :

    sudo sed -ri 's!upload_max_filesize = .+!upload_max_filesize = 10M!g' /etc/php/*/fpm/php.ini
    sudo sed -ri 's!post_max_size = .+!post_max_size = 20M!g' /etc/php/*/fpm/php.ini
    sudo sed -ri 's!memory_limit = .+!memory_limit = 512M!g' /etc/php/*/fpm/php.ini

## Et redémarrer le service PHP-FPM :

    sudo systemctl restart phpX.X-fpm
# (Remplacez X.X par la version spécifique de PHP.)

# Installer Git et Composer :

    sudo apt install git composer -y

# Partie 2 : Configuration de GlusterFS
# Formatez la partition /dev/nvme1n1 en ext4 et montez-la en tant que /mnt/brick1 sur chaque serveur.

# Serveur 1 et Serveur 2
# Formatter la partition :

    sudo mkfs.ext4 /dev/nvme1n1

# Monter la partition :

    sudo mkdir /mnt/brick1
    sudo mount /dev/nvme1n1 /mnt/brick1

# Autoriser le trafic TCP entrant dans le port 24007 (GlusterFS) :

    sudo iptables -A INPUT -p tcp --dport 24007 -j ACCEPT
    sudo service iptables-persistent save
    sudo service iptables-persistent reload

# Initialiser les volumes distants sur chaque serveur :

    sudo gluster peer probe Srv2-D12
    sudo gluster peer probe Srv1-D12

# Vérifier les pairs existants :

    sudo gluster peer status

# Créer un volume répliqué appelé gvol1 composé de deux bricks, un sur chaque serveur :

    sudo gluster volume create gvol1 transport tcp Srv1-D12:/mnt/brick1 Srv2-D12:/mnt/brick1

    Activer le volume :

    sudo gluster volume start gvol1

# Partie 3 : Installation et configuration de BookStack
# Cloner le repository officiel de BookStack et importer la structure de la base de données.

# Serveur 1 et Serveur 2
# Clonez le repo officiel de BookStack :

    cd /var/www/html
    sudo git clone https://github.com/BookStackApp/BookStack.git

# Importez la structure de la base de données :

    wget https://raw.githubusercontent.com/BookStackApp/BookStack/main/structure.sql
    sudo mysql -u root -p &lt; structure.sql

# Entrez le mot de passe root lorsque prompté.
# Accorder les droits au script d'initialisation :

    sudo chmod o+x /var/www/html/BookStack/artisan

# Exécuter le script d'initialisation de BookStack :

    sudo -u www-data php /var/www/html/BookStack/artisan migrate --seed
    
# Prenez note des valeurs renvoyées par l'interface en ligne de commande. Vous en aurez besoin ultérieurement.
# Partie 4 : Relier le volume GlusterFS à BookStack
# Reliez le volume GlusterFS à l'application BookStack.
# Serveur 1 et Serveur 2

# Arrêtez les services Apache et PHP-FPM :

    sudo systemctl stop apache2
    sudo systemctl stop phpX.X-fpm

# (Remplacez X.X par la version spécifique de PHP.)

# Démontez le répertoire existant de BookStack :

    sudo umount /var/www/html/BookStack/data

# Montez le volume GlusterFS dans le chemin d'accès prévu pour BookStack :

    sudo mount -t glusterfs Srv1-D12:/gvol1/BookStack/data /var/www/html/BookStack/data

# Remarque : Lors du montage, vous devez spécifier le chemin absolu du répertoire BookStack/data dans le volume GlusterFS (/gvol1/BookStack/data) plutôt que le volume lui-même (/gvol1).

# Recopiez le contenu existant (si applicable) du répertoire /var/www/html/BookStack/data dans le volume GlusterFS :

    sudo cp -rp /var/www/html/BookStack/data/* /gvol1/BookStack/data

# Redémarrez les services Apache et PHP-FPM :

    sudo systemctl start apache2
    sudo systemctl start phpX.X-fpm

# Partie 5 : Configuration de NGINX en tant que reverse proxy
# Installez NGINX et configurez-le comme reverse proxy pour exposer BookStack derrière une adresse IP virtuelle ou un nom de domaine.
# Serveur 1 et Serveur 2

# Installer NGINX :

    sudo apt install nginx -y

# Écrivez un bloc NGINX personnalisé pour servir des demandes HTTP provenant de l'extérieur :

    sudo tee /etc/nginx/conf.d/bookstack.conf &lt;&lt;EOF
    server {
        listen 80;
        return 301 https://\$host\$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name mydomain.fr;

        ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

        location / {
            proxy_pass http://localhost:8000;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }
    EOF
# Adjustez le nom de domaine et les chemins d'accès aux certificats SSL selon votre configuration.

# Activez et démarrez le service NGINX :

    sudo systemctl enable nginx
    sudo systemctl start nginx



















