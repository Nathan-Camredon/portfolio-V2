# 04 — Roadmap et parcours d'apprentissage

Principe : **chaque étape produit quelque chose qui tourne**, introduit 1 ou 2 concepts neufs, et se termine par des tests.
Méthode : j'explique le concept → tu codes → je relis → on corrige. Je ne donne la solution qu'en dernier recours.

Git : une branche par étape (`feature/…`), Conventional Commits, PR vers `preprod`, merge après ta validation.

## Phase A — Fondations

| Étape | Branche | Tu apprends | Livrable | Critère de fin |
|-------|---------|-------------|----------|----------------|
| 0 | `feature/init-project` | JDK, Gradle Kotlin DSL, version catalog, structure du dépôt, `.gitignore`, `.env.example` | Ktor « Hello » + `/api/v1/health` | `./gradlew run` répond 200 |
| 1 | `feature/routing` | Routing, paramètres de chemin/requête, codes HTTP, client HTTP (`.http` IntelliJ) | Routes bouchonnées pour projets/offres | Toutes les routes publiques répondent |
| 2 | `feature/json-dto` | kotlinx.serialization, ContentNegotiation, DTO vs domaine | Réponses JSON typées | Contrat de `03-api.md` respecté |
| 3 | `feature/architecture-di` | Couches, interfaces, Koin, configuration par env (HOCON/`application.yaml`) | Routes → services → repos (en mémoire) | Un repo se remplace sans toucher la route |

## Phase B — Persistance

| Étape | Branche | Tu apprends | Livrable | Critère de fin |
|-------|---------|-------------|----------|----------------|
| 4 | `feature/database` | Docker Compose pour Postgres, HikariCP, Flyway, Exposed (tables, requêtes, transactions) | Schéma V1 + lecture des projets en base | `GET /projects` lit Postgres |
| 5 | `feature/i18n-content` | Jointures, tables de traduction, filtres/pagination, index | Projets, tags, médias, offres, timeline, profil en lecture | Tout le contenu public servi en FR/EN |
| 6 | `feature/error-handling` | StatusPages, exceptions métier, validation, format d'erreur unique | Erreurs homogènes | Aucune stacktrace côté client |

## Phase C — Sécurité et administration

| Étape | Branche | Tu apprends | Livrable | Critère de fin |
|-------|---------|-------------|----------|----------------|
| 7 | `feature/admin-auth` | Hash de mot de passe, JWT, plugin Authentication, rôles, secrets | Login admin + routes protégées | 401 sans token, 200 avec |
| 8 | `feature/admin-crud` | CRUD complet, transactions, conflits (409), slugs | CRUD projets/offres/tags/timeline/profil | Tout le contenu modifiable via l'API |
| 9 | `feature/uploads` | Multipart, validation de fichiers, stockage disque, service de fichiers statiques | Upload images + CV, téléchargement | Mauvais type/taille refusés |

## Phase D — Interactions visiteurs

| Étape | Branche | Tu apprends | Livrable | Critère de fin |
|-------|---------|-------------|----------|----------------|
| 10 | `feature/quotes` | Validation d'entrée, rate limiting, honeypot, envoi SMTP asynchrone (coroutines) | `POST /quotes` + email + admin | Un devis crée une ligne **et** un mail |
| 11 | `feature/reviews` | Machine à états, modération, ne pas exposer de données privées | Avis + modération | Un avis `PENDING` n'apparaît pas en public |

## Phase E — Qualité et mise en production

| Étape | Branche | Tu apprends | Livrable | Critère de fin |
|-------|---------|-------------|----------|----------------|
| 12 | `feature/openapi-docs` | OpenAPI, Swagger UI, CORS, en-têtes de cache | Doc interactive | Le front peut se brancher dessus |
| 13 | `feature/ci` | GitHub Actions, cache Gradle, Testcontainers en CI | Pipeline build + tests | PR rouge si test cassé |
| 14 | `feature/docker-deploy` | Dockerfile multi-étapes, compose prod, Caddy, variables d'env, sauvegardes | Stack déployable | `docker compose up` en local = prod |
| 15 | `feature/go-live` | VPS, DNS, HTTPS, logs, monitoring de base | Site en ligne | Accessible sur ton domaine |

> **Les tests ne sont pas une étape à part** : à partir de l'étape 1, chaque étape ajoute ses tests (`testApplication` pour les routes, unitaires pour les services, Testcontainers pour les repos). Je te montrerai comment dès l'étape 1.

## Côté front (Claude, en parallèle)

| Quand | Quoi |
|-------|------|
| Après étape 2 | Squelette Next.js + thème Steam, pages branchées sur des données mockées conformes à `03-api.md` |
| Après étape 5 | Pages Profil, Bibliothèque, Shop branchées sur l'API réelle |
| Après étape 8 | Panel admin |
| Après étape 10-11 | Formulaires devis et avis |
| Après étape 14 | SEO, performance, accessibilité, déploiement |

## Hors périmètre v1 (idées pour plus tard)

- Compteurs de vues, webhook Discord, comptes visiteurs, paiement Stripe.
- Réécriture partielle en Kotlin Multiplatform si tu veux pousser KMP.
- Observabilité (Prometheus/Grafana), cache Redis.

## Définition de « terminé » pour une étape

- [ ] Le code compile et tourne localement
- [ ] Tests ajoutés et verts
- [ ] Contrat d'API respecté (`03-api.md` mis à jour si on dévie)
- [ ] Commits en Conventional Commits, PR vers `preprod`
- [ ] Tu sais **expliquer** ce que tu as écrit (c'est le vrai critère)
