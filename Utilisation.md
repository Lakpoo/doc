#  Guide d’utilisation des différents services - Groupe 4

## Table des Matières
1. [Architecture Générale](#architecture-générale)
2. [Configuration Nginx](#configuration-nginx)
3. [Configuration React](#configuration-react)
4. [Configuration Symfony](#configuration-symfony)
5. [Configuration PHPMyAdmin](#configuration-phpmyadmin)
6. [Sécurité](#sécurité)
7. [Gestion des Logs](#gestion-des-logs)
8. [Maintenance](#maintenance)

## Architecture Générale

### Structure des Dossiers
```
/var/www/groupe-4/
├── react/
│   ├── dist/      # Build React
│   └── src/       # Code source React
└── symfony/
    └── public/    # Root Symfony
```

### Services et Ports
| Service     | Port | Protocol | URL                                        |
|-------------|------|----------|-------------------------------------------|
| HTTP        | 80   | HTTP     | http://groupe-4.lycee-stvincent.net       |
| HTTPS       | 443  | HTTPS    | https://groupe-4.lycee-stvincent.net      |
| PHPMyAdmin  | 82   | HTTPS    | https://groupe-4.lycee-stvincent.net:82   |
| SSH         | 22   | SSH      | ssh user@groupe-4.lycee-stvincent.net     |

## Configuration Nginx

### Configuration Principale
```nginx
server {
    server_name groupe-4.lycee-stvincent.net;

    location /symfony {
        root /var/www/groupe-4/symfony/public;
        index index.php;
        try_files $uri /index.php$is_args$args;
    }

    location ~ \.php$ {
        root /var/www/groupe-4/symfony/public;
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location /react {
        alias /var/www/groupe-4/react/dist;  # Changé de root à alias et retiré le slash final
        index index.html;
        try_files $uri $uri/ /react/index.html;  # Corrigé le try_files
    }

    error_log /var/log/nginx/groupe-4-error.log;
    access_log /var/log/nginx/groupe-4-access.log;
    
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/groupe-4.lycee-stvincent.net/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/groupe-4.lycee-stvincent.net/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = groupe-4.lycee-stvincent.net) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name groupe-4.lycee-stvincent.net;
    return 404; # managed by Certbot


}
```

## Configuration React

### vite.config.js
```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import fs from 'fs'

// Création du dossier de logs s'il n'existe pas
const logDir = '/var/log/react'
if (!fs.existsSync(logDir)) {
    fs.mkdirSync(logDir, { recursive: true })
}

export default defineConfig({
    plugins: [react()],
    server: {
        port: 3001,
        logger: {
            level: 'info',
            prefix: '[React App]',
            dest: '/var/log/react/react-app.log'
        }
    },
    build: {
        rollupOptions: {
            onwarn(warning, warn) {
                // Log les avertissements de build
                fs.appendFileSync('/var/log/react/build.log', ${new Date().toISOString()} - ${warning.message}\n)
                warn(warning)
            }
        }
    },
    base: "/react/"
})
```

### Commandes de Build
```bash
cd /var/www/groupe-4/react
npm run build
```

## Configuration Symfony
Symfony est configuré pour fonctionner avec PHP 8.3 et est accessible via le préfixe `/symfony`.

### PHP-FPM Configuration
- Version: PHP 8.3
- Socket: `/var/run/php/php8.3-fpm.sock`

## Configuration PHPMyAdmin

```nginx
server {
    listen 82 ssl default_server;
    listen [::]:82 ssl default_server;

    server_name groupe-4.lycee-stvincent.net;

    ssl_certificate /etc/letsencrypt/live/groupe-4.lycee-stvincent.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/groupe-4.lycee-stvincent.net/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /usr/share/phpmyadmin;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    error_log /var/log/nginx/phpmyadmin-error.log;
    access_log /var/log/nginx/phpmyadmin-access.log;
}
```

## Sécurité

### SSL/TLS
- Certificats Let's Encrypt
- Renouvellement automatique via certbot
- Configuration SSL optimisée

### Fail2ban
Configuration (/etc/fail2ban/jail.d/custom.conf):
```ini
[DEFAULT]
bantime = 300
findtime = 300
maxretry = 3

[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
```

## Gestion des Logs

### Structure des Logs
- Nginx: `/var/log/nginx/groupe-4-*.log`
- React: `/var/log/react/react-app.log`
- PHP-FPM: `/var/log/php8.3-fpm.log`

### Rotation des Logs
Configuration (/etc/logrotate.d/react-app):
```conf
/var/log/react/*.log {
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
```