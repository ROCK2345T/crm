# Décisions prises pour le projet (et pourquoi)

> Cheick m'a demandé de décider, puis d'expliquer. Chaque décision est réversible.
> Dernière mise à jour : 3 octobre 2026.

## Vue d'ensemble de ce qui s'est passé

1. **Lecture.** J'ai lu le CRM d'origine (`trycompai/crm`). Trois programmes (interface,
   API, agent IA) se parlent par une base Postgres. Détail : `ARCHITECTURE.md`.
2. **Mauvais dépôt au départ.** La session était liée à « Trouve ton stage ». Rien n'y a été
   modifié, hormis un dossier temporaire ajouté puis retiré. Votre fork `ROCK2345T/crm` est
   désormais la base du travail, sur la branche `claude/docs-cheick`.
3. **Documentation.** Architecture, configuration, installation Windows, et ce fichier.
4. **Vérifications.** Installation et typage réussis dans le cloud. Les tests exigent Docker,
   donc ils se lancent sur votre PC.
5. **Supabase.** `crm-dakar` créé (non branché). `supabase-pink-island` (Trouve ton stage)
   est en pause.

## Les décisions

| # | Question | Décision | Pourquoi | Pour revenir en arrière |
|---|---|---|---|---|
| 1 | Où va la clé Context.dev ? | **Dans l'interface du CRM** (écran d'accueil ou Settings → General), jamais dans un fichier | Le CRM stocke cette clé en base. Ce n'est pas une variable d'environnement. Une clé dans un fichier risque d'être commitée | Changer la clé dans Settings → General |
| 2 | Un CRM par client ou un seul partagé ? | **Une installation par PME cliente** | Le code est mono-locataire par conception. Partager un CRM mélangerait les données de clients différents. Cela garde l'architecture d'origine à 100 % | Une refonte multi-locataires serait un gros chantier (à discuter avant) |
| 3 | Quelle base pour développer ? | **Postgres local via Docker** (celui de `docker compose`) | C'est le chemin documenté et testé par les auteurs. Supabase n'est pas testé avec ce CRM | Pointer `DATABASE_URL` vers Supabase plus tard |
| 4 | Télémétrie ? | **Désactivée** : `CRM_TELEMETRY_DISABLED=1` | Par défaut, des comptes anonymes partent chaque jour vers le projet des auteurs. Inacceptable à l'insu de vos clients | Retirer la variable |
| 5 | Mode de connexion ? | **Google seul** pour commencer, avec `ALLOWED_SIGN_IN` = votre adresse | C'est le chemin le plus court. Microsoft et SSO s'ajoutent plus tard | Ajouter Microsoft ou un SSO dans les réglages |
| 6 | Branche de travail ? | **`claude/docs-cheick`**, partie de `release` | Dépôt personnel : pas besoin de la discipline `main`/`release` des auteurs | Créer une autre branche depuis `main` |
| 7 | Secret entre app et agent ? | **Définir `AGENT_BRIDGE_SECRET`** | Sans lui, en local, l'agent ne démarre aucune tâche et la clé Context n'est pas vérifiée | Retirer la variable |
| 8 | Code de l'application ? | **Aucune modification pour l'instant** | Votre objectif est d'abord un projet qui tourne tel quel | — |

## Ce que je n'ai pas décidé à votre place

- **Les fonctionnalités à ajouter** (cibles, données à suivre, usages des commerciaux).
- **L'abandon de l'architecture d'origine.** Vous l'avez exclu.
- **Le plan tarifaire** des services payants.

## Règles de sécurité retenues

- Aucune clé dans un fichier versionné, ni dans le chat. Une clé collée dans le chat doit être
  révoquée puis régénérée.
- `.env` reste local et ignoré par Git.
