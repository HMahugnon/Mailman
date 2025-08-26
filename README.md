# Documentation complète - Installation Mailman 3 avec Docker

## Table des matières

1. [Vue d'ensemble](#vue-d'ensemble)
2. [Prérequis](#prérequis)
3. [Architecture](#architecture)
4. [Installation](#installation)
5. [Configuration](#configuration)
6. [Déploiement](#déploiement)
7. [Post-installation](#post-installation)
8. [Sécurisation SSL](#sécurisation-ssl)
9. [Administration](#administration)
10. [Dépannage](#dépannage)
11. [Maintenance](#maintenance)
12. [Sauvegarde](#sauvegarde)

## Vue d'ensemble

Cette documentation décrit l'installation complète de Mailman 3 avec Docker, incluant :

- **Mailman Core** : Moteur de gestion des listes de diffusion
- **Mailman Web (Postorius)** : Interface web d'administration moderne
- **HyperKitty** : Archives consultables des emails
- **PostgreSQL** : Base de données relationnelle
- **Nginx** : Serveur web reverse proxy
- **Postfix** : Serveur SMTP pour l'envoi d'emails

## Prérequis

### Système requis

- **OS** : Ubuntu 20.04+ / Debian 11+ / CentOS 8+ (recommandé : Ubuntu 22.04 LTS)
- **RAM** : Minimum 2 GB, recommandé 4 GB
- **Stockage** : Minimum 20 GB d'espace libre
- **Réseau** : IP publique fixe

### Logiciels requis

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Docker
sudo apt install -y docker.io docker-compose-plugin

# Démarrer et activer Docker
sudo systemctl start docker
sudo systemctl enable docker

# Ajouter l'utilisateur au groupe docker (optionnel)
sudo usermod -aG docker $USER
# Puis se déconnecter/reconnecter ou : newgrp docker
```

### Configuration réseau

- **Domaine** : Nom de domaine pointant vers votre serveur (ex: `lists.mondomaine.com`)
- **Ports requis** : 25, 80, 443, 587 (SMTP), 993, 995 (IMAP/POP3 optionnels)
- **Pare-feu** : Ports ouverts sur le serveur et chez le fournisseur cloud

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Nginx (Port 80/443)                  │
│                    Reverse Proxy + SSL                      │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                  Mailman Web (Port 8000)                    │
│              Interface d'administration                      │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                 Mailman Core (Port 8001)                    │
│              Moteur de listes + API REST                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│               PostgreSQL (Port 5432)                        │
│                    Base de données                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 Postfix (Ports 25/587)                      │
│                    Serveur SMTP                             │
└─────────────────────────────────────────────────────────────┘
```

## Installation

### 1. Préparation de l'environnement

```bash
# Créer le répertoire du projet
mkdir mailman3-docker && cd mailman3-docker

# Créer la structure des dossiers
mkdir -p data/{core,web,database,nginx/ssl,postfix,certbot} nginx

# Vérifier la structure
tree -L 3 .
```

### 2. Configuration des variables d'environnement

Créer le fichier `.env` :

```bash
cat > .env << 'EOF'
# ===========================================
# CONFIGURATION MAILMAN 3 - DOCKER
# ===========================================

# Domaine principal (MODIFIER OBLIGATOIREMENT)
DOMAIN=lists.mondomaine.com

# Configuration administrateur (MODIFIER OBLIGATOIREMENT)
ADMIN_EMAIL=admin@mondomaine.com
ADMIN_PASSWORD=VotreMotDePasseAdminSecurise123!

# Configuration base de données (MODIFIER OBLIGATOIREMENT)
DB_PASSWORD=MotDePassePostgreSQLSecurise456!

# Clés de sécurité (GÉNÉRER DE NOUVELLES CLÉS)
SECRET_KEY=CleDjangoSecrete789RandomStringTresLongue!
HYPERKITTY_API_KEY=CleAPIHyperKitty987RandomStringSecurise!
REST_PASSWORD=MotDePasseRESTAPISecurise654!

# Configuration SMTP (MODIFIER SELON BESOINS)
SMTP_USER=noreply@mondomaine.com
SMTP_PASSWORD=MotDePasseSMTPSecurise321!

# Configuration avancée (généralement pas à modifier)
POSTGRES_USER=mailman
POSTGRES_DB=mailman
MAILMAN_REST_USER=restadmin
DJANGO_ADMIN_USER=admin
EOF
```

### 3. Configuration Docker Compose

Créer le fichier `docker-compose.yml` :

```bash
cat > docker-compose.yml << 'EOF'
services:
  # Base de données PostgreSQL
  database:
    image: postgres:15
    hostname: database
    restart: unless-stopped
    volumes:
      - ./data/database:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    networks:
      - mailman
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Mailman Core - Moteur des listes
  mailman-core:
    image: maxking/mailman-core:0.4
    hostname: mailman-core
    restart: unless-stopped
    volumes:
      - ./data/core:/opt/mailman/
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${DB_PASSWORD}@database/${POSTGRES_DB}
      DATABASE_TYPE: postgres
      DATABASE_CLASS: mailman.database.postgresql.PostgreSQLDatabase
      MTA: postfix
      SMTP_HOST: postfix
      SMTP_PORT: 25
      LMTP_HOST: 0.0.0.0
      LMTP_PORT: 8024
    depends_on:
      database:
        condition: service_healthy
    networks:
      - mailman
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8001/3.1/system/versions || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Mailman Web - Interface d'administration
  mailman-web:
    image: maxking/mailman-web:0.4
    hostname: mailman-web
    restart: unless-stopped
    volumes:
      - ./data/web:/opt/mailman-web-data
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${DB_PASSWORD}@database/${POSTGRES_DB}
      DATABASE_TYPE: postgres
      HYPERKITTY_API_KEY: ${HYPERKITTY_API_KEY}
      SECRET_KEY: ${SECRET_KEY}
      DJANGO_ADMIN_EMAIL: ${ADMIN_EMAIL}
      DJANGO_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      SERVE_FROM_DOMAIN: ${DOMAIN}
      MAILMAN_REST_URL: http://mailman-core:8001
      MAILMAN_REST_USER: ${MAILMAN_REST_USER}
      MAILMAN_REST_PASSWORD: ${REST_PASSWORD}
      MAILMAN_ADMIN_USER: ${DJANGO_ADMIN_USER}
      MAILMAN_ADMIN_EMAIL: ${ADMIN_EMAIL}
      UWSGI_STATIC_MAP: /static=/opt/mailman-web-data/static
    depends_on:
      database:
        condition: service_healthy
      mailman-core:
        condition: service_healthy
    networks:
      - mailman

  # Serveur SMTP Postfix
  postfix:
    image: catatnight/postfix:latest
    hostname: ${DOMAIN}
    restart: unless-stopped
    environment:
      maildomain: ${DOMAIN}
      smtp_user: ${SMTP_USER}:${SMTP_PASSWORD}
    volumes:
      - ./data/postfix:/var/spool/postfix
    ports:
      - "25:25"
      - "587:587"
    networks:
      - mailman

  # Serveur Web Nginx
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./data/nginx/ssl:/etc/nginx/ssl
      - ./data/web/static:/opt/mailman-web-data/static:ro
      - ./data/certbot:/var/www/html
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - mailman-web
    networks:
      - mailman

networks:
  mailman:
    driver: bridge

volumes:
  database_data:
  core_data:
  web_data:
EOF
```

### 4. Configuration Nginx

Créer le fichier `nginx/nginx.conf` :

```bash
cat > nginx/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Configuration de logs
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;

    # Configuration upstream
    upstream mailman {
        server mailman-web:8000;
    }

    # Configuration du serveur HTTP
    server {
        listen 80;
        server_name _;

        # Pour Let's Encrypt
        location /.well-known/acme-challenge/ {
            root /var/www/html;
        }

        # Redirection vers HTTPS (une fois SSL configuré)
        # location / {
        #     return 301 https://$server_name$request_uri;
        # }

        # Configuration temporaire pour HTTP
        location / {
            proxy_pass http://mailman;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_read_timeout 300;
            proxy_connect_timeout 300;
            proxy_send_timeout 300;
        }

        # Fichiers statiques
        location /static/ {
            alias /opt/mailman-web-data/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }

    # Configuration HTTPS (à décommenter après obtention SSL)
    # server {
    #     listen 443 ssl http2;
    #     server_name _;
    #
    #     ssl_certificate /etc/nginx/ssl/live/VOTRE_DOMAINE/fullchain.pem;
    #     ssl_certificate_key /etc/nginx/ssl/live/VOTRE_DOMAINE/privkey.pem;
    #     
    #     ssl_protocols TLSv1.2 TLSv1.3;
    #     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    #     ssl_prefer_server_ciphers on;
    #
    #     client_max_body_size 100M;
    #
    #     location / {
    #         proxy_pass http://mailman;
    #         proxy_set_header Host $host;
    #         proxy_set_header X-Real-IP $remote_addr;
    #         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #         proxy_set_header X-Forwarded-Proto $scheme;
    #     }
    #
    #     location /static/ {
    #         alias /opt/mailman-web-data/static/;
    #         expires 30d;
    #         add_header Cache-Control "public, immutable";
    #     }
    # }
}
EOF
```

## Configuration

### 1. Génération des clés de sécurité

```bash
# Générer des clés aléatoirement
echo "SECRET_KEY=$(openssl rand -base64 32)" 
echo "HYPERKITTY_API_KEY=$(openssl rand -base64 32)"
echo "REST_PASSWORD=$(openssl rand -base64 16)"

# Mettre à jour le fichier .env avec ces nouvelles clés
```

### 2. Configuration DNS

Ajoutez ces enregistrements DNS chez votre registrar :

```dns
# A record - Pointer votre domaine vers l'IP du serveur
lists.mondomaine.com.    IN    A       VOTRE_IP_PUBLIQUE

# MX record - Pour recevoir les emails
mondomaine.com.          IN    MX      10 lists.mondomaine.com.

# Enregistrements de sécurité email (recommandés)
# SPF
mondomaine.com.          IN    TXT     "v=spf1 mx ~all"

# DMARC (optionnel)
_dmarc.mondomaine.com.   IN    TXT     "v=DMARC1; p=none; rua=mailto:admin@mondomaine.com"
```

### 3. Configuration du pare-feu

```bash
# Avec UFW (Ubuntu/Debian)
sudo ufw allow 22      # SSH
sudo ufw allow 25      # SMTP
sudo ufw allow 80      # HTTP
sudo ufw allow 443     # HTTPS
sudo ufw allow 587     # SMTP submission
sudo ufw --force enable

# Vérifier
sudo ufw status

# Avec iptables (alternative)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 25 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 587 -j ACCEPT
```

## Déploiement

### 1. Vérification pré-déploiement

```bash
# Vérifier que Docker fonctionne
sudo docker --version
sudo docker-compose --version

# Tester la configuration
sudo docker-compose config

# Vérifier les fichiers
ls -la
cat .env | grep DOMAIN
```

### 2. Déploiement étape par étape

```bash
# Étape 1 : Démarrer la base de données
sudo docker-compose up -d database
echo "Attente de l'initialisation de la base de données..."
sleep 20

# Vérifier la base de données
sudo docker-compose logs database
sudo docker-compose exec database pg_isready -U mailman

# Étape 2 : Démarrer Mailman Core
sudo docker-compose up -d mailman-core
echo "Attente de l'initialisation de Mailman Core..."
sleep 30

# Vérifier Mailman Core
sudo docker-compose logs mailman-core

# Étape 3 : Démarrer Mailman Web
sudo docker-compose up -d mailman-web
echo "Attente de l'initialisation de Mailman Web..."
sleep 30

# Vérifier Mailman Web
sudo docker-compose logs mailman-web

# Étape 4 : Démarrer Nginx
sudo docker-compose up -d nginx

# Étape 5 (optionnel) : Démarrer Postfix
sudo docker-compose up -d postfix

# Vérifier l'état final
sudo docker-compose ps
```

### 3. Vérification du déploiement

```bash
# Vérifier que tous les services sont UP
sudo docker-compose ps

# Tester la connectivité HTTP
curl -I http://localhost
curl -I http://VOTRE_DOMAINE

# Vérifier les logs
sudo docker-compose logs --tail=50

# Tester l'accès web
# Aller sur http://VOTRE_DOMAINE dans un navigateur
```

## Post-installation

### 1. Initialisation de l'administration Django

```bash
# Migrations de base de données
sudo docker-compose exec mailman-web python manage.py migrate

# Créer un superutilisateur Django
sudo docker-compose exec mailman-web python manage.py createsuperuser
# Entrer : nom d'utilisateur, email, mot de passe

# Collecter les fichiers statiques
sudo docker-compose exec mailman-web python manage.py collectstatic --noinput

# Redémarrer les services pour prendre en compte les changements
sudo docker-compose restart mailman-web nginx
```

### 2. Configuration initiale de Mailman

```bash
# Vérifier l'API REST
sudo docker-compose exec mailman-core mailman info

# Créer un domaine de mail (si nécessaire)
sudo docker-compose exec mailman-core mailman create domain mondomaine.com

# Vérifier les domaines
sudo docker-compose exec mailman-core mailman domains
```

### 3. Test de fonctionnement

```bash
# Accéder aux interfaces :
echo "Interface d'administration : http://${DOMAIN}/admin/"
echo "Interface utilisateur : http://${DOMAIN}/"
echo "Archives : http://${DOMAIN}/hyperkitty/"

# Créer une liste de test via l'interface web ou en ligne de commande :
sudo docker-compose exec mailman-core mailman create test@mondomaine.com
```

## Sécurisation SSL

### 1. Obtenir les certificats SSL avec Let's Encrypt

```bash
# Méthode 1 : Avec un conteneur Certbot
sudo docker run --rm \
  -v $(pwd)/data/certbot:/var/www/html \
  -v $(pwd)/data/nginx/ssl:/etc/letsencrypt \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/html \
  --email ${ADMIN_EMAIL} \
  --agree-tos \
  --no-eff-email \
  -d ${DOMAIN}

# Méthode 2 : Installation native de Certbot
sudo apt install snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Obtenir le certificat
sudo certbot certonly --webroot -w $(pwd)/data/certbot -d ${DOMAIN}
```

### 2. Configuration Nginx avec SSL

```bash
# Sauvegarder la configuration actuelle
cp nginx/nginx.conf nginx/nginx.conf.bak

# Mettre à jour nginx.conf pour SSL
cat > nginx/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    upstream mailman {
        server mailman-web:8000;
    }

    # Redirection HTTP vers HTTPS
    server {
        listen 80;
        server_name _;
        
        location /.well-known/acme-challenge/ {
            root /var/www/html;
        }
        
        location / {
            return 301 https://$server_name$request_uri;
        }
    }

    # Configuration HTTPS
    server {
        listen 443 ssl http2;
        server_name _;

        # Certificats SSL
        ssl_certificate /etc/nginx/ssl/live/VOTRE_DOMAINE/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/live/VOTRE_DOMAINE/privkey.pem;
        
        # Configuration SSL moderne
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # Headers de sécurité
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
        add_header X-Content-Type-Options nosniff;
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";

        client_max_body_size 100M;

        # Proxy vers Mailman Web
        location / {
            proxy_pass http://mailman;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_read_timeout 300;
            proxy_connect_timeout 300;
            proxy_send_timeout 300;
        }

        # Fichiers statiques
        location /static/ {
            alias /opt/mailman-web-data/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }
}
EOF

# Remplacer VOTRE_DOMAINE par votre vrai domaine
sed -i 's/VOTRE_DOMAINE/lists.mondomaine.com/g' nginx/nginx.conf
```

### 3. Redémarrer avec SSL

```bash
# Redémarrer Nginx avec la nouvelle configuration
sudo docker-compose restart nginx

# Vérifier les logs
sudo docker-compose logs nginx

# Tester HTTPS
curl -I https://VOTRE_DOMAINE
```

### 4. Renouvellement automatique SSL

```bash
# Créer un script de renouvellement
cat > renew-ssl.sh << 'EOF'
#!/bin/bash
docker run --rm \
  -v $(pwd)/data/certbot:/var/www/html \
  -v $(pwd)/data/nginx/ssl:/etc/letsencrypt \
  certbot/certbot renew --webroot --webroot-path=/var/www/html

# Redémarrer Nginx si les certificats ont été renouvelés
docker-compose restart nginx
EOF

chmod +x renew-ssl.sh

# Ajouter une tâche cron pour le renouvellement automatique
echo "0 3 * * * $(pwd)/renew-ssl.sh" | sudo crontab -
```

## Administration

### 1. Interfaces d'administration

| Interface | URL | Description |
|-----------|-----|-------------|
| **Postorius** | `https://DOMAIN/` | Interface principale des listes |
| **HyperKitty** | `https://DOMAIN/hyperkitty/` | Archives des emails |
| **Django Admin** | `https://DOMAIN/admin/` | Administration système |

### 2. Gestion des listes via interface web

1. **Créer une liste** :
   - Aller sur `https://DOMAIN/`
   - Se connecter avec le compte admin
   - Cliquer sur "Create New List"
   - Remplir les informations

2. **Configuration de liste** :
   - Paramètres d'abonnement
   - Modération des messages
   - Archives publiques/privées
   - Templates personnalisés

### 3. Gestion en ligne de commande

```bash
# Lister toutes les listes
sudo docker-compose exec mailman-core mailman lists

# Créer une liste
sudo docker-compose exec mailman-core mailman create liste@mondomaine.com

# Supprimer une liste
sudo docker-compose exec mailman-core mailman remove liste@mondomaine.com

# Ajouter un membre à une liste
sudo docker-compose exec mailman-core mailman addmembers -r member liste@mondomaine.com user@example.com

# Lister les membres d'une liste
sudo docker-compose exec mailman-core mailman members liste@mondomaine.com

# Informations système
sudo docker-compose exec mailman-core mailman info
sudo docker-compose exec mailman-core mailman status
```

### 4. Gestion des utilisateurs

```bash
# Créer un utilisateur Django
sudo docker-compose exec mailman-web python manage.py createsuperuser

# Lister les utilisateurs Django
sudo docker-compose exec mailman-web python manage.py shell -c "from django.contrib.auth.models import User; print([u.username for u in User.objects.all()])"
```

## Dépannage

### 1. Problèmes courants

#### Interface inaccessible

```bash
# Vérifier l'état des conteneurs
sudo docker-compose ps

# Vérifier les logs
sudo docker-compose logs nginx
sudo docker-compose logs mailman-web

# Redémarrer les services
sudo docker-compose restart
```

#### Erreur 401 REST API

```bash
# Vérifier les variables d'environnement
sudo docker-compose exec mailman-web env | grep MAILMAN
sudo docker-compose exec mailman-core env | grep DATABASE

# Redémarrer avec recréation des conteneurs
sudo docker-compose down
sudo docker-compose up -d
```

#### Problèmes de base de données

```bash
# Vérifier la connectivité PostgreSQL
sudo docker-compose exec database pg_isready -U mailman

# Se connecter à la base de données
sudo docker-compose exec database psql -U mailman -d mailman

# Migrations manuelles si nécessaire
sudo docker-compose exec mailman-web python manage.py migrate --run-syncdb
```

#### Problèmes d'envoi d'emails

```bash
# Vérifier Postfix
sudo docker-compose logs postfix

# Tester l'envoi SMTP
sudo docker-compose exec postfix telnet localhost 25

# Vérifier la configuration DNS
dig MX mondomaine.com
dig TXT mondomaine.com
```

### 2. Commandes de diagnostic

```bash
# État complet du système
sudo docker-compose ps
sudo docker stats --no-stream
df -h

# Logs détaillés
sudo docker-compose logs --tail=100 -f

# Vérification réseau
sudo docker network ls
sudo docker network inspect mailman3-docker_mailman

# Tests de connectivité
curl -v http://localhost
curl -v https://VOTRE_DOMAINE
telnet VOTRE_DOMAINE 25
```

### 3. Reset complet (en cas de problème majeur)

```bash
# ⚠️ ATTENTION : Ceci supprime TOUTES les données
sudo docker-compose down -v
sudo rm -rf data/
mkdir -p data/{core,web,database,nginx/ssl,postfix,certbot}
sudo docker-compose up -d

# Puis refaire la configuration post-installation
```

## Maintenance

### 1. Mises à jour

```bash
# Sauvegarder avant mise à jour
./backup.sh

# Mettre à jour les images Docker
sudo docker-compose pull

# Redémarrer avec les nouvelles images
sudo docker-compose up -d

# Vérifier que tout fonctionne
sudo docker-compose ps
```

### 2. Monitoring

#### Script de monitoring basique

```bash
cat > monitor.sh << 'EOF'
#!/bin/bash
echo "=== État des services Mailman ==="
docker-compose ps

echo -e "\n=== Utilisation des ressources ==="
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

echo -e "\n=== Espace disque ==="
df -h /home/ubuntu/mailman3-docker/data

echo -e "\n=== Test HTTP ==="
curl -s -o /dev/null -w "%{http_code}" http://localhost || echo "ERREUR HTTP"

echo -e "\n=== Dernières erreurs Nginx ==="
docker-compose logs --tail=5 nginx | grep -i error
EOF

chmod +x monitor.sh
```

#### Monitoring automatique

```bash
# Ajouter une tâche cron pour monitoring
echo "*/15 * * * * $(pwd)/monitor.sh >> $(pwd)/monitoring.log 2>&1" | crontab -

# Script d'alerte simple
cat > alert.sh << 'EOF'
#!/bin/bash
if ! curl -s http://localhost > /dev/null; then
    echo "ALERTE: Mailman inaccessible" | mail -s "Mailman DOWN" admin@mondomaine.com
fi
EOF

chmod +x alert.sh
echo "*/5 * * * * $(pwd)/alert.sh" | crontab -
```

### 3. Optimisation des performances

```bash
# Nettoyer les logs Docker
sudo docker system prune -f

# Optimiser PostgreSQL
sudo docker-compose exec database psql -U mailman -d mailman -c "VACUUM ANALYZE;"

# Nettoyer les archives anciennes (si configuré)
sudo docker-compose exec mailman-web python manage.py runjob daily
```

## Sauvegarde

### 1. Script de sauvegarde complet

```bash
cat > backup.sh << 'EOF'
#!/bin/bash

# Configuration
BACKUP_DIR="/backup/mailman"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Créer le répertoire de sauvegarde
mkdir -p $BACKUP_DIR

echo "Démarrage de la sauvegarde Mailman - $DATE"

# 1. Sauvegarde de la base de données
echo "Sauvegarde de la base de données..."
docker-compose exec -T database pg_dump -U mailman -c mailman > $BACKUP_DIR/database_$DATE.sql
if [ $? -eq 0 ]; then
    echo "✓ Base de données sauvegardée"
    gzip $BACKUP_DIR/database_$DATE.sql
else
    echo "✗ Erreur sauvegarde base de données"
fi

# 2. Sauvegarde des données Mailman
echo "Sauvegarde des données Mail
