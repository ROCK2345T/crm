# Configuration du CRM : variables d'environnement et services externes

> Sources : `.env.example`, `docs/environment.md`, `docs/setup.md`,
> `apps/api/src/config/env.validation.ts`, `turbo.json`, et un balayage du code
> (`process.env.*`). Aucune valeur secrète n'est écrite ici. Les prix ne sont **pas**
> indiqués : je ne les ai pas vérifiés, consultez les pages tarifaires des fournisseurs.

## 1. Où mettre les variables

- **Un seul fichier `.env`, à la racine du dépôt**, lu par les trois programmes
  (`apps/app`, `apps/api`, `apps/agent`). Ne créez jamais de `.env` par dossier : quand
  ils divergent, le navigateur boucle entre `/sign-in` et `/`.
- `.env.local` est lu **en dernier** et **gagne** sur `.env`.
- Les vraies variables d'environnement (Vercel, Docker) l'emportent toujours sur le fichier.
- `.env` est ignoré par Git. **Ne le commitez jamais.**
- Une nouvelle variable a **trois** emplacements : `.env.example`, `env.validation.ts` (si
  l'API la lit) et `globalPassThroughEnv` dans `turbo.json` (sinon Turbo la cache aux
  tâches et le code reçoit `undefined`, sans message).

## 2. Variables obligatoires pour démarrer

`DATABASE_URL`, `BETTER_AUTH_SECRET`, `ALLOWED_SIGN_IN`, **et** au moins un moyen de
connexion (Google, Microsoft ou SSO ajouté ensuite).

## 3. Tableau de toutes les variables

Légende : **Obl.** = obligatoire ; **Opt.** = optionnelle ; **Payant** = coût direct
connu ou probable (« non » = gratuit ou inclus dans l'hébergement).

### 3.1 Base de données

| Nom exact | Rôle | Statut | Où l'obtenir | Payant |
| --- | --- | --- | --- | --- |
| `DATABASE_URL` | Chaîne de connexion Postgres. Par défaut celle du Postgres de `docker compose` | **Obl.** | `.env.example` (local) ; ou le Postgres de votre choix | Non en local ; hébergement Postgres selon fournisseur |
| `DIRECT_DATABASE_URL` | Connexion directe (sans pooler) pour `prisma migrate deploy` au build Vercel | Opt. (nécessaire seulement si `DATABASE_URL` passe par un pooler) | Tableau de bord du fournisseur Postgres | Non |
| `POSTGRES_URL_NON_POOLING` | Repli lu si `DIRECT_DATABASE_URL` est absent | Opt. | Intégration Vercel/Postgres | Non |
| `DATABASE_URL_UNPOOLED` | Autre repli lu par `apps/api/scripts/build-func.mjs` | Opt. | Intégration Vercel/Neon | Non |
| `TEST_DATABASE_URL` | Base jetable pour `bun run test`. Son nom doit finir par `_test` ; les tests refusent de démarrer sans elle | **Obl. pour les tests** | `.env.example` ; `bun run db:test` la crée | Non |
| `ALLOW_REMOTE_DB` | Mettre `1` pour autoriser `db:migrate`, `db:push`, `db:reset`, `db:seed` contre une base **distante** (garde-fou) | Opt. | À définir vous-même, avec prudence | Non |
| `PRISMA_LOG_QUERIES` | `true` journalise chaque requête SQL | Opt. | À définir vous-même | Non |

### 3.2 Authentification et accès

| Nom exact | Rôle | Statut | Où l'obtenir | Payant |
| --- | --- | --- | --- | --- |
| `BETTER_AUTH_SECRET` | Signe les cookies de session. Minimum 32 caractères. **Doit être identique** pour l'API et l'app | **Obl.** | `openssl rand -base64 32` | Non |
| `BETTER_AUTH_URL` | Ancien repli de `API_URL` | Opt. | Inutile si `API_URL` est défini | Non |
| `ALLOWED_SIGN_IN` | Liste d'autorisation : domaines ou adresses séparés par des virgules. **Vide = personne ne peut se connecter** | **Obl.** | À écrire vous-même (ex. votre adresse Gmail) | Non |
| `GOOGLE_CLIENT_ID` | Identifiant OAuth Google : connexion **et** lecture Gmail/Agenda | Opt., **à fournir avec le secret** | Google Cloud Console | Non (voir note Gmail) |
| `GOOGLE_CLIENT_SECRET` | Secret OAuth Google | Opt., par paire | Google Cloud Console | Non |
| `MICROSOFT_CLIENT_ID` | Identifiant Entra ID : connexion et lecture Outlook | Opt., par paire | Portail Azure | Non |
| `MICROSOFT_CLIENT_SECRET` | Secret Entra ID (expire : 24 mois au plus) | Opt., par paire | Portail Azure (copier la **Valeur**) | Non |
| `MICROSOFT_TENANT_ID` | Locataire autorisé (`common` par défaut) | Opt. | Portail Azure | Non |
| `SLACK_CLIENT_ID` / `SLACK_CLIENT_SECRET` | Liaison de comptes Slack (Paramètres > Connexions) | Opt. | Application Slack | Non |
| `AUTH_COOKIE_DOMAIN` | Domaine parent du cookie si app et API sont sur deux sous-domaines | Opt. (déploiement) | Votre domaine, ex. `.exemple.com` | Non |
| `IS_MARKETING` | `true` sert la page d'accueil marketing de Comp AI à la racine. Valeur par défaut : désactivé | Opt. | À définir vous-même | Non |

Règle des paires : Google (ID + secret) et Microsoft (ID + secret) se définissent **ensemble
ou pas du tout** ; une moitié de paire fait lever une erreur ou donne un bouton qui échoue.

### 3.3 Adresses et ports

| Nom exact | Rôle | Statut | Où l'obtenir | Payant |
| --- | --- | --- | --- | --- |
| `API_URL` | Origine de l'API (défaut `http://localhost:3001`). Sert à construire **toutes** les adresses de retour OAuth | Opt. (obligatoire hors localhost) | Votre déploiement | Non |
| `APP_URL` | Origine de l'app (défaut `http://localhost:3000`) | Opt. (idem) | Votre déploiement | Non |
| `NEXT_PUBLIC_API_URL` | Dérivée automatiquement de `API_URL` par `next.config.ts` | Ne pas définir | — | Non |
| `PORT` | Port de l'API (défaut 3001) ; l'agent le lit aussi en repli | Opt. | — | Non |
| `AGENT_URL` | Adresse de l'agent, **avec le schéma** (ex. `http://127.0.0.1:2000`) | Opt. (local : valeur par défaut) | — | Non |
| `AGENT_PORT` | Port de l'agent auto-hébergé (défaut 2000) | Opt. | — | Non |

### 3.4 Agent, IA et sources de recherche

| Nom exact | Rôle | Statut | Où l'obtenir | Payant |
| --- | --- | --- | --- | --- |
| `AGENT_BRIDGE_SECRET` | Active l'onglet Agent **et** le « poke » qui réveille l'agent. **Même valeur** dans l'app et l'agent | Opt. mais **fortement conseillé** (voir §5) | `openssl rand -base64 32` | Non |
| `AI_GATEWAY_API_KEY` | Accès au modèle via Vercel AI Gateway. Inutile sur Vercel (OIDC) | Opt. sur Vercel ; **nécessaire en local pour la voie « recherche »** (hypothèse tirée du `.env.example`) | Tableau de bord Vercel > AI Gateway | **Oui** (à l'usage) |
| `VERCEL_OIDC_TOKEN` | Jeton automatique fourni par Vercel | Automatique | Vercel | — |
| `PERPLEXITY_API_KEY` | Recherche web avec citations ; trouve aussi l'adresse LinkedIn | Opt. | perplexity.ai/settings/api | **Oui** |
| `GITHUB_TOKEN` | Relève la limite de l'API GitHub (60 requêtes/heure sans jeton) pour rapprocher contacts et profils GitHub | Opt. | Jeton GitHub « classic » **sans aucune permission** | Non |
| `BLOB_READ_WRITE_TOKEN` | Stockage des logos et photos (Vercel Blob) | Opt. | Vercel > Storage > Blob | **Oui** (selon volume) |
| *(pas une variable)* clé **Context.dev** | Données de marque + LinkedIn. **Saisie dans l'interface**, stockée en base | **Obligatoire à l'accueil** (voir §5) | context.dev | **Oui** |

### 3.5 Exploitation

| Nom exact | Rôle | Statut | Où l'obtenir | Payant |
| --- | --- | --- | --- | --- |
| `CRON_SECRET` | Jeton `Bearer` (16 caractères minimum) qui protège `/internal/sync/mailboxes`, `/internal/sync/rates`, `/internal/tracking/retention` ; ces routes **refusent de tourner sans lui** | Opt. mais requis pour la synchro mail | `openssl rand -base64 32` | Non |
| `REDIS_URL` | Cache partagé. Sans lui : cache en mémoire par instance | Opt. | Upstash ou un Redis | Selon fournisseur |
| `CACHE_TTL_MS` | Durée du cache en ms (défaut 60 000) | Opt. | — | Non |
| `CRM_TELEMETRY_DISABLED` | `1` coupe la télémétrie anonyme | Opt. | — | Non |
| `DO_NOT_TRACK` | `1` fait la même chose | Opt. | — | Non |

### 3.6 Variables internes ou de test (ne pas les définir)

`NODE_ENV`, `VERCEL`, `VERCEL_ENV`, `VERCEL_GIT_COMMIT_SHA`, `GIT_COMMIT_SHA`, `GITHUB_SHA`,
`NEXT_RUNTIME`, `BUN_BIN` (chemin de bun pour le build de l'API), `TEST_RUN_ID`,
`E2E_LIVE_MODEL` (`1` = tests qui **dépensent de vrais crédits**), `E2E_SLACK_JOIN`
(`1` = modifie un **vrai** espace Slack), `E2E_LOAD_COUNT`.

## 4. Services externes payants

| Service | À quoi il sert | Comment il est facturé |
| --- | --- | --- |
| **Vercel AI Gateway** | Le modèle de langage de l'agent (modèle par défaut : `zai/glm-5.2-fast`, réglable dans l'interface) | À l'usage (crédits). Tarifs : **à vérifier** |
| **Context.dev** | Logo, couleurs, secteur, ville, réseaux d'une entreprise à partir de son domaine ; lecture d'un profil LinkedIn | Crédits. `docs/agent.md` indique **10 crédits par entreprise** pour le balayage de départ. Tarifs : **à vérifier** |
| **Perplexity API** | Recherche web citée | À l'usage. Tarifs : **à vérifier** |
| **Vercel Blob** | Copie permanente des logos et photos | Selon stockage et trafic. **À vérifier** |
| **Vercel (hébergement)** | Trois déploiements (app, API, agent) + crons | Les crons à la minute exigent le plan **Pro** : sur Hobby, le planning devient quotidien (`docs/environment.md`). Le bac à sable Vercel Sandbox en production est aussi facturé (hypothèse sur le détail) |
| **Postgres hébergé** | La base. Le déploiement d'origine utilise Neon | Selon fournisseur (votre projet Supabase `crm-dakar` est une option **non testée**) |
| **Redis (Upstash)** | Cache partagé | Selon fournisseur ; optionnel |

Gratuits : Google OAuth (connexion, Gmail, Agenda ; voir note), Microsoft Entra, GitHub
(jeton sans permission), Postgres local via Docker.

Note Gmail : `gmail.readonly` est un périmètre **restreint** de Google. Un compte Google
Workspace choisit « Interne » et évite toute vérification. **Un compte Gmail personnel
ne peut être qu'« Externe »**, ce qui demande une validation Google (et un audit annuel
CASA) pour dépasser le mode test. Je n'ai pas vérifié les limites exactes du mode test.

## 5. Que se passe-t-il si une clé est absente ?

Principe du dépôt : tout ce qu'un auto-hébergeur peut ne pas avoir est **optionnel et ne
lève jamais d'erreur** ; une clé absente retire une capacité (`apps/agent/agent/lib/capabilities.ts`).
L'agent est informé au début de chaque session de ce dont il dispose.

| Clé absente | Effet constaté dans le code ou la documentation |
| --- | --- |
| **Clé Context.dev** | La tâche `brand` est marquée `SKIPPED` **avant** de passer en `RUNNING` ; l'entreprise reste `PENDING`. Elle est remise en file dès qu'une clé est enregistrée (reprise immédiate, lancée en arrière-plan). **Mais voir le piège ci-dessous.** Perdu aussi : lecture LinkedIn |
| **`PERPLEXITY_API_KEY`** | Pas de recherche web ; l'agent s'appuie sur l'historique du CRM (vos mails, réunions, signatures) et indique ce qu'il n'a pas pu vérifier. Pas d'erreur |
| **`AI_GATEWAY_API_KEY`** (hors Vercel) | **Hypothèse** : les sessions de recherche (modèle) ne peuvent pas tourner. Les tâches `brand` et `portrait` n'utilisent pas de modèle et ne sont pas concernées. À tester |
| **`BLOB_READ_WRITE_TOKEN`** | Aucune **photo** de contact n'est stockée (une URL qui expire est jugée pire que des initiales). Logos, favicons et avatars gardent l'URL d'origine, affichée comme image externe |
| **`AGENT_BRIDGE_SECRET`** | L'onglet Agent affiche « non configuré » ; le « poke » est **ignoré** (jamais envoyé sans authentification) ; une clé Context est enregistrée **sans vérification**. L'agent suit son planning. En local avec `eve dev`, **rien ne démarre** sans ce secret ou sans `bun run --filter=agent dispatch` |
| **`GITHUB_TOKEN`** | Limite à 60 requêtes/heure pour le rapprochement GitHub |
| **`REDIS_URL`** | Cache en mémoire par instance ; compteurs de limitation non partagés en multi-instances |
| **`CRON_SECRET`** | Les routes de synchro mail et de nettoyage **refusent de s'exécuter** : pas de synchro Gmail/Outlook automatique |
| **Google ET Microsoft ET SSO** | **Aucune connexion possible** ; la page de connexion l'indique explicitement |
| **`ALLOWED_SIGN_IN`** vide | Personne ne peut se connecter (échec volontairement sûr) ; l'API refuse de démarrer |
| **`BETTER_AUTH_SECRET`** < 32 caractères | L'API refuse de démarrer |

### Piège : la clé Context n'est pas vraiment « pour plus tard »

Vous avez décidé d'ajouter les clés payantes à la fin. Or, d'après `docs/api.md`
(« There is no way past the key gate but to answer »), `apps/app/proxy.ts` et
`research-form.tsx` : après l'écran d'accueil, **l'application redirige vers
`/onboarding/research` et le formulaire exige une clé Context non vide, sans bouton
« passer »**. Les auteurs l'ont fait exprès (un « Skip » laissait des installations
bloquées avec des entreprises `PENDING` sans explication).

Conséquences, **à trancher avec vous** :

1. Sans compte Context.dev, vous ne pouvez pas dépasser l'accueil pour voir le CRM.
2. Saisir une fausse valeur n'est **pas** une solution que je recommande : si l'agent et
   `AGENT_BRIDGE_SECRET` sont actifs, une clé fausse peut être refusée (réponse `401`) ;
   sinon elle est enregistrée sans contrôle et les enrichissements échoueront ensuite.
3. La vraie solution est soit de créer le compte Context.dev dès le départ, soit de
   modifier cette porte dans `apps/app/proxy.ts` et `apps/api` (session de code
   ultérieure, avec votre accord, car cela s'écarte du comportement d'origine).

Je n'ai **pas vérifié** si Context.dev propose une offre gratuite ou un essai.

## 6. Sécurité et bonne hygiène

- `.env.example` ne contient **aucun secret** (chaînes vides, un test le garantit).
  Générez les vôtres ; ne réutilisez jamais une valeur d'un exemple ou d'un autre environnement.
- `vercel env pull` écrit par défaut dans `.env.local`, qui **gagne** : un seul `pull` fait
  pointer tous les processus vers la production. Utilisez `vercel env pull .env.vercel`.
- La télémétrie envoie par défaut, une fois par jour, des **comptes anonymes** (tranches
  de volumes, outils appelés, présence de clés en booléens, UUID aléatoire) vers le projet
  PostHog des auteurs. Pour un CRM vendu à des clients, **mettez `CRM_TELEMETRY_DISABLED=1`**
  (recommandation à valider avec vous ; voir `docs/telemetry.md`, que je n'ai pas lu en détail).
