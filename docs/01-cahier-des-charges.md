# 01 — Cahier des charges

> Portfolio + vitrine freelance de Nathan, avec l'identité visuelle d'une plateforme de jeux façon Steam.
> Le backend (Kotlin / Ktor) est écrit par Nathan, guidé étape par étape. Le front est géré par Claude.

## 1. Objectifs

| # | Objectif | Mesure de réussite |
|---|----------|--------------------|
| O1 | Présenter mon profil, mon CV et mon parcours | Page Profil en ligne, CV téléchargeable |
| O2 | Montrer mes projets comme une « bibliothèque de jeux » | Fiches projets complètes, filtrables |
| O3 | Vendre mes services freelance via un « shop » | Offres + options, demande de devis fonctionnelle |
| O4 | **Apprendre à construire un vrai backend Kotlin** | API complète, testée, dockerisée, déployée |
| O5 | Partager ce que j'apprends dans un **blog** (progression + visibilité) | Articles publiés depuis l'admin |

## 2. Périmètre

### Inclus (v1)
- Site public bilingue **FR / EN**.
- Page **Profil** : identité (avatar, pseudo, statut de disponibilité, bio), **liens utiles**, **CV** (aperçu + téléchargement PDF), **parcours / timeline**.
- Page **Bibliothèque** : liste de projets, fiche détaillée (captures + vidéo, tags/stack/statut, liens démo + GitHub, temps passé, date de sortie), filtres par tag/statut.
- Page **Shop** : offres avec prix « à partir de » et **options (DLC)** ; formulaire de **demande de devis**.
- Page **Social** : **blog** (articles en Markdown, brouillons, tags, projet lié en option, FR obligatoire et EN facultatif) et **livre d'or / avis** avec **modération** par l'admin.
- **Panel admin** (un seul compte) : CRUD de tout le contenu (articles compris), upload de médias, d'images d'articles et de CV, traitement des devis, modération des avis.
- Notification **par email** à chaque devis, stockage en base.

### Exclu (v1)
- Comptes visiteurs, wishlist, paiement en ligne, compteurs de vues, webhook Discord.
- Commentaires sous les articles (modération et spam).

### Choix retenus
- Le **Profil** contient les bases (avatar, pseudo, statut, bio) et la timeline, en complément du CV.
- Les **avis** forment un livre d'or global affiché dans **Social**, avec un projet ou une offre rattachés en option.
- Idéalement, chaque étape du backend donne lieu à un article.
- Un seul compte admin (Nathan).

## 3. Acteurs

- **Visiteur** : consulte, filtre, lit le blog, télécharge le CV, demande un devis, laisse un avis.
- **Admin (Nathan)** : gère tout le contenu et traite les demandes.
- **Système** : envoie les emails, applique la limitation de débit.

## 4. Exigences fonctionnelles

### Public
| ID | Exigence |
|----|----------|
| F1 | Afficher le profil (champs traduits selon la langue demandée) |
| F2 | Afficher et télécharger le CV courant |
| F3 | Afficher la timeline (formation / expérience / jalon), triée |
| F4 | Lister les projets publiés, filtrer par tag, statut, catégorie |
| F5 | Afficher une fiche projet par `slug` |
| F6 | Lister les offres publiées avec leurs options |
| F7 | Envoyer une demande de devis (offre optionnelle + options cochées) |
| F8 | Lister les avis approuvés ; soumettre un avis (statut `PENDING`) |
| F9 | Lister les articles publiés (paginé, filtre par tag) ; lire un article par `slug` |

### Admin
| ID | Exigence |
|----|----------|
| A1 | Se connecter (JWT) |
| A2 | CRUD profil, liens, timeline |
| A3 | Téléverser un CV, choisir la version courante |
| A4 | CRUD projets, tags, médias (upload image, lien vidéo) |
| A5 | CRUD offres et options |
| A6 | Lister les devis, changer leur statut |
| A7 | Modérer les avis (approuver / rejeter / supprimer) |
| A8 | CRUD articles, brouillon / publication, upload d'images |

## 5. Exigences non fonctionnelles

- **Sécurité** : mots de passe hachés (Argon2id ou BCrypt), JWT à durée courte, CORS restreint, limitation de débit sur les POST publics, champ honeypot anti-spam, validation stricte des entrées, vérification type/taille des uploads.
- **RGPD** : les emails des devis/avis sont des données personnelles → mentions légales, politique de confidentialité, durée de conservation définie, suppression possible sur demande.
- **Performance** : réponses publiques < 200 ms ; en-têtes de cache sur les listes ; images redimensionnées.
- **Fiabilité** : migrations versionnées, sauvegarde `pg_dump` quotidienne, endpoint `/health`.
- **Qualité** : tests automatisés (unitaires + intégration), CI sur chaque PR, API documentée en OpenAPI.
- **Maintenabilité** : architecture en couches, DTO séparés des entités, injection de dépendances.
- **i18n** : le contenu éditorial existe en FR et EN, sauf les articles (EN facultatif, repli sur le FR) ; le front choisit via `?lang=` ou en-tête `Accept-Language`.
- **Markdown** : le rendu des articles est assaini côté front (pas de HTML ou script injecté).
- **Accessibilité / SEO** : gérés côté front (SSG, balises meta, contrastes).

## 6. Pages du site (front)

Navigation principale : **Boutique · Bibliothèque · Social · Profil**.

| Route | Équivalent Steam | Contenu |
|-------|------------------|---------|
| `/` ou `/profile` | Page profil | Identité, statut, liens utiles, CV, timeline |
| `/library` | Bibliothèque | Liste + filtres |
| `/library/[slug]` | Page du jeu | Fiche projet |
| `/shop` | Boutique | Offres |
| `/shop/[slug]` | Page produit | Détail + options (DLC) + devis |
| `/social` | Communauté | Articles du blog + livre d'or |
| `/social/[slug]` | Actualité | Lecture d'un article |
| `/admin/**` | — | Panel admin (protégé) |
| `/legal` | — | Mentions légales / confidentialité |

## 7. Stack technique

| Couche | Choix | Pourquoi |
|--------|-------|----------|
| Langage | Kotlin (JVM 21) | Ta stack principale |
| Framework | **Ktor** | Léger, idiomatique Kotlin, formateur |
| Build | Gradle (Kotlin DSL) + version catalog | Standard |
| Sérialisation | kotlinx.serialization | Natif Kotlin |
| DI | Koin | Simple à comprendre |
| Base de données | PostgreSQL 16 | Robuste, standard |
| Accès données | **Exposed** (DSL) + HikariCP | Idiomatique Kotlin |
| Migrations | Flyway | Schéma versionné |
| Auth | Ktor Auth JWT + hash Argon2/BCrypt | Standard |
| Email | SMTP (Jakarta Mail ou simple-java-mail) | Notif devis |
| Doc API | OpenAPI / Swagger UI | Contrat clair pour le front |
| Tests | kotlin.test + `testApplication` + Testcontainers | BD réelle en test |
| Conteneurs | Docker + Docker Compose | Portable |
| Reverse proxy | Caddy (HTTPS auto) | Config minimale |
| CI/CD | GitHub Actions | Build + tests + déploiement |
| Front (Claude) | Next.js + Tailwind (SSG) | SEO, perf |

> Les versions exactes seront figées à l'étape 0 (vérification des dernières versions stables).

## 8. Risques

| Risque | Impact | Parade |
|--------|--------|--------|
| Trop de périmètre pour apprendre | Abandon | Livrer par étapes, chaque étape utilisable |
| Blog laissé à l'abandon | Page Social vide | Articles courts, un par étape, brouillons |
| Spam sur devis/avis | Boîte mail polluée | Rate limit, honeypot, modération |
| Perte de données | Contenu perdu | Sauvegardes + migrations |
| Fuite de secrets | Compromission | Variables d'env, `.env` hors Git |

## 9. Questions encore ouvertes

1. Nom de domaine / pseudo affiché sur le site ?
2. Palette exacte et logo (on reprend l'ancien style : à examiner ensemble) ?
3. Fournisseur SMTP (Gmail, Brevo, OVH) ?
4. Devise et TVA (micro-entreprise → prix HT/TTC) ?
