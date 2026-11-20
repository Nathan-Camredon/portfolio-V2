# 02 — Architecture et UML

> Les diagrammes sont en **Mermaid** : ils s'affichent directement dans GitHub et dans l'aperçu Markdown d'IntelliJ.

## 1. Cas d'usage

```mermaid
flowchart LR
    V([Visiteur])
    A([Admin])
    S([Système email])

    subgraph Public
        UC1(Consulter le profil)
        UC2(Télécharger le CV)
        UC3(Voir la timeline)
        UC4(Parcourir et filtrer les projets)
        UC5(Voir une fiche projet)
        UC6(Voir les offres et options)
        UC7(Demander un devis)
        UC8(Lire / laisser un avis)
    end

    subgraph Administration
        UC10(Se connecter)
        UC11(Gérer profil, liens, timeline)
        UC12(Gérer le CV)
        UC13(Gérer projets, tags, médias)
        UC14(Gérer offres et options)
        UC15(Traiter les devis)
        UC16(Modérer les avis)
    end

    V --> UC1 & UC2 & UC3 & UC4 & UC5 & UC6 & UC7 & UC8
    A --> UC10
    UC10 -.-> UC11 & UC12 & UC13 & UC14 & UC15 & UC16
    UC7 -.-> S
```

## 2. Architecture globale

```mermaid
flowchart LR
    subgraph Client
        B[Navigateur]
    end
    subgraph VPS["VPS (Docker Compose)"]
        C[Caddy<br/>HTTPS + reverse proxy]
        W[Next.js<br/>front statique/SSR]
        K[Ktor API<br/>/api/v1]
        P[(PostgreSQL)]
        F[/Volume uploads<br/>images + CV/]
    end
    M[(SMTP)]

    B --> C
    C -->|/| W
    C -->|/api| K
    W -.->|fetch au build/ISR| K
    K --> P
    K --> F
    K -->|devis| M
```

## 3. Architecture interne du backend (en couches)

```mermaid
flowchart TB
    R["Routes (HTTP)<br/>parse, valide, répond"] --> SV["Services<br/>logique métier"]
    SV --> RP["Repositories<br/>accès BD via Exposed"]
    RP --> DB[(PostgreSQL)]
    SV --> EX["Adapters externes<br/>EmailSender, FileStorage"]
    R -. DTO .- SV
    SV -. Domaine .- RP
```

Règles :
- une route **ne parle jamais** à la base directement ;
- les **DTO** (ce que voit l'API) sont séparés du **modèle de domaine** (ce que manipule le métier) ;
- les dépendances sont injectées par **Koin** (donc faciles à remplacer en test).

### Structure du dépôt

```
Portfollio/
├── backend/                 # Kotlin / Ktor (Nathan)
│   ├── build.gradle.kts
│   └── src/
│       ├── main/kotlin/dev/nathan/portfolio/
│       │   ├── Application.kt
│       │   ├── config/          # env, Koin, plugins Ktor
│       │   ├── routes/          # public/ et admin/
│       │   ├── service/
│       │   ├── repository/
│       │   ├── domain/          # entités + enums
│       │   ├── dto/
│       │   ├── security/        # JWT, hash
│       │   └── infra/           # email, stockage fichiers
│       ├── main/resources/db/migration/   # V1__init.sql ...
│       └── test/kotlin/
├── web/                     # Next.js (Claude)
├── infra/                   # docker-compose, Caddyfile, scripts backup
├── docs/
└── .github/workflows/
```

## 4. Modèle de données (ER)

Stratégie i18n : les textes éditoriaux vivent dans des **tables de traduction** (`*_translation`) avec une colonne `locale` (`fr` / `en`).

```mermaid
erDiagram
    ADMIN_USER {
        uuid id PK
        string email UK
        string password_hash
        timestamp created_at
    }

    PROFILE {
        int id PK "singleton"
        string display_name
        string avatar_path
        string availability "AVAILABLE | BUSY | UNAVAILABLE"
    }
    PROFILE_TRANSLATION {
        int profile_id FK
        string locale
        string tagline
        text bio
    }
    SOCIAL_LINK {
        uuid id PK
        string label
        string url
        int position
    }

    TIMELINE_ENTRY {
        uuid id PK
        string type "EDUCATION | EXPERIENCE | MILESTONE"
        date start_date
        date end_date "nullable"
        int position
    }
    TIMELINE_TRANSLATION {
        uuid entry_id FK
        string locale
        string title
        text description
    }

    CV_DOCUMENT {
        uuid id PK
        string locale
        string file_path
        boolean is_current
        int download_count
        timestamp uploaded_at
    }

    PROJECT {
        uuid id PK
        string slug UK
        string status "IN_PROGRESS | RELEASED | ARCHIVED"
        string category "WEB | MOBILE | BACKEND | OTHER"
        date release_date
        int hours_spent
        string demo_url
        string repo_url
        string trailer_url
        boolean published
        timestamp created_at
    }
    PROJECT_TRANSLATION {
        uuid project_id FK
        string locale
        string title
        string summary
        text description
    }
    TAG {
        uuid id PK
        string name UK
        string kind "TECH | TOPIC"
    }
    PROJECT_TAG {
        uuid project_id FK
        uuid tag_id FK
    }
    MEDIA {
        uuid id PK
        uuid project_id FK
        string type "IMAGE | VIDEO"
        string path_or_url
        string caption
        int position
        boolean is_cover
    }

    OFFER {
        uuid id PK
        string slug UK
        int base_price_cents
        string currency
        int delivery_days
        boolean published
    }
    OFFER_TRANSLATION {
        uuid offer_id FK
        string locale
        string title
        string summary
        text description
    }
    OFFER_OPTION {
        uuid id PK
        uuid offer_id FK
        int price_cents
        int position
    }
    OFFER_OPTION_TRANSLATION {
        uuid option_id FK
        string locale
        string title
        string description
    }

    QUOTE_REQUEST {
        uuid id PK
        string name
        string email
        text message
        uuid offer_id FK "nullable"
        string status "NEW | IN_PROGRESS | ACCEPTED | DECLINED"
        string locale
        timestamp created_at
    }
    QUOTE_REQUEST_OPTION {
        uuid quote_id FK
        uuid option_id FK
    }

    REVIEW {
        uuid id PK
        string author_name
        string author_email "privé, jamais exposé"
        int rating "1 à 5"
        text comment
        uuid project_id FK "nullable"
        uuid offer_id FK "nullable"
        string status "PENDING | APPROVED | REJECTED"
        timestamp created_at
    }

    PROFILE ||--o{ PROFILE_TRANSLATION : traduit
    TIMELINE_ENTRY ||--o{ TIMELINE_TRANSLATION : traduit
    PROJECT ||--o{ PROJECT_TRANSLATION : traduit
    PROJECT ||--o{ MEDIA : contient
    PROJECT ||--o{ PROJECT_TAG : a
    TAG ||--o{ PROJECT_TAG : classe
    OFFER ||--o{ OFFER_TRANSLATION : traduit
    OFFER ||--o{ OFFER_OPTION : propose
    OFFER_OPTION ||--o{ OFFER_OPTION_TRANSLATION : traduit
    OFFER ||--o{ QUOTE_REQUEST : concerne
    QUOTE_REQUEST ||--o{ QUOTE_REQUEST_OPTION : choisit
    OFFER_OPTION ||--o{ QUOTE_REQUEST_OPTION : sélectionnée
    PROJECT ||--o{ REVIEW : "reçoit (option)"
    OFFER ||--o{ REVIEW : "reçoit (option)"
```

Notes de conception :
- `PROFILE` est un **singleton** (une seule ligne) ; `PROFILE_TRANSLATION` a pour clé `(profile_id, locale)`.
- Les prix sont en **centimes** (entiers) : jamais de `Float` pour l'argent.
- `author_email` des avis n'est **jamais** renvoyé par l'API publique.
- Index à prévoir : `project.slug`, `offer.slug`, `(project_id, tag_id)`, `review.status`, `quote_request.status`.

## 5. Diagramme de classes (exemple de couches — domaine « Projet »)

```mermaid
classDiagram
    class ProjectRoutes {
        +Route.projectRoutes()
    }
    class ProjectService {
        -ProjectRepository repo
        +list(filter, locale) List~ProjectSummaryDto~
        +get(slug, locale) ProjectDetailDto
        +create(cmd) Project
        +update(id, cmd) Project
        +delete(id)
    }
    class ProjectRepository {
        <<interface>>
        +findPublished(filter) List~Project~
        +findBySlug(slug) Project?
        +insert(project) Project
        +update(project) Project
        +delete(id)
    }
    class ExposedProjectRepository
    class Project {
        +UUID id
        +String slug
        +ProjectStatus status
        +ProjectCategory category
        +LocalDate releaseDate
        +Int hoursSpent
        +Map~Locale, ProjectText~ texts
        +List~Tag~ tags
        +List~Media~ media
    }
    class ProjectStatus {
        <<enumeration>>
        IN_PROGRESS
        RELEASED
        ARCHIVED
    }

    ProjectRoutes --> ProjectService
    ProjectService --> ProjectRepository
    ExposedProjectRepository ..|> ProjectRepository
    ProjectService ..> Project
    Project --> ProjectStatus
```

## 6. Séquence — Demande de devis

```mermaid
sequenceDiagram
    actor V as Visiteur
    participant F as Front (Next.js)
    participant R as QuoteRoutes
    participant S as QuoteService
    participant DB as QuoteRepository
    participant M as EmailSender

    V->>F: Remplit le formulaire (offre + options)
    F->>R: POST /api/v1/quotes
    R->>R: Rate limit + honeypot + validation
    alt entrée invalide
        R-->>F: 400 + détail des erreurs
    else valide
        R->>S: create(QuoteCommand)
        S->>DB: vérifie offre/options, enregistre (NEW)
        DB-->>S: QuoteRequest
        S->>M: notifie Nathan (asynchrone)
        S-->>R: id du devis
        R-->>F: 201 Created
        F-->>V: Confirmation
    end
```

## 7. Séquence — Connexion admin et action protégée

```mermaid
sequenceDiagram
    actor A as Admin
    participant F as Front /admin
    participant R as AuthRoutes
    participant S as AuthService
    participant DB as AdminRepository
    participant P as Routes protégées

    A->>F: email + mot de passe
    F->>R: POST /api/v1/admin/auth/login
    R->>S: authenticate(email, password)
    S->>DB: findByEmail
    DB-->>S: AdminUser (hash)
    S->>S: vérifie le hash
    alt identifiants faux
        S-->>R: échec
        R-->>F: 401
    else ok
        S-->>R: JWT signé (exp. courte)
        R-->>F: 200 { token }
    end
    F->>P: PATCH /admin/reviews/{id} (Authorization: Bearer)
    P->>P: valide signature + expiration
    P-->>F: 200
```

## 8. États — Devis et avis

```mermaid
stateDiagram-v2
    direction LR
    [*] --> NEW: soumission
    NEW --> IN_PROGRESS: Nathan répond
    IN_PROGRESS --> ACCEPTED
    IN_PROGRESS --> DECLINED
    NEW --> DECLINED
    ACCEPTED --> [*]
    DECLINED --> [*]
```

```mermaid
stateDiagram-v2
    direction LR
    [*] --> PENDING: soumission
    PENDING --> APPROVED: modération
    PENDING --> REJECTED: modération
    APPROVED --> REJECTED: retrait
    REJECTED --> [*]
```

## 9. Déploiement

```mermaid
flowchart TB
    DEV[Dev local<br/>docker compose up] -->|PR vers preprod| CI[GitHub Actions<br/>build + tests]
    CI -->|merge preprod| STG[Environnement preprod]
    STG -->|validation| PRD[Production - VPS]
    subgraph VPS
        direction LR
        CD[Caddy] --> API[ktor-api] --> PG[(postgres)]
        CD --> WEB[web]
        BK[backup cron pg_dump] --> PG
    end
    PRD --- VPS
```

> Cohérent avec tes règles Git : features → `preprod` → `main`.
