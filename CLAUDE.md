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
