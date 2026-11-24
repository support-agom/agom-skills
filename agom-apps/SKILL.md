---
name: agom-apps
description: Génère ou adapte une application web pour qu'elle se déploie telle quelle sur AgomApps (https://agom.app), l'hébergement d'applications web d'Agom, en ajoutant à la racine du projet un fichier agomapps.yaml et, si besoin, un Dockerfile. À utiliser dès que l'utilisateur veut créer, adapter ou mettre en ligne une application web destinée à AgomApps ou agom.app, quelle que soit la stack (Node.js, Bun, Python, PHP, C#/.NET, Go, Java, Rust…). Keywords: deploy, hosting, AgomApps, agom.app, agomapps.yaml, Dockerfile, mise en ligne, hébergement.
---

# AgomApps : générer une application web prête à déployer

L'utilisateur est un client d'**AgomApps** (https://agom.app), une plateforme d'hébergement d'applications web. Il te décrit son projet ; tu génères (ou adaptes) l'application **et** ce qu'il faut pour qu'il la mette en ligne tout de suite sur AgomApps, sans réglage technique de sa part.

## Ce que tu dois produire

À la **racine** du projet :

1. les fichiers de l'application ;
2. un fichier **`agomapps.yaml`** qui indique à AgomApps comment installer, construire et lancer l'application
   (§5). Nom exact : `agomapps.yaml` (un fichier `agomapps.yml` n'est **pas** lu) ;
3. si la stack l'exige (C#, Go, Java, Rust…) ou pour un besoin particulier : un **`Dockerfile`** (§7).

Puis, dans ta réponse, les informations de mise en ligne décrites au §10.

Si tu ne peux pas créer de fichiers (simple chat), donne le contenu **complet** de chaque fichier, avec son chemin.

## Procédure

1. Choisis la stack demandée par l'utilisateur, sinon la plus simple pour le besoin. Node.js, Bun, Python et PHP se déploient **sans Dockerfile** ; les autres stacks **avec un Dockerfile**.
2. Écris l'application en respectant les contraintes d'exécution (§2) et les règles communes (§3) : écoute sur
   `0.0.0.0:$PORT`, disque en lecture seule sauf `/tmp`, aucun état conservé localement, route `/health`.
3. Données à conserver : base SQL hébergée par Agom (§2, « Stockage persistant »), jamais un fichier local.
4. Ajoute `agomapps.yaml` en partant de la recette de la stack (§4).
5. Relis la liste de vérification (§10) et rends le projet avec les instructions de mise en ligne.

Ne demande pas à l'utilisateur de configurer un serveur : tout passe par les fichiers du projet et par les
variables d'environnement qu'il saisira dans son tableau de bord AgomApps.

---

## 1. Fonctionnement d'AgomApps

1. **Source** : depuis son tableau de bord (https://agom.app), le client relie un dépôt **GitHub ou GitLab** (privé
   possible) et choisit une branche, **ou** envoie le projet en **archive zip**. Chaque nouveau commit (ou zip) peut
   donner un nouveau déploiement.
2. **Build** : AgomApps récupère le code (dernier commit, sans historique ni sous-modules ; ou contenu du zip) et
   construit une image Docker :
   - avec un `Dockerfile` à la racine : il est utilisé **tel quel** (§7) ;
   - sinon, AgomApps génère le Dockerfile du runtime indiqué dans `agomapps.yaml` (ou détecté, §5) : image officielle
     légère, commande d'installation, commande de build facultative, utilisateur non root.
3. **Mise en production** : le conteneur démarre, AgomApps attend qu'il réponde au **health check**, puis bascule
   l'URL publique vers lui, sans coupure.
4. **URL publique** en HTTPS (certificat géré par la plateforme), de la forme
   `https://<projet>-<id>-<compte>.agom.app`, plus les domaines personnalisés du client.

---

## 2. Contraintes à l'exécution (toutes stacks)

| Contrainte | Conséquence pour l'application |
|---|---|
| **Disque en lecture seule** | Impossible d'écrire dans le dossier de l'application, `$HOME` ou ailleurs. **Seul `/tmp` est inscriptible** |
| `/tmp` : en mémoire, **256 Mo**, **vidé à chaque redémarrage et à chaque déploiement** | Rien de ce qui y est écrit n'est conservé ; `/tmp` compte dans la mémoire du conteneur |
| **Port imposé** | Écouter sur **`0.0.0.0:$PORT`** (variable `PORT` fournie, égale au port déclaré dans `agomapps.yaml`). Jamais `127.0.0.1` ni `localhost` |
| HTTPS terminé en amont par un proxy | L'app reçoit du HTTP ; les en-têtes `X-Forwarded-Proto`, `X-Forwarded-For`, `X-Forwarded-Host` donnent la requête d'origine. Leur faire confiance (une couche de proxy) pour générer des URL en `https` sur le bon domaine |
| **Aucun état conservé localement** | Données, fichiers envoyés, sessions serveur : base SQL ou service externe (voir ci-dessous) |
| Utilisateur non root, aucune capacité système, bac à sable | Pas de `sudo`, `apt-get`, `systemctl`, création d'utilisateur, pare-feu, montage, ni port < 1024 |
| Ressources | 1 CPU et 512 Mo par défaut (davantage selon l'offre du client), 256 processus, 1024 fichiers ouverts |
| Réseau | Internet accessible ; le réseau local du serveur et les autres conteneurs ne le sont pas |
| **Health check** | `GET <chemin>` (défaut `/`) doit répondre un code **< 400 en moins de 60 s** après le démarrage (redirections suivies) |
| Logs | Écrire sur la **sortie standard / d'erreur** (visibles dans le tableau de bord), pas dans des fichiers |

### Stockage persistant

- **Données** : Agom propose l'**hébergement de bases SQL : MySQL, PostgreSQL et Supabase**. L'application s'y
  connecte par une URL de connexion en variable d'environnement (ex. `DATABASE_URL`, ou `SUPABASE_URL` et
  `SUPABASE_ANON_KEY` pour Supabase), communiquée au client par Agom. Prévois la création du schéma (migrations ou
  script SQL fourni, §10).
- **Pas de SQLite** ni de fichier de données local pour ce qui doit être conservé : il serait perdu au prochain
  déploiement. SQLite dans `/tmp` n'est acceptable que pour une démonstration jetable.
- **Fichiers envoyés par les utilisateurs** : en base (petits fichiers) ou dans un stockage objet externe
  (compatible S3, ou Supabase Storage avec Supabase) ; `/tmp` seulement pour un traitement temporaire.
- **Sessions** : cookies signés avec une clé fixe en variable d'environnement, ou en base.

### Pendant le build

(commandes d'installation et de build, ou `RUN` d'un Dockerfile)

- accès à Internet, durée maximale **15 minutes**, image finale de **4 Go** au plus ;
- pas de systemd, de service ni de pare-feu : un script d'installation « serveur classique » ne convient pas ;
- un script appelé directement doit être **exécutable dans Git** (`git update-index --chmod=+x script.sh`), sinon
  l'appeler par `sh script.sh` ;
- **`/tmp` du build n'existe plus à l'exécution** : n'y préparer rien d'utile au lancement.

---

## 3. Règles communes à toutes les stacks

1. **Port** : lire `PORT` dans l'environnement et écouter sur `0.0.0.0`.
2. **Configuration** par **variables d'environnement** (secrets, URL de base, clés d'API). Jamais de secret dans le
   code, dans `agomapps.yaml` ni dans un `.env` versionné.
3. **Écritures disque** : caches, fichiers temporaires et compilations vers `/tmp` ; données à garder en base.
4. **Route de santé** `/health` qui répond `200` rapidement, sans base de données ni redirection, déclarée dans
   `agomapps.yaml`.
5. **Serveur de production** (pas un serveur de développement) et dépendances aux **versions figées** (fichier de
   verrouillage : `package-lock.json`, `bun.lock`, `composer.lock`, versions exactes dans `requirements.txt`…).
6. **Démarrage rapide** (< 60 s) : ce qui peut être préparé au build l'est au build.
7. **Proxy** : faire confiance aux en-têtes `X-Forwarded-*`.

---

## 4. Recettes par stack

Chaque recette donne l'`agomapps.yaml` correspondant. Adapte les commandes au projet généré.

### Node.js (Express, Fastify, Next.js, Nuxt, SvelteKit…)

Dockerfile généré : image `node:<version>-slim`, `NODE_ENV=production`, utilisateur `node`. Versions : 18, 20,
22 (défaut), 24.

- Écoute : `app.listen(process.env.PORT || 3000, '0.0.0.0')`. Proxy (Express) : `app.set('trust proxy', 1)`.
- **`NODE_ENV=production` au build** : `npm install` / `npm ci` n'installent **pas** les `devDependencies`. Si le
  build en a besoin (TypeScript, Vite, Next.js…), installer avec `npm ci --include=dev`.
- `npm ci` exige un `package-lock.json` dans le projet ; sinon utiliser `npm install`.
- Next.js : lancement `next start -p $PORT -H 0.0.0.0`. L'optimisation d'images écrit dans `.next/cache`, en lecture
  seule à l'exécution : `images: { unoptimized: true }` dans `next.config`.

```yaml
runtime: node@22
build:
  install: npm ci --include=dev
  command: npm run build          # supprimer si le projet n'a pas d'étape de build
web:
  command: npm start
  port: 3000
  healthcheck:
    path: /health
```

### Bun

Dockerfile généré : image `oven/bun:<version>-slim`, utilisateur `bun`. Détecté avec `package.json` + `bun.lock`.

- `Bun.serve({ port: Number(process.env.PORT) || 3000, hostname: '0.0.0.0', fetch })`.

```yaml
runtime: bun@1.4
build:
  install: bun install --frozen-lockfile
web:
  command: bun run src/index.ts
  port: 3000
  healthcheck:
    path: /health
```

### Python (Flask, Django, FastAPI…)

Dockerfile généré : image `python:<version>-slim`, utilisateur non root. Versions : 3.10 à 3.13 (défaut 3.12).
Détecté avec `requirements.txt` (installation par défaut : `pip install -r requirements.txt`) ou `pyproject.toml`.

- Serveur de production dans `requirements.txt` : `gunicorn` (WSGI) ou `uvicorn` (ASGI). Base : `psycopg[binary]`
  (PostgreSQL) ou `PyMySQL` (MySQL), sans bibliothèque système à installer.
- **Flask** : `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1, x_prefix=1)` ; `SECRET_KEY` fixe
  en variable d'environnement ; dossier `instance/` sous `/tmp` s'il est utilisé.
- **Django** : `ALLOWED_HOSTS` et `CSRF_TRUSTED_ORIGINS` depuis l'environnement ;
  `SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")`, `USE_X_FORWARDED_HOST = True` ; base via
  `DATABASE_URL` ; statiques avec `whitenoise` et `collectstatic` en commande de build ; migrations au lancement.
- **FastAPI** : `uvicorn main:app --host 0.0.0.0 --port $PORT --proxy-headers --forwarded-allow-ips='*'`.
- gunicorn : `--worker-tmp-dir /tmp` **obligatoire**. Caches de bibliothèques : `XDG_CACHE_HOME=/tmp/.cache`,
  `MPLCONFIGDIR=/tmp/matplotlib`, `HF_HOME=/tmp/hf`.

```yaml
runtime: python@3.12
build:
  install: pip install -r requirements.txt
  # command: python manage.py collectstatic --noinput     # Django
web:
  command: gunicorn --bind 0.0.0.0:$PORT --workers 2 --worker-tmp-dir /tmp --access-logfile - app:app
  port: 8000
  healthcheck:
    path: /health
```

### PHP (scripts simples, Symfony, Laravel…)

Dockerfile généré : image officielle `php:<version>-apache` (Apache + PHP), versions 8.2 à 8.5 (défaut 8.4).
**Aucune commande de lancement** : Apache écoute sur le port **8080**. Détecté avec `composer.json` ou un fichier
`.php` à la racine.

- **Installation** : `composer install --no-dev --optimize-autoloader` automatique si `composer.json` est présent
  (fournir `composer.lock`). Pas de Node dans cette image : les assets front sont fournis déjà compilés (ou générés
  par Symfony AssetMapper : `php bin/console asset-map:compile` en commande de build).
- **Dossier servi** : `public/` s'il existe (URL redirigées vers `public/index.php`), sinon la racine ; autre
  dossier : `php.document_root`. Les `.htaccess` sont pris en compte ; `vendor/`, `.git`, `composer.json`,
  `*.yaml`, `*.md`, `*.log`… ne sont jamais servis.
- **Extensions** toujours présentes : `pdo_mysql`, `pdo_pgsql`, `intl`, `zip`, `opcache` (plus celles de base :
  `mbstring`, `curl`, `json`…). Autres : `php.extensions` (ex. `gd`, `redis`, `bcmath`) ou `ext-*` dans
  `composer.json`.
- **Variables d'environnement** disponibles dans `getenv()`, `$_ENV` et `$_SERVER`.
- **Symfony** : la plateforme fixe `APP_ENV=prod`, `APP_CACHE_DIR=/tmp/app/cache`, `APP_LOG_DIR=/tmp/app/log` (lus
  par le Kernel). Logs Monolog vers `php://stderr`. `APP_SECRET` et `DATABASE_URL` en variables d'environnement.
- **Laravel** : stockage dans `/tmp` (dans `bootstrap/app.php` : `$app->useStoragePath('/tmp/storage')`, et créer au
  démarrage `framework/{cache,sessions,views}` et `logs`), `LOG_CHANNEL=stderr`, sessions en cookie ou en base,
  `APP_KEY` en variable d'environnement.
- **Sessions PHP natives** : stockées dans `/tmp`, donc perdues à chaque déploiement : préférer cookies ou base.

```yaml
runtime: php@8.4
php:
  extensions: [gd]               # en plus des extensions toujours installées
  # document_root: public        # défaut : public/ s'il existe, sinon la racine
web:
  port: 8080
  healthcheck:
    path: /health                # ex. health.php ou route du framework
```

### C# / ASP.NET Core (publié sur le poste, envoyé en zip)

AgomApps ne compile pas le .NET : le client **publie** l'application sur son poste (Visual Studio ou
`dotnet publish`) et envoie un zip contenant le résultat et un Dockerfile minimal qui le copie.

- Publication : `dotnet publish -c Release -r linux-x64 --self-contained false -o deploy/publish`.
- Port : les images ASP.NET écoutent déjà sur **8080** ; déclarer `port: 8080`.
- Proxy : `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true` (HTTPS et IP du visiteur corrects sans code).
- Disque en lecture seule : `DOTNET_EnableDiagnostics=0`. Clés Data Protection (cookies d'authentification,
  antiforgery) : les stocker en base pour qu'elles survivent aux déploiements.
- Base : `Npgsql` (PostgreSQL) ou `MySqlConnector` (MySQL), chaîne de connexion en variable d'environnement
  (`ConnectionStrings__Default`).
- La version de l'image (`aspnet:10.0`) doit correspondre au `TargetFramework` (`net10.0`).

Contenu du zip (fichiers dans `deploy/`) :

```
Dockerfile
agomapps.yaml
publish/          ← sortie de dotnet publish
```

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled
WORKDIR /app
COPY publish/ ./
ENV DOTNET_EnableDiagnostics=0 \
    ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
EXPOSE 8080
ENTRYPOINT ["dotnet", "MonApp.dll"]
```

```yaml
web:
  port: 8080
  healthcheck:
    path: /health
```

### Autres stacks (Go, Java, Rust, Ruby, Deno…)

Avec un **Dockerfile** (§7) : compilation au build (Dockerfile multi-étapes), ou binaire déjà construit copié dans
l'image. Mêmes règles : écouter `0.0.0.0:$PORT`, n'écrire que dans `/tmp`, route `/health`.

---

## 5. Référence `agomapps.yaml`

Toutes les options ; celles qui sont absentes prennent la valeur par défaut ou la détection automatique.

```yaml
name: mon-app                    # facultatif : minuscules, chiffres, tirets
runtime: node@22                 # node@…, bun@…, python@…, php@… ; absent : détection automatique
                                 # (composer.json ou .php → PHP ; package.json → Node, ou Bun avec bun.lock ;
                                 # requirements.txt ou pyproject.toml → Python). Ignoré avec un Dockerfile.
build:                           # ignoré avec un Dockerfile
  install: npm ci                # installation des dépendances (défaut selon le runtime)
  command: npm run build         # facultatif, lancée après l'installation
web:
  command: npm start             # commande de lancement (PHP : aucune ; Dockerfile : son CMD par défaut)
  port: 3000                     # obligatoire dès que « web » est présent ; l'app reçoit aussi PORT
  healthcheck:
    path: /health                # défaut /
  screenshot:
    path: /                      # page capturée pour la vignette du tableau de bord ; défaut /
php:                             # runtime PHP seulement
  extensions: [gd, redis]
  document_root: public
resources:
  cpu: 1                         # défaut 1
  memory: 512M                   # défaut 512M ; ex. 256M, 1G
workers:                         # programmes de fond, 10 au plus (§6)
  - name: mails                  # minuscules, chiffres, tirets, 20 caractères au plus
    command: node worker.js
    egress: true                 # accès sortant (Internet, base, API) ; défaut false
```

Règles :

- clés inconnues refusées (le build échoue avec le détail) ;
- une ligne commentée (`#`) est ignorée ; un fichier sans aucune option active équivaut à l'absence de fichier ;
- `build.command` seule (sans `build.install`) remplace l'installation par défaut et doit installer les dépendances ;
- chemins (`healthcheck`, `screenshot`) : commencent par `/` ;
- les réglages saisis par le client dans son tableau de bord **l'emportent** sur ce fichier, champ par champ ;
- jamais de secret dans ce fichier.

---

## 6. Variables d'environnement, workers, chemin de base

**Variables d'environnement** : saisies par le client dans son tableau de bord, chiffrées, appliquées au prochain
déploiement. Une variable peut être rendue visible **aussi pendant le build** (ex. jeton d'un registre de paquets
privé), sans être écrite dans l'image. Noms réservés : `PORT`, `HOME`, `PATH`, `HOSTNAME`, `BASE_PATH`, `VCEL_*`.

**Workers** : programmes de fond (consommateur de file, envoi d'e-mails…) lancés depuis la **même image** que
l'application, chacun dans son conteneur, sans port ni URL. Sans `egress: true`, un worker n'a ni Internet ni accès
à la base. Il tourne en continu : s'il s'arrête dans les 5 premières secondes, la mise en production échoue.
Une base de données, Redis ou RabbitMQ ne sont **pas** des workers.

**Tâches planifiées (cron)** : pas encore prises en charge. Utiliser un worker qui boucle, ou un déclencheur externe
qui appelle une route protégée.

**Chemin de base** (facultatif, réglé par le client) : si l'application est servie sous un préfixe (ex.
`/tickets`), elle reçoit `BASE_PATH=/tickets` au build et à l'exécution et doit le gérer (Next.js :
`basePath: process.env.BASE_PATH` ; ASP.NET : `UsePathBase` ; Flask : monter l'app sous le préfixe). Par défaut,
l'application est servie à la racine.

---

## 7. Déployer avec un Dockerfile

Un `Dockerfile` à la racine est utilisé **tel quel** : `runtime`, `build.install` et `build.command` de
`agomapps.yaml` sont alors ignorés. Déclarer quand même `web.port` (et `web.command` si l'image n'a ni `CMD` ni
`ENTRYPOINT`). À utiliser pour une stack non prise en charge directement, une bibliothèque système (`ffmpeg`,
`poppler`, `libvips`…) ou un build particulier.

```dockerfile
# Exemple : application Go compilée au build (multi-étapes)
FROM golang:1.24 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12
COPY --from=build /out/app /app
USER nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

L'image est **refusée** si :

- elle porte un `LABEL` dont le nom contient `caddy` ou commence par `vcel.` ;
- elle déclare un **`VOLUME`** (y compris hérité de l'image de base, ex. `mysql`, `postgres`, `redis` : une base de
  données ne se déploie pas comme une application ; utiliser les bases hébergées par Agom) ;
- elle dépasse **4 Go**.

Les contraintes du §2 s'appliquent : disque en lecture seule sauf `/tmp`, écoute sur `0.0.0.0:$PORT`. Terminer
par un `USER` non root.

---

## 8. Tester en local (si tu peux exécuter des commandes)

```sh
docker build -t monapp .        # avec le Dockerfile du projet, ou un Dockerfile équivalent à celui généré (§4)
docker run --rm -p 8080:8080 -e PORT=8080 \
  --read-only --tmpfs /tmp:rw,size=256m --cap-drop ALL --security-opt no-new-privileges \
  --memory 512m --cpus 1 monapp
curl -i http://localhost:8080/health        # doit répondre 200
```

Si l'application démarre et répond ainsi (disque en lecture seule, sans privilège), elle démarrera sur AgomApps.
Sinon, donne cette commande au client pour qu'il la lance.

---

## 9. Diagnostic

Les logs du déploiement, dans le tableau de bord, affichent chaque étape (installation, build, lancement) puis la
**sortie de l'application**. Si le client te les transmet :

| Message | Cause | Correction |
|---|---|---|
| `Read-only file system`, `EROFS`, `Permission denied` à l'écriture | Écriture hors de `/tmp` | Rediriger vers `/tmp` ou la base |
| `Permission denied`, code 126 au build | Script non exécutable dans Git | `git update-index --chmod=+x`, ou `sh script.sh` |
| `"npm ci" nécessite un package-lock.json` | Lockfile absent | Fournir `package-lock.json`, ou `npm install` |
| Outil de build introuvable (`tsc`, `vite`, `next`…) | `devDependencies` non installées | `npm ci --include=dev` |
| `s'est arrêté (état restarting)` | L'app plante au démarrage | Lire la sortie de l'application dans le log |
| `health check en échec … Connection refused` | N'écoute pas sur `0.0.0.0:$PORT` | Corriger l'adresse ou le port |
| `health check en échec … HTTP 404 / 500` | Route de santé absente ou en erreur | Route `/health` simple |
| Health check qui expire | Démarrage > 60 s | Préparer au build ce qui peut l'être |
| `commande de lancement requise` | Ni Dockerfile, ni `web.command` (hors PHP) | Renseigner `web.command` |
| `image refusée : labels réservés / VOLUME / trop lourde` | Dockerfile (§7) | Retirer le label ou le `VOLUME`, alléger l'image |
| Redirections vers `http://` ou vers une adresse interne | En-têtes de proxy ignorés | Faire confiance à `X-Forwarded-*` |
| Données perdues après un déploiement | Stockage dans `/tmp` ou dans le conteneur | Base SQL hébergée par Agom |

---

## 10. Liste de vérification et réponse au client

Avant de rendre le projet, vérifie :

- [ ] `agomapps.yaml` à la racine (nom exact), avec `web.port` et `web.healthcheck.path` ;
- [ ] l'application écoute sur `0.0.0.0:$PORT` ;
- [ ] aucune écriture hors de `/tmp` ; aucune donnée à conserver hors de la base ;
- [ ] route `/health` qui répond `200` sans base ni redirection ;
- [ ] aucun secret dans les fichiers ; fichier de verrouillage des dépendances présent ;
- [ ] `Dockerfile` présent si la stack n'est ni Node.js, ni Bun, ni Python, ni PHP.

Puis termine ta réponse par une section **« Mise en ligne sur AgomApps »** contenant :

1. **Déploiement** : se connecter à https://agom.app, créer un projet, relier le dépôt GitHub/GitLab **ou** envoyer
   le zip. Pour un zip : contenu à la racine (ou dans un seul dossier), **sans** `node_modules`, `vendor`, `.venv`,
   `__pycache__`, `bin/`, `obj/` ni `.git` ; 50 Mo au plus, 300 Mo décompressé, 20 000 fichiers, aucun lien
   symbolique. Exemple : `zip -r app.zip . -x 'node_modules/*' '.git/*' 'vendor/*'`.
2. **Variables d'environnement** à saisir dans le tableau de bord : un tableau nom / rôle / secret ou non / exemple
   ou façon de le générer (ex. `openssl rand -base64 48`). Signaler celles à rendre visibles pendant le build.
3. **Base de données** : le type choisi (MySQL, PostgreSQL ou Supabase, hébergés par Agom), la variable qui reçoit
   la connexion, et comment créer le schéma (migrations lancées au démarrage, ou script SQL fourni).
4. Ce qui **ne sera pas conservé** entre deux déploiements, s'il y en a (ex. fichiers dans `/tmp`).
5. La commande de **test local** (§8), si utile.

