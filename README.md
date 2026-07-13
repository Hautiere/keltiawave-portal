# KeltiaWave Portal

Portail statique KeltiaWave pour presenter les outils numeriques autour des langues brittoniques :

- corpus vocal breton ;
- transcription audio/video ;
- enregistrement vocal ;
- futur studio de sous-titres ;
- ouverture progressive au gallois et au cornique.

Ce dossier est volontairement autonome pour pouvoir etre copie dans un projet VSCode separe, par exemple `keltiawave-portal`, sans dependre du projet `corpus-collaboratif`.

## Structure

```text
portal-standalone/
├── index.html
├── README.md
└── assets/
    ├── coast-brittany.jpg
    ├── fanch-avatar-bzh-01.webp
    ├── fanch-avatar-cymru-rugby.webp
    ├── fanch-avatar-hoodie.webp
    ├── flag-brittany.png
    ├── flag-cornwall.svg
    └── flag-wales.svg
```

La page `index.html` contient son CSS inline. Les images sont referencees avec des chemins relatifs :

```text
assets/...
```

Elle peut donc etre deplacee telle quelle dans un autre depot.

## Liens publics utilises

Le portail pointe vers les services publics suivants :

```text
https://keltiawave.com/
  -> portail KeltiaWave

https://corpus.keltiawave.com/
  -> application corpus collaboratif

https://transcription.keltiawave.com/
  -> Brittonic Speech Studio / Breizh Transcriptor

https://transcription.keltiawave.com/record
  -> enregistrement vocal
```

## Preview locale

Ouverture directe :

```text
portal-standalone/index.html
```

Ou avec un serveur statique depuis ce dossier :

```bash
cd portal-standalone
python3 -m http.server 8080
```

Puis ouvrir :

```text
http://localhost:8080/
```

## Deploiement OVH actuel

Etat actuel documente dans le projet corpus :

```text
VPS: vps-dc75d8a6.vps.ovh.net
SSH: ssh ubuntu@vps-dc75d8a6.vps.ovh.net
Projet corpus: /home/ubuntu/apps/corpus-collaboratif
Projet transcripteur: /home/ubuntu/apps/breizh-transcriptor-whisper
```

Le routage public actif est aujourd'hui porte par le Caddy du projet corpus :

```text
/home/ubuntu/apps/corpus-collaboratif/deploy/ovh/Caddyfile
```

Il route actuellement :

```text
keltiawave.com                  -> frontend corpus, avec redirection / vers /portal/
corpus.keltiawave.com           -> frontend corpus
transcription.keltiawave.com    -> app transcripteur
```

Important : ne pas lancer un second Caddy qui ecoute directement les ports `80` et `443` tant que le Caddy du projet corpus les possede deja.

## Deploiement OVH recommande pour projet separe

Objectif : deployer le portail comme projet Docker separe, puis faire router `keltiawave.com` vers ce conteneur depuis le Caddy public.

### 1. Creer le projet sur le VPS

```bash
ssh ubuntu@vps-dc75d8a6.vps.ovh.net
mkdir -p ~/apps/keltiawave-portal
cd ~/apps/keltiawave-portal
```

Copier dans ce dossier le contenu du projet portail :

```text
index.html
assets/
README.md
```

Si le projet est versionne dans Git :

```bash
git clone <url-du-repo-keltiawave-portal> ~/apps/keltiawave-portal
cd ~/apps/keltiawave-portal
```

### 2. Ajouter un Dockerfile simple

Dans le futur projet portail, creer :

```Dockerfile
FROM nginx:1.27-alpine
COPY . /usr/share/nginx/html
```

### 3. Ajouter un docker-compose sans Caddy public

Creer `docker-compose.yml` :

```yaml
name: keltiawave-portal

services:
  portal:
    container_name: keltiawave-portal
    build: .
    restart: unless-stopped
    expose:
      - "80"
    networks:
      - ovh_proxy

networks:
  ovh_proxy:
    external: true
    name: ovh_default
```

Le reseau `ovh_default` correspond au reseau externe utilise par les services OVH existants.

### 4. Lancer le portail

```bash
docker compose up -d --build
docker compose ps
```

### 5. Adapter le Caddy public

Tant que le Caddy public reste dans le projet corpus, modifier :

```text
/home/ubuntu/apps/corpus-collaboratif/deploy/ovh/Caddyfile
```

Remplacer le bloc `keltiawave.com` pour router vers le conteneur portail :

```caddy
keltiawave.com {
	encode zstd gzip
	log {
		output file /var/log/caddy/keltiawave-access.json {
			roll_size 10MiB
			roll_keep 10
			roll_keep_for 720h
		}
		format filter {
			request>remote_ip ip_mask 16 32
			request>client_ip ip_mask 16 32
		}
	}

	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "strict-origin-when-cross-origin"
	}

	@blocked_scans path /.env /.env.* /.git /.git/* /wp-admin /wp-admin/* /wp-login.php
	respond @blocked_scans 404

	reverse_proxy keltiawave-portal:80
}
```

Puis recreer seulement Caddy :

```bash
cd ~/apps/corpus-collaboratif
docker compose --env-file deploy/ovh/.env.ovh -f deploy/ovh/docker-compose.yml up -d --force-recreate caddy
```

Ne pas utiliser `--remove-orphans` sans verification, car le transcripteur peut etre vu comme conteneur externe/orphelin.

### 6. Verifier

```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://keltiawave.com/
curl -sS -o /dev/null -w "%{http_code}\n" https://corpus.keltiawave.com/
curl -sS -o /dev/null -w "%{http_code}\n" https://transcription.keltiawave.com/
curl -sS -o /dev/null -w "%{http_code}\n" https://transcription.keltiawave.com/record
```

Resultats attendus :

```text
keltiawave.com                  -> 200
corpus.keltiawave.com           -> 200
transcription.keltiawave.com    -> 200
transcription.../record         -> 200
```

## Strategie cible plus propre

A terme, le plus propre est de creer un projet dedie `keltiawave-gateway`, seul proprietaire des ports `80/443`.

Architecture cible :

```text
keltiawave-gateway
  -> Caddy public
  -> route keltiawave.com vers keltiawave-portal
  -> route corpus.keltiawave.com vers corpus-frontend
  -> route transcription.keltiawave.com vers transcriptor app

keltiawave-portal
  -> nginx statique

corpus-collaboratif
  -> frontend/backend/postgres/minio sans Caddy public

breizh-transcriptor-whisper
  -> app transcripteur sans Caddy public
```

Dans cette cible, le projet corpus doit etre lance avec sa variante sans Caddy public :

```bash
docker compose --env-file deploy/ovh/.env.ovh -f deploy/ovh/docker-compose.no-caddy.yml up -d --build
```

Le portail peut alors evoluer et etre redeploye sans rebuild du corpus.

## Mise a jour du portail apres modification

Dans le projet portail separe :

```bash
git pull --ff-only
docker compose up -d --build portal
```

Si seuls `index.html` ou `assets/` changent, aucun service corpus/backend/minio/postgres n'a besoin d'etre relance.

## Points d'attention

- Garder les assets locaux dans `assets/`.
- Eviter les chemins absolus du type `/portal/index.html` si le portail devient la racine de `keltiawave.com`.
- Ne pas exposer de fichier `.env`, `.git` ou archive de travail dans le dossier servi par nginx.
- Conserver les domaines publics :
  - `keltiawave.com` pour le portail ;
  - `corpus.keltiawave.com` pour le corpus ;
  - `transcription.keltiawave.com` pour le transcripteur.
- Ne pas lancer plusieurs reverse proxies sur les ports `80/443` du VPS.
