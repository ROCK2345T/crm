# Architecture du CRM (fork de trycompai/crm)

> Document préparé pour Cheick. Il décrit le dépôt `trycompai/crm` tel que je l'ai lu
> (clone en lecture seule, branche par défaut `release`, licence MIT, Bun 1.3.x).
> Chaque affirmation vient d'un fichier lu. Ce qui est une **hypothèse** est marqué comme tel.

## 1. L'idée centrale (le « pourquoi »)

La plupart des CRM sont une base de données avec un formulaire devant. Celui-ci est
construit à l'envers : **l'agent IA est le travailleur, et le CRM est le cahier où il
note ce qu'il a trouvé.** L'agent tourne dans son propre déploiement, selon son propre
calendrier, sur sa propre file de travail. Si vous fermez le navigateur, il continue.

Trois règles gouvernent tout le code (source : `README.md`, `AGENTS.md`, `docs/api.md`) :

1. **L'intelligence ne vit jamais dans l'API.** L'API signale « il s'est passé quelque
   chose » en écrivant une ligne en base. Seul l'agent décide ce que ça signifie.
2. **`packages/ui` est l'unique source de composants d'interface.** On n'écrase pas leurs
   styles à l'endroit où on les utilise.
3. **Il n'y a pas d'organisations.** Un seul locataire (« single tenant »), volontairement.
   Pour vendre à plusieurs PME, il faudra **une installation par client** (hypothèse de
   conception à valider avec vous, voir les questions en fin de fichier).

Une règle de données en plus : **rien sur une personne n'est deviné.** Les outils de
l'agent rapportent ce qu'ils ont observé ; une preuve forte écrit dans la fiche, une
preuve faible devient une suggestion qu'un humain tranche.

## 2. Vue d'ensemble : trois programmes et une base

```
 Navigateur ──► apps/app  (Next.js, :3000) ──tRPC──► apps/api (NestJS, :3001) ──► Postgres
                    │                                       │  ▲
                    │ /eve/v1/* (jeton signé 2 min)         │  │ lit / écrit
                    ▼                                       ▼  │
                apps/agent (eve, :2000) ◄──── lit les lignes AgentTask ────────┘
                    │
                    ├─ Vercel AI Gateway  (le modèle de langage)
                    ├─ Context.dev        (marque d'une entreprise, profil LinkedIn)
                    └─ Perplexity         (recherche web, optionnel)
```

Ils ne s'appellent presque jamais directement. **Ils communiquent par la base.** Les deux
seules choses qu'ils doivent partager sont `DATABASE_URL` et `BETTER_AUTH_SECRET`
(`README.md`, section Deploying).

## 3. Le rôle de chaque dossier

### `apps/app` — l'interface (Next.js App Router, port 3000)

Ce que voit le commercial : listes d'entreprises, fiches, onglet **Agent** sur chaque
fiche. Elle parle à l'API en **tRPC** (typage de bout en bout) et garde les filtres, tris
et pages dans l'URL. Elle sert aussi de relais pour l'agent : le navigateur n'appelle
jamais l'agent directement ; l'app vérifie la session puis fabrique un jeton signé de
deux minutes (`AGENT_BRIDGE_SECRET`). Un fichier `proxy.ts` y applique des « portes » :
accueil (onboarding), puis saisie de la clé Context (voir `CONFIGURATION.md`, section
« Piège »).

### `apps/api` — le serveur de données (NestJS, port 3001)

HTTP, authentification, tRPC (21 routeurs, 160 procédures d'après la génération de types
que j'ai exécutée), synchronisation des boîtes mail (Gmail, Outlook), calendrier, Slack,
suivi de site. **Il ne contient aucune IA.** Quand quelque chose arrive (entreprise
créée, contact créé), `AgentTriggerService` écrit une ligne `AgentTask` dans Postgres.
Les tâches planifiées (« crons ») sont dans `apps/api/vercel.json` : synchro des mails
toutes les 5 minutes, taux de change, télémétrie, nettoyage.

### `apps/agent` — l'agent de recherche (framework eve, port 2000)

Un déploiement séparé, construit sur **eve** (framework Vercel pour agents durables).
Principe « tout est un fichier » :

| Dossier | Contenu |
| --- | --- |
| `agent/tools/` | Les outils (27 fichiers `.ts` : `enrich_company`, `research_person`, `record_fact`, `schedule_recheck`, `search_crm`…). Le README annonce « 18 outils », le dossier en contient plus : le README est probablement en retard (hypothèse) |
| `agent/skills/` | 4 fiches en markdown que l'agent lit : `evidence`, `identity-matching`, `data-boundaries`, `writing-a-brief` |
| `agent/schedules/dispatch.ts` | La seule planification : toutes les minutes, elle « loue » les tâches dues et lance une session par ligne |
| `agent/lib/` | La logique (file d'attente `tasks.ts`, `capabilities.ts`, `brand.ts`, `context-dev.ts`, `perplexity.ts`, `evidence.ts`…) |
| `agent/sandbox/` | Un shell isolé (`bash`, `grep`) **sans réseau et sans accès à la base** |

### `packages/*` — le code partagé

| Package | Rôle |
| --- | --- |
| `packages/db` | Schéma Prisma, migrations, client Postgres partagé, `seed.ts` (fausses données), utilitaires (`agent-tasks`, `blob`, `fields`, `settings`…) |
| `packages/auth` | Configuration Better Auth (Google, Microsoft, SSO), liste d'autorisation `ALLOWED_SIGN_IN`, cookies |
| `packages/ui` | Composants shadcn/ui et thème Tailwind (vert de marque `#006B4F`) |
| `packages/validation` | Schémas Zod des données qui traversent les paquets (une forme = un module) |
| `packages/env` | Trouve et charge **le** fichier `.env` à la racine du dépôt |
| `packages/telemetry` | Télémétrie anonyme côté serveur (désactivable, voir `CONFIGURATION.md`) |
| `packages/typescript-config` | Réglages TypeScript communs |

Dossiers racine utiles : `docs/` (règles détaillées), `adrs/` (décisions), `tools/`,
`.agents/skills/` (guides pour agents de code), `docker-compose.yml` (Postgres local).

## 4. Le parcours d'une entreprise : de l'ajout à l'enrichissement

Chemin vérifié dans `companies.service.ts`, `agent-trigger.service.ts`,
`schedules/dispatch.ts`, `lib/tasks.ts` et `lib/brand.ts`.

1. **Ajout.** Le commercial crée l'entreprise (nom, domaine) dans `apps/app`. Le domaine
   est normalisé ; un doublon actif sur le même domaine est refusé.
2. **Enregistrement.** `apps/api` (`CompaniesService.create`) écrit la ligne `company`
   (statut d'enrichissement `PENDING`) et un événement `company.created`, dans une même
   transaction.
3. **Deux tâches en file.** `AgentTriggerService.companyCreated` écrit deux lignes
   `AgentTask` : `brand` (priorité 900, budget 2) et `company-profile` (priorité 40,
   budget 4). **Une ligne, pas un appel HTTP** : elle survit même si l'agent est arrêté.
4. **Réveil immédiat (optionnel).** L'API « pique » l'agent (`poke`) pour ne pas attendre
   la minute suivante. Cela exige `AGENT_BRIDGE_SECRET` ; sans lui, la tâche attend le
   passage planifié.
5. **Prise en charge.** À chaque minute, `dispatch.ts` appelle `claimDue`, qui loue les
   lignes dues avec `FOR UPDATE SKIP LOCKED` (deux répartiteurs ne prennent jamais la
   même ligne ; un plantage libère la ligne à l'expiration du bail).
6. **Deux voies.**
   - *Voie visible* (`brand`, `portrait`) : traitée **directement, sans modèle IA**
     (60 par passage). Pour `brand`, `lib/brand.ts` interroge Context.dev avec le domaine
     et remplit logo, couleur, secteur, ville, réseaux sociaux. Il **ne remplit que les
     champs vides** et ne remplace jamais ce qu'un humain a saisi. Les images sont copiées
     dans Vercel Blob.
   - *Voie recherche* (le reste, dont `company-profile`) : **une session eve avec le
     modèle IA** par ligne (12 par passage), qui peut chercher dans le CRM, sur le web
     (Perplexity) et écrire des faits via `record_fact`.
7. **Règle d'or.** Preuve forte : l'agent écrit dans la fiche. Preuve faible : suggestion
   pour un humain.
8. **Clôture.** Le statut passe à `COMPLETE`, `FAILED` ou `SKIPPED`. Si l'agent veut
   revoir la fiche plus tard, il appelle `schedule_recheck` et **doit donner la raison**,
   affichée au commercial.
9. **Affichage.** L'onglet **Agent** de la fiche montre les étapes, ce qui a été écarté et
   pourquoi.

Si une clé manque, le travail concerné est **sauté sans erreur** et la fiche reste
`PENDING` pour être reprise plus tard (voir `CONFIGURATION.md`).

## 5. Lancer l'agent en local

D'après `docs/setup.md` et `apps/agent/package.json` :

```bash
bun run dev                      # lance app :3000, api :3001 et agent :2000 ensemble
```

- La commande `dev` de l'agent est `eve dev`, une **interface interactive (TUI)**. Dans
  le terminal Turbo, sélectionnez le panneau de l'agent et appuyez sur **Entrée**.
- Si votre terminal n'affiche pas bien la TUI : `bun run dev:headless --filter=agent`
  (équivaut à `eve dev --no-ui`). Passez par Turbo et non par
  `bun run --filter=agent dev:headless` : ce dernier saute `dev:prepare` et l'agent
  démarrerait sur une base non migrée.
- Chaque `bun run dev` applique d'abord les migrations et régénère le client Prisma.
- **Piège majeur :** `eve dev` **ne déclenche jamais le planning à la minute**. Les
  tâches ne partent que si `AGENT_BRIDGE_SECRET` est défini (le « poke »). Sinon, forcez
  un passage : `bun run --filter=agent dispatch`.
- `AGENT_URL` doit être `http://127.0.0.1:2000` et **pas** `localhost` (eve dev est IPv4).
- Un second `bun run dev` fait échouer toute l'exécution Turbo.
- Au démarrage, l'agent affiche ce qui est actif :
  ```
  [agent] off  Web research (PERPLEXITY_API_KEY)
  [agent] on   Company brand data (Settings → General)
  ```
- Pour la production : `eve start` (script `bun scripts/start.ts`), qui exécute le planning.

## 6. Ce qui n'est PAS vérifié

- Je n'ai **pas pu lancer l'application** : l'environnement cloud n'a ni démon Docker ni
  serveur Postgres (voir `INSTALLATION-WINDOWS.md`, section état du projet).
- **Fonctionnement de la TUI d'eve sous Windows : non testé** (hypothèse : correct, car
  `scripts/start.ts` gère `eve.cmd` sur `win32`).
- **Hypothèse non testée :** Supabase (projet `crm-dakar`) peut servir de Postgres, car le
  CRM n'exige qu'une chaîne Postgres. Le déploiement d'origine utilise Neon. Prisma avec
  un « pooler » demande `DIRECT_DATABASE_URL` pour les migrations.

## 7. Questions ouvertes pour vous

1. Une installation par PME cliente, ou un seul CRM partagé ? (Le code est mono-locataire.)
2. Faut-il adapter l'interface au français et au franc CFA ? (`docs/currency.md` traite
   des devises ; je ne l'ai pas encore lu en détail.)
