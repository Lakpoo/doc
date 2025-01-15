## Guide d'Installation complète - Groupe 4

### 1. Modification des bases

```bash
# Mise à jour du nom du serveur
sudo nano /etc/hosts
On y ajoute : 127.0.0.1 localhost
sudo nano /etc/hostname
Personalisation du hostname en : groupe-3

# Preserver le nom du host sur OVH
sudo nano /etc/cloud/cloud.cfg
preserve_hostname: true

sudo reboot
pour appliquer les modifications
```

### 2. Installation de PHP 

```bash
# Installation de PHP
sudo apt install gnupg
sudo curl -o- https://packages.sury.org/php/README.txt | bash
sudo apt update
PHP_VERSION="8.4"
sudo apt install php$PHP_VERSION php$PHP_VERSION-cli php$PHP_VERSION-fpm
sudo apt remove apache2
sudo apt autoremove
```

### 3. Création de user et de dossier

```bash
# Création et modification de fichier
sudo adduser website
sudo mkdir -p /data/website
sudo chown website:website /data/website
sudo apt install git
exit
ssh website@51.91.108.98
touch /data/website/index.php
echo "<?php
echo "Hello world";
" > /data/website/index.php
exit

ssh debian@51.91.108.98
usermod -aG sudo website
sudo apt remove apache2
sudo apt autoremove
sudo apt install nginx
sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.backup
sudo sh -c 'echo "server {
listen 80 default_server;
listen [::]:80 default_server;
root /data/website;
index index.php;
servername ;
location / {
First attempt to serve request as file, then
as directory, then fall back to displaying a 404.
try_files $uri $uri/ =404;
}
pass PHP scripts to FastCGI server
location ~ \.php$ {
include snippets/fastcgi-php.conf;
fastcgi_pass unix:/run/php/php8.2-fpm.sock;
}
location ~ /\.ht {
deny all;
}
}" > /etc/nginx/sites-available/default'

#Verification de la conformite de la config
sudo nginx -t
sudo service nginx restart
```

### 4. Configuration de Phpmyadmin

```bash
# Configuration
sudo echo "server {
listen 82 default_server;
listen [::]:82 default_server;
root /usr/share/phpmyadmin;
Add index.php to the list if you are using PHP
index index.php;
server_name groupe-3.lycee-stvincent.net;
location / {
First attempt to serve request as file, then
as directory, then fall back to displaying a 404.
try_files $uri $uri/ =404;
}
pass PHP scripts to FastCGI server
location ~ .php$ {
include snippets/fastcgi-php.conf;
fastcgi_pass unix:/run/php/php8.1-fpm.sock;
}
location ~ /.ht {
deny all;
}
}
" > /etc/nginx/sites-available/phpMyAdmin
sudo ln -s /etc/nginx/sites-available/phpMyAdmin /etc/nginx/sites-enabled/phpMyAdmin

#Relancer le service 

sudo service nginx restart

```

### 5. Installation de Composer

```bash
cd /tmp
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '55ce33d7678c5a611085589f1f3ddf8b3c52d662cd01d4ba75c0ee0
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer

Vérifions la bonne installation de composer :
composer -v 
```

### 6. Configuration UFW

```bash
# Installation de UFW
Autoriser SSH (important pour ne pas perdre l'accès)
sudo ufw allow 22/tcp

Autoriser HTTP et HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

Autoriser PHPMyAdmin (port 82)
sudo ufw allow 82/tcp

Bloquer tout le reste en entrée
sudo ufw default deny incoming

Autoriser toutes les connexions sortantes
sudo ufw default allow outgoing
```

### 7. Installation de logrotate

```bash
#Utilisation de logrotate pour la journalisation des logs

sudo apt update
sudo apt install logrotate
sudo mkdir -p /var/log/react
sudo touch /var/log/react/react-app.log
Définir les bonnes permissions :
sudo chown -R www-data:www-data /var/log/react
sudo chmod -R 644 /var/log/react/react-app.log
sudo chmod 755 /var/log/react
sudo nano /etc/logrotate.d/react-app
Ajoutez la configuration :

bashCopy/var/log/react/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 www-data www-data
    postrotate
        systemctl reload nginx
    endscript
}
Testez la configuration :

bashCopysudo logrotate -d /etc/logrotate.d/react-app
```