# Installer et lancer le CRM sur Windows (Git Bash + Bun + Docker Desktop)

> Les commandes viennent du `README.md` et de `docs/setup.md` du dépôt. J'ai exécuté
> `bun install` et `bun run check-types` dans le cloud (Linux) avec succès. **Je n'ai
> jamais lancé le CRM sous Windows** : tout ce qui est propre à Windows est donc à
> confirmer en le faisant, et je le marque « hypothèse » quand je ne suis pas sûr.

## Pourquoi ces trois outils

- **Git Bash** : un terminal Linux sous Windows. Les commandes du dépôt (`cp`, `openssl`)
  sont écrites pour ce type de terminal, pas pour PowerShell.
- **Bun** : le moteur qui installe les dépendances et lance les trois programmes
  (le dépôt impose `bun@1.3.12`).
- **Docker Desktop** : fait tourner Postgres (la base) dans une boîte isolée, sans rien
  installer d'autre sur Windows.

## Étape 0. Vérifier votre PC

- Windows 10/11 64 bits, virtualisation activée dans le BIOS (Docker Desktop s'appuie
  sur WSL2).
- Compter plusieurs Go de disque. Les dépendances sont nombreuses (1 351 paquets chez moi).

## Étape 1. Installer Git (avec Git Bash)

1. Téléchargez Git sur https://git-scm.com et lancez l'installeur (options par défaut).
2. Ouvrez **Git Bash** (menu Démarrer) et vérifiez : `git --version`.

## Étape 2. Installer Bun

Dans **PowerShell** (pas Git Bash) :

```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

Fermez puis rouvrez Git Bash, et vérifiez : `bun --version`.
Si la commande n'est pas trouvée, ajoutez `~/.bun/bin` au PATH (hypothèse : le cas se
présente parfois après une installation Windows).

Le dépôt demande Bun **1.3.12**. J'ai utilisé la 1.3.14 sans problème pour l'installation
et la vérification des types. Une version proche suffit probablement ; en cas de souci,
installez exactement la 1.3.12.

## Étape 3. Installer Docker Desktop

1. Téléchargez Docker Desktop, installez-le, **acceptez l'activation de WSL2**.
2. **Redémarrez Windows si demandé.**
3. Lancez Docker Desktop et attendez que l'état soit « Running ».
4. Vérifiez dans Git Bash : `docker --version` puis `docker ps` (aucune erreur).

## Étape 4. Récupérer le code dans VS Code

1. `Ctrl + Maj + P` → **Git: Clone** → adresse **de votre fork**.
2. Ouvrez le dossier, puis la branche donnée par Claude (icône de branche en bas à gauche)
   et faites **Pull**.
3. Ouvrez un terminal VS Code de type **Git Bash** (menu déroulant du terminal).

## Étape 5. Créer le client OAuth Google (connexion + Gmail + Agenda)

Pourquoi : sans connexion Google, Microsoft ou SSO, personne ne peut entrer dans le CRM.
Le même client Google lit aussi vos mails et votre agenda, ce qui nourrit l'agent.

1. Allez sur https://console.cloud.google.com et créez un **projet** (ex. `crm-dakar`).
2. Menu **API et services > Bibliothèque** : activez **Gmail API** et
   **Google Calendar API**.
3. **Écran de consentement OAuth** :
   - Compte **Google Workspace** : type d'utilisateur **Interne** (pas de validation Google).
   - Compte **Gmail personnel** : seul **Externe** est possible. `gmail.readonly` est un
     périmètre restreint ; au-delà du mode test, Google exige une validation. En mode
     test, **ajoutez votre propre adresse comme « utilisateur test »** (hypothèse sur ce
     détail d'interface : à confirmer à l'écran).
4. **Identifiants > Créer des identifiants > ID client OAuth > Application Web.**
5. Sous **URI de redirection autorisés**, ajoutez exactement :
   `http://localhost:3001/api/auth/callback/google`
   (c'est l'adresse de l'**API**, port 3001, pas celle de l'app).
6. Copiez l'**ID client** et le **Code secret du client** dans votre `.env` (étape 6).
   Ne les collez nulle part d'autre (ni chat, ni Git).

## Étape 6. Préparer le fichier `.env`

Dans Git Bash, à la racine du dépôt :

```bash
cp .env.example .env
```

Générez le secret de session :

```bash
openssl rand -base64 32
```

(Git for Windows fournit normalement `openssl` dans Git Bash : hypothèse à confirmer.
Si la commande est introuvable, générez-la autrement, avec au moins 32 caractères
aléatoires.)

Copiez le résultat comme valeur de `BETTER_AUTH_SECRET` dans `.env`. Remplissez ensuite
**au minimum** :

| Variable | Valeur |
| --- | --- |
| `BETTER_AUTH_SECRET` | le résultat de la commande ci-dessus |
| `ALLOWED_SIGN_IN` | votre adresse, ex. `vous@gmail.com` (une seule adresse = installation à une personne) |
| `GOOGLE_CLIENT_ID` | l'ID client de l'étape 5 |
| `GOOGLE_CLIENT_SECRET` | le secret de l'étape 5 |
| `CRM_TELEMETRY_DISABLED` | `1` (coupe la télémétrie anonyme ; voir `DECISIONS.md`) |

Laissez `DATABASE_URL` tel quel : il correspond déjà au Postgres de Docker. Pour faire
fonctionner le poke de l'agent, générez aussi
`AGENT_BRIDGE_SECRET` avec la même commande `openssl rand -base64 32`, et fixez
`AGENT_URL="http://127.0.0.1:2000"` (pas `localhost`).

La **clé Context.dev ne va pas dans `.env`** : vous la saisirez dans l'application, à l'étape 8.

Ne commitez **jamais** `.env` (il est ignoré par Git ; ne le forcez pas).

## Étape 7. Lancer

```bash
bun install
docker compose up -d        # démarre Postgres sur le port 5432
bun run db:deploy           # applique les migrations
bun run db:seed             # facultatif : fausses données pour explorer
bun run dev                 # app :3000, api :3001, agent :2000
```

Notes importantes :

- `bun install` échoue au script `postinstall` si `.env` n'existe pas encore (Prisma a
  besoin de `DATABASE_URL`). **Faites donc l'étape 6 avant.** C'est ce qui m'est arrivé.
- `bun install` configure un hook Git `pre-push` qui exécute les contrôles (types, lint,
  tests) et **exige Postgres**. Pour pousser sans Docker : `git push --no-verify`.
- Dans le terminal Turbo, l'agent est **interactif** : sélectionnez son panneau et
  appuyez sur **Entrée**. Sinon, `bun run dev:headless --filter=agent` (voir
  `ARCHITECTURE.md`).
- Ne lancez pas deux fois `bun run dev` (le second fait échouer le premier).
- Sous Windows, la commande `lsof` du dépôt (pour trouver un agent resté actif sur le
  port 2000) n'existe pas. Utilisez `netstat -ano | findstr :2000` dans PowerShell
  (hypothèse).

## Étape 8. Se connecter

1. Ouvrez `http://localhost:3000` (dans le navigateur de votre PC, jamais depuis une page web ou un chat) et connectez-vous avec Google (avec l'adresse de
   `ALLOWED_SIGN_IN`).
2. L'accueil demande le nom et le site de votre entreprise.
3. **Attention :** l'écran suivant demande une **clé Context.dev** et n'a pas de bouton
   « passer ». Voir `CONFIGURATION.md`, section « Piège ». C'est le point à trancher
   avant votre premier test.

## À propos des adresses `localhost`

`localhost` signifie « cette machine-ci ». Les adresses `http://localhost:3000` (app),
`http://localhost:3001` (API) et `http://127.0.0.1:2000` (agent) ne fonctionnent que dans le
navigateur **du PC où `bun run dev` tourne**. Cliquées dans un chat, une session
Claude Code cloud ou un document en ligne, elles visent une autre machine et échouent :
c'est normal. L'application doit d'abord tourner sur votre PC.

## Dépannage rapide

| Symptôme | Cause probable |
| --- | --- |
| Boucle entre `/sign-in` et `/` | `BETTER_AUTH_SECRET` différent entre processus, ou plusieurs `.env` |
| Erreur Google « redirect_uri did not match » | L'URI de l'étape 5 n'est pas exactement `http://localhost:3001/api/auth/callback/google` |
| Impossible de se connecter | `ALLOWED_SIGN_IN` vide, ou adresse différente de celle utilisée chez Google |
| Onglet Agent : `503` | `AGENT_BRIDGE_SECRET` absent côté app |
| Onglet Agent : `401` | Les deux processus n'ont pas la même valeur |
| Onglet Agent : `502` | Agent arrêté ou `AGENT_URL` faux |
| Entreprises qui restent en attente | Pas de clé Context, ou pas de `AGENT_BRIDGE_SECRET` (poke) : lancer `bun run --filter=agent dispatch` |
| Erreur `P1001` | Postgres injoignable : `docker compose up -d` pas lancé ou Docker Desktop arrêté |

## État du projet dans l'environnement cloud (point 5 de la mission)

| Commande | Résultat |
| --- | --- |
| `bun install` | **Échec au premier essai** : `PrismaConfigEnvError: Cannot resolve environment variable: DATABASE_URL`. Après `cp .env.example .env` : **réussi** (1 351 paquets) |
| `bun run check-types` | **Réussi** : 13 tâches sur 13, 37 s. La génération des types tRPC a donné 21 routeurs et 160 procédures |
| `bun run test` | **Échec attendu, lié à l'environnement** : erreur Prisma `P1001` (base injoignable). Dans `@crm/telemetry` : 45 tests réussis, 16 échoués. Turbo s'arrête à la première erreur : **les suites des autres paquets n'ont pas toutes tourné** |

Cause : l'environnement cloud n'a **pas de démon Docker** (`/var/run/docker.sock`
absent) et aucun serveur Postgres. Je n'ai rien contourné, comme demandé (par exemple,
je n'ai pas installé Postgres à la main). Les tests du CRM sont de vrais tests
d'intégration qui écrivent dans une base : ils doivent donc se lancer **sur votre PC**,
avec Docker Desktop, après `docker compose up -d` et `bun run db:test`.

Je ne peux donc pas affirmer que la suite de tests passe. Seul le typage est vérifié.
