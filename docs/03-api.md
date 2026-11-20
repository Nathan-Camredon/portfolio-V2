# 03 — Contrat d'API (v1)

Base : `/api/v1` — JSON, UTF-8. Langue : `?lang=fr|en` (défaut `fr`).
Auth admin : en-tête `Authorization: Bearer <jwt>`.

## Format d'erreur unique

```json
{
  "error": "VALIDATION_FAILED",
  "message": "Le champ email est invalide",
  "details": [{ "field": "email", "issue": "invalid_format" }],
  "timestamp": "2026-11-20T21:00:00Z"
}
```

| HTTP | `error` | Cas |
|------|---------|-----|
| 400 | `VALIDATION_FAILED` | Entrée invalide |
| 401 | `UNAUTHORIZED` | Token absent/invalide |
| 404 | `NOT_FOUND` | Ressource inconnue |
| 409 | `CONFLICT` | Slug déjà pris |
| 413 | `PAYLOAD_TOO_LARGE` | Upload trop gros |
| 429 | `RATE_LIMITED` | Trop de requêtes |
| 500 | `INTERNAL_ERROR` | Erreur serveur (rien de sensible exposé) |

## Public

| Méthode | Route | Description | Réponse |
|---------|-------|-------------|---------|
| GET | `/health` | Vivacité + BD | 200 |
| GET | `/profile` | Profil + liens traduits | 200 |
| GET | `/timeline` | Entrées triées | 200 |
| GET | `/cv` | Métadonnées du CV courant | 200 |
| GET | `/cv/download` | Télécharge le PDF (incrémente le compteur) | 200 `application/pdf` |
| GET | `/projects` | Liste publiée. Filtres : `tag`, `status`, `category`, `q`, `page`, `size` | 200 paginé |
| GET | `/projects/{slug}` | Fiche complète | 200 / 404 |
| GET | `/tags` | Tous les tags | 200 |
| GET | `/offers` | Offres publiées + options | 200 |
| GET | `/offers/{slug}` | Détail offre | 200 / 404 |
| POST | `/quotes` | Demande de devis | 201 / 400 / 429 |
| GET | `/reviews` | Avis approuvés (paginé) | 200 |
| POST | `/reviews` | Soumettre un avis (`PENDING`) | 201 / 400 / 429 |

### Exemples

`GET /projects?lang=fr&tag=kotlin&status=RELEASED`

```json
{
  "items": [
    {
      "slug": "pokemon-api",
      "title": "API Pokémon",
      "summary": "Une API REST en Kotlin.",
      "status": "RELEASED",
      "category": "BACKEND",
      "releaseDate": "2026-03-01",
      "hoursSpent": 120,
      "tags": ["kotlin", "ktor"],
      "coverUrl": "/uploads/pokemon-api/cover.webp"
    }
  ],
  "page": 1, "size": 12, "total": 1
}
```

`POST /quotes`

```json
{
  "name": "Alice Martin",
  "email": "alice@example.com",
  "message": "Je voudrais un site vitrine pour mon cabinet.",
  "offerSlug": "site-vitrine",
  "optionIds": ["8f1c…", "a2b9…"],
  "locale": "fr",
  "website": ""
}
```
`website` est le **honeypot** : il doit rester vide, sinon la demande est ignorée silencieusement.

Réponse : `201 { "id": "…", "status": "NEW" }`

## Admin (JWT requis sauf login)

| Méthode | Route | Description |
|---------|-------|-------------|
| POST | `/admin/auth/login` | `{email, password}` → `{token, expiresAt}` |
| PUT | `/admin/profile` | Met à jour profil + traductions |
| POST | `/admin/profile/avatar` | Upload avatar (multipart) |
| CRUD | `/admin/social-links` | Liens du profil |
| CRUD | `/admin/timeline` | Entrées de timeline |
| GET/POST | `/admin/cv` | Liste / upload d'un CV (multipart) |
| PATCH | `/admin/cv/{id}/current` | Définir le CV courant |
| CRUD | `/admin/projects` | Projets (+ traductions, tags) |
| POST/PATCH/DELETE | `/admin/projects/{id}/media` | Médias d'un projet |
| CRUD | `/admin/tags` | Tags |
| CRUD | `/admin/offers` | Offres (+ traductions) |
| CRUD | `/admin/offers/{id}/options` | Options (DLC) |
| GET | `/admin/quotes` | Liste des devis (filtre `status`) |
| PATCH | `/admin/quotes/{id}` | Changer le statut |
| GET | `/admin/reviews` | Avis (filtre `status`) |
| PATCH | `/admin/reviews/{id}` | Approuver / rejeter |
| DELETE | `/admin/reviews/{id}` | Supprimer |

## Règles transverses

- **Pagination** : `page` (1-based), `size` (max 50).
- **Slugs** : `[a-z0-9-]+`, uniques, générés depuis le titre si absent.
- **Validation** (exemples) : `email` valide ; `message` 10 à 2000 caractères ; `rating` 1 à 5 ; `name` 2 à 80.
- **Uploads** : images `png/jpg/webp` ≤ 5 Mo ; CV `pdf` ≤ 5 Mo ; le type est vérifié sur le contenu, pas seulement l'extension ; nom de fichier regénéré (UUID).
- **Rate limit** : `POST /quotes` et `POST /reviews` limités par IP (ex. 5 / heure).
- **CORS** : seule l'origine du front est autorisée.
- **Cache** : `Cache-Control` + `ETag` sur les `GET` publics.
- **Documentation** : OpenAPI générée, Swagger UI sur `/docs` (désactivée en prod ou protégée).
