# 🐳 Odoocker — Déploiement Odoo Docker sans prise de tête

Mettre en place Odoo avec Docker from scratch, c'est environ 3h de documentation éparpillée,
des erreurs Nginx cryptiques, et un certificat SSL qui expire au mauvais moment.

Odoocker règle tout ça : un seul `.env` à configurer, et vous avez une instance Odoo
Community ou Enterprise en production avec HTTPS automatique.

---

## Ce que vous obtenez

```
  docker-compose up
        │
        ├──► 🟢 Odoo (port 8069 interne)
        │         │ addons Community
        │         │ addons Enterprise (si configuré)
        │         │ addons Custom (vos modules)
        │         │
        │    ┌────▼──────────────────────────┐
        │    │   Dossiers montés en volume   │
        │    │   /mnt/extra-addons/          │
        │    │   /var/lib/odoo/filestore/    │
        │    └───────────────────────────────┘
        │
        ├──► 🐘 PostgreSQL (port 5432 interne)
        │         │ données persistantes
        │         │ volume nommé Docker
        │
        └──► 🔒 Nginx (ports 80 + 443 publics)
                  │ reverse proxy → Odoo :8069
                  │ reverse proxy longpolling → :8072
                  │ SSL Let's Encrypt auto-renew
                  └── certbot (renouvellement auto)
```

---

## Configuration en 5 minutes

```bash
# 1. Cloner le repo
git clone https://github.com/elouafi-abderrahmane-2002/odoocker.git
cd odoocker

# 2. Configurer l'environnement
cp .env.example .env
nano .env
```

Les variables importantes dans `.env` :

```bash
ODOO_VERSION=16.0
IS_ENTERPRISE=false          # true si vous avez la licence Enterprise
DOMAIN=odoo.mondomaine.com   # votre domaine (pour SSL Let's Encrypt)
ADMIN_EMAIL=admin@email.com  # email pour les alertes SSL
DB_PASSWORD=supersecretpwd
ODOO_MASTER_PASSWORD=masterpass
ENABLE_SSL=true              # génère et renouvelle le cert auto
```

```bash
# 3. Démarrer tout
docker-compose up -d

# Et voilà. Odoo accessible sur https://odoo.mondomaine.com
```

---

## Commandes utiles

```bash
# Déploiement complet (pull + rebuild + restart)
docker-compose down && git pull && \
docker-compose pull && \
docker-compose build --no-cache && \
docker-compose up -d && \
docker-compose logs -f odoo

# Redémarrage rapide (sans rebuild)
docker-compose down && git pull && \
docker-compose up -d --build && \
docker-compose logs -f --tail 2000 odoo

# Suivre les logs Odoo en temps réel
docker-compose logs -f --tail 2000 odoo

# Accéder au shell du container Odoo
docker exec -it odoo bash

# Backup de la base de données
docker exec db pg_dump -U odoo mydb > backup_$(date +%Y%m%d).sql
```

---

## Structure du projet

```
odoocker/
│
├── .env.example              ← template de config à copier
├── docker-compose.yml        ← services: odoo, db, nginx, certbot
├── Dockerfile                ← image Odoo custom avec vos dépendances
├── config/
│   └── odoo.conf             ← paramètres Odoo (workers, proxy mode...)
├── nginx/
│   ├── nginx.conf            ← reverse proxy + longpolling :8072
│   └── ssl.conf              ← config HTTPS + renouvellement certbot
├── addons/
│   └── custom/               ← vos modules Odoo custom ici
└── scripts/
    ├── init.sh               ← initialisation base de données
    └── backup.sh             ← script de backup automatisé
```

---

## Pourquoi Odoo en mode production c'est plus compliqué qu'on croit

Deux choses m'ont surpris en mettant ça en place :

**Le longpolling.** Odoo utilise un port séparé (`:8072`) pour les notifications temps réel
(messagerie, activités). Si vous ne configurez pas le reverse proxy pour rediriger
`/longpolling` vers ce port, l'interface Odoo semble lente et les notifs ne marchent pas.

**Le `proxy_mode`.** Quand Odoo est derrière Nginx, il faut activer `proxy_mode = True`
dans `odoo.conf`, sinon Odoo génère des URLs en HTTP même quand le site est en HTTPS.
Ça casse les redirections OAuth, les emails avec liens, et les QR codes. Petit paramètre,
gros impact.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
