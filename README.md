<div align="center">

# Vanyla

**Construisez votre bot Discord visuellement, repartez avec le code.**
Un builder no-code pour assembler un bot Discord modulaire (tickets, modération, économie, Minecraft…), prévisualisable en direct, puis exportable en projet source complet et auto-hébergeable.

![Status](https://img.shields.io/badge/status-en%20développement-orange)
![Modules](https://img.shields.io/badge/modules-35%2B-blueviolet)
![License](https://img.shields.io/badge/license-propriétaire-lightgrey)

</div>

---

> **À propos de ce dépôt.** Vanyla est un projet propriétaire en développement ; le code source n'est pas publié ici. Ce dépôt documente l'architecture, les choix techniques et le fonctionnement du produit, à des fins de portfolio.

## Le principe

La plupart des bots Discord "no-code" du marché restent **locataires** : le bot tourne chez le fournisseur, l'utilisateur ne récupère jamais le code. Vanyla fait l'inverse — l'utilisateur configure visuellement son bot (active des modules, règle leurs options), le teste en conditions réelles sur son propre serveur grâce à un aperçu live, puis **exporte un vrai projet discord.js** qu'il héberge où il veut. Il repart propriétaire de son code, pas abonné à un service.

## Fonctionnement

```
┌──────────────────────┐
│   Builder visuel       │   L'utilisateur active des modules et règle
│   (React Flow)          │   leurs options dans une interface de workflow
└───────────┬───────────┘
            │
            ▼
┌──────────────────────┐
│   Bot sandbox          │   Un bot unique, propriété de la plateforme,
│   (aperçu live)         │   invité sur le serveur Discord de test de
│                          │   l'utilisateur : il simule le comportement
│                          │   réel du bot pendant la configuration
└───────────┬───────────┘
            │  export
            ▼
┌──────────────────────┐
│   Moteur de génération  │   Assemble les templates des modules activés
│   de code                │   + la config utilisateur → projet discord.js
│                          │   complet (commands/, events/, package.json,
│                          │   README, .env.example)
└───────────┬───────────┘
            │
            ▼
   Projet livré en .zip, prêt à être hébergé sur le serveur
   du client (son propre token Discord, son propre VPS)
```

## Catalogue de modules

Plus de 35 modules prêts à l'emploi, panachables librement :

| Catégorie | Modules |
|---|---|
| **Modération** | modération, auto-modération, anti-raid, vérification, logs |
| **Communauté** | tickets, suggestions, sondages, giveaways, starboard, réaction-rôles |
| **Engagement** | niveaux (leveling), économie, mini-jeux, anniversaires, bienvenue |
| **Minecraft** | lien de compte, statut serveur, alertes de statut, whitelist |
| **Utilitaires** | auto-thread, salon de compteur de membres, commandes personnalisées, messages planifiés |
| **Social** | notifications de réseaux sociaux, jeux gratuits, profils gaming, LFG, hubs vocaux |
| **Premium** | assistant IA, dashboard web embarqué |

## Modèle économique

| Offre | Prix | Ce que ça débloque |
|---|---|---|
| **Gratuit** | 0 € | Modules de base, 1 export par mois |
| **Solo** | 49,99 € (paiement unique) | Tous les modules, un projet, exports illimités sur ce projet |
| **Entreprise** | 99,99 € / mois | Tous les modules, plusieurs projets en parallèle |

Configurer et tester un bot dans le sandbox est toujours gratuit, y compris sur l'offre gratuite — seul l'**export final** du code est soumis au quota. C'est la propriété du code généré, et non la location d'un bot hébergé, qui justifie le positionnement.

## Stack

`React` · `Vite` · `TypeScript` · `React Flow` · `Node.js` · `PostgreSQL` · `Prisma` · `discord.js` · `Stripe` · `pnpm workspaces` · `Turborepo`

---

## Architecture technique

<details>
<summary><strong>Détails pour les développeurs</strong> (monorepo, génération de code, config vivante)</summary>

### Monorepo

```
apps/
  web/              builder UI + dashboard (React + Vite)
  api/               API principale (auth, billing, CRUD projets)
  sandbox-bot/        process Node.js connecté en permanence au Gateway
                        Discord, pour l'aperçu live
  dashboard-web-ui/   UI React du module premium "dashboard web"
packages/
  shared-types/       types TS partagés (manifest de module, billing)
  db/                  client Prisma partagé
  module-templates/    manifest + templates de chaque module (37+)
  codegen/             moteur qui assemble templates + config → export
```

Géré en `pnpm workspaces` + `Turborepo`.

### Deux identités Discord bien distinctes

Point d'architecture volontairement non négociable : le **bot sandbox** (un seul, propriété de la plateforme, dédié à la prévisualisation) et le **bot exporté** (propre à chaque client, son propre token créé sur son propre Developer Portal Discord) ne doivent jamais être confondus. La plateforme ne fournit jamais de token partagé pour le bot final livré au client.

### Système de modules

Chaque module est défini par un **manifest** (métadonnées + schéma de configuration) associé à un ou plusieurs fichiers template. Le moteur de génération (`packages/codegen`) lit la configuration du projet, injecte les valeurs dans les templates, puis assemble l'arborescence finale du bot exporté (commandes, event handlers, `package.json` avec les dépendances filtrées selon les modules réellement activés, `README.md` et `.env.example` générés).

Les champs texte libre fournis par l'utilisateur (messages, templates avec variables type `{user}`) sont systématiquement échappés avant injection dans le code généré, pour éviter l'injection de mentions de masse (`@everyone`/`@here`) ou de code non désiré.

### Config vivante (post-export)

Certains réglages d'un bot déjà exporté restent modifiables sans réexport ni redémarrage : les champs structurels (couleur, salon, seuil, booléen) d'un module déjà actif sont lus dynamiquement via un `configStore` à chaque invocation du handler, et persistés en base si le module **Dashboard Web** (premium) est actif. Activer/désactiver un module, en revanche, change l'arborescence de fichiers et les dépendances : cela nécessite toujours une réexportation.

Le module Dashboard Web embarque un petit serveur Express dans le bot exporté lui-même, qui sert une UI React pré-buildée et une API REST protégée par OAuth2, en réutilisant l'application Discord du bot du client — pas de double authentification.

### Sécurité & robustesse

- TypeScript strict sur tout le monorepo.
- Variables d'environnement validées au démarrage (schéma Zod), jamais de secret en dur.
- Versions des dépendances (notamment `discord.js`) épinglées dans le code exporté, pour qu'une montée de version majeure ne casse pas un bot déjà déployé chez un client.
- Paiements Stripe : mode *payment* (one-time) pour l'offre Solo, mode *subscription* avec webhooks pour l'offre Entreprise.

</details>

---

<sub>Documentation de portfolio — le code source reste privé.</sub>
