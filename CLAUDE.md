@AGENTS.md

## Contexte du projet (Cheick)

- Propriétaire : Cheick, consultant indépendant à Dakar (Sénégal). Ce dépôt est un fork de
  `trycompai/crm` (licence MIT). Il en fera son propre CRM, vendu à des PME de Dakar.
- Décision prise, à ne pas remettre en question : garder l'architecture d'origine à 100 %
  (Vercel, eve, AI Gateway, Context.dev, Perplexity). Cheick paiera les clés API nécessaires,
  mais les comptes payants seront ajoutés plus tard dans le projet.
- Méthode : le code s'écrit avec Claude Code dans le cloud, puis Cheick récupère la branche
  dans VS Code sur Windows (Git Bash, Bun, Docker Desktop) pour lancer et tester.
- Niveau de Cheick : intermédiaire (comprend les API et l'architecture, pas développeur de
  formation). Répondre toujours en français, expliquer le pourquoi avant le comment, et
  signaler clairement toute hypothèse.
- Règles : si un point est ambigu, poser la question au lieu de deviner. Ne jamais mettre de
  clé, mot de passe ou secret dans un fichier versionné. Ne pas toucher au fichier `LICENSE`.
- Documentation préparée pour Cheick : `docs/cheick/ARCHITECTURE.md`, `CONFIGURATION.md`,
  `INSTALLATION-WINDOWS.md`.
- Base de données : un projet Supabase `crm-dakar` (région eu-west-3) existe, non branché.
  Son usage avec Prisma n'est pas testé.
- Décisions déjà prises et leurs raisons : `docs/cheick/DECISIONS.md`. Le compte Context.dev
  existe ; sa clé se saisit dans l'interface du CRM, jamais dans un fichier.

## Context.dev (trace d'intégration)

- Le CRM intègre déjà Context.dev. Ne pas ajouter de second module, ni de client dans `apps/api`.
- Module unique : `apps/agent/agent/lib/context-dev.ts` (marque, recherche web, extraction) et
  `apps/agent/agent/lib/people.ts` (personnes). SDK officiel `context.dev` 2.10.0 dans `apps/agent`.
- Clé : stockée en base (`AppSetting`, lecteur `readContextDevKey` de `@crm/db/settings`), saisie dans
  l'interface (écran d'accueil, puis Settings → General). Ce n'est PAS une variable d'environnement :
  ne jamais créer `CONTEXT_DEV_API_KEY` (voir `docs/environment.md`). Jamais dans un fichier versionné.
- Contrôle de clé gratuit déjà codé (`verifyKey`) : `brand.retrieve` par e-mail Gmail, `401` = clé inconnue.
- Appels utilisés par le module : `brand.retrieve` (POST /brand/retrieve, 10 crédits), `web.search`
  (POST /web/search), `web.extract`, `people.enrich` (POST /people/enrich, 20 crédits), `utility.prefetch`.
- Documentation : https://docs.context.dev (ajouter `.md` à une URL pour du texte brut).
  Marque : https://docs.context.dev/api-reference/brand-intelligence/brand
  Recherche : https://docs.context.dev/api-reference/web-scraping/search
  Personnes : https://docs.context.dev/api-reference/people/enrich
- Environnement cloud : le proxy bloque `api.context.dev` tant que l'hôte n'est pas autorisé dans les
  réglages réseau de l'environnement. Aucun appel réel n'a donc été fait depuis le cloud.
- Les tests automatiques ne doivent pas appeler l'API réelle (chaque appel coûte des crédits).
