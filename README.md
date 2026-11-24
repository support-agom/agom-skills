# Skills AgomApps pour les assistants IA

Ce dépôt contient le skill **`agom-apps`** : un fichier d'instructions qui apprend à votre assistant IA (Claude,
ChatGPT, Gemini, Cursor, Codex…) à générer une application web **prête à déployer sur
[AgomApps](https://agom.app)**, l'hébergement d'applications web d'Agom, sans réglage technique de votre part.

Avec ce skill, l'IA :

- écrit, à la racine du projet, le fichier `agomapps.yaml` qui indique à AgomApps comment installer et lancer
  l'application (et un `Dockerfile` si votre stack en a besoin) ;
- respecte les contraintes de l'hébergement : port imposé, disque en lecture seule, route de santé `/health` ;
- range les données à conserver dans une base SQL hébergée par Agom : **MySQL, PostgreSQL ou Supabase** ;
- vous donne la liste des variables d'environnement à créer dans votre tableau de bord AgomApps ;
- vous indique comment tester l'application et l'envoyer depuis votre tableau de bord
  [agom.app](https://agom.app) (dépôt GitHub/GitLab ou archive zip).

Stacks prises en charge : Node.js, Bun, Python, PHP sans Dockerfile ; C#/.NET, Go, Java, Rust, Ruby… avec un
Dockerfile.

## Contenu du dépôt

```
skills/
└── agom-apps/
    └── SKILL.md     ← les instructions pour l'IA
```

## Installation selon votre outil

### Claude (claude.ai, application de bureau ou mobile)

1. Téléchargez **[agom-apps.zip](https://github.com/support-agom/agom-skills/releases/latest/download/agom-apps.zip)**.
2. Dans Claude, ouvrez les **paramètres**, section **Skills**, puis importez le fichier zip (sans le décompresser).
3. Vérifiez que le skill `agom-apps` est activé.

Claude l'utilisera ensuite automatiquement dès que vous lui demanderez une application pour AgomApps.

### Claude Code

Pour tous vos projets :

```sh
git clone https://github.com/support-agom/agom-skills.git
mkdir -p ~/.claude/skills
cp -r agom-skills/skills/agom-apps ~/.claude/skills/
```

Pour un seul projet, copiez plutôt le dossier dans `.claude/skills/` à la racine du projet.

### Autres agents de code (Cursor, Codex, Gemini CLI, Copilot…)

Copiez le dossier `skills/agom-apps/` dans le dossier de skills de votre outil s'il en gère un. Sinon, placez
`SKILL.md` dans votre projet et demandez à l'agent de le lire avant de commencer :

> Lis `SKILL.md` et applique ses instructions pour ce projet.

### ChatGPT, Gemini ou tout autre chat

Joignez le fichier **[SKILL.md](https://raw.githubusercontent.com/support-agom/agom-skills/main/skills/agom-apps/SKILL.md)**
à votre conversation (clic droit sur le lien, « Enregistrer sous… »), puis décrivez votre projet.

## Exemple de demande

> Utilise le skill AgomApps joint. Crée une application de prise de rendez-vous pour un cabinet de kinésithérapie,
> en Python avec Flask : formulaire de réservation, liste des rendez-vous pour le praticien, e-mail de
> confirmation. Je veux la déployer sur AgomApps depuis un dépôt GitHub.

L'IA vous rendra le projet complet, la liste des variables d'environnement à créer, la base de données à
utiliser (MySQL, PostgreSQL ou Supabase, hébergées par Agom) et la façon de la préparer, et les étapes de mise en
ligne sur [agom.app](https://agom.app).

## Mise à jour

Le skill évolue avec la plateforme. Téléchargez à nouveau le zip ou le fichier, ou, si vous avez cloné le dépôt :

```sh
cd agom-skills && git pull
cp -r skills/agom-apps ~/.claude/skills/
```

## Assistance

Un déploiement échoue ? Les logs du déploiement, dans votre tableau de bord [agom.app](https://agom.app),
indiquent l'étape en cause. Copiez-les dans votre conversation avec l'IA : le skill contient un tableau de
diagnostic des erreurs courantes.
