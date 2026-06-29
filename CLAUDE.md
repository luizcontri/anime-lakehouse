# Anime Lakehouse — Project Context

> Open-source data platform for collecting, consolidating and serving anime metadata and user interaction data at scale. Built on a medallion architecture (bronze → silver → gold) with MinIO, Apache Spark, Airflow and Kubernetes, designed to power recommendation models and analytical workloads.

---

## Architecture Overview

### Medallion layers

| Layer | Purpose | State |
|---|---|---|
| **Bronze** | Raw ingestion — data exactly as received from sources, no transformations | In design |
| **Silver** | Deduplication, consolidation, canonical IDs, business rules | Planned |
| **Gold** | Feature tables ready for ML training and analytical workloads | Planned |

### Infrastructure stack

- **Storage** — MinIO (S3-compatible object storage)
- **Orchestration** — Apache Airflow
- **Processing** — Apache Spark via Spark Operator on Kubernetes
- **IaC / GitOps** — Infrastructure as Code + ArgoCD
- **Format** — Parquet + Delta Lake (TBD)

---

## Data Sources — Bronze Layer

### Priority order for deduplication
When the same anime exists in multiple sources, fields are populated following this priority:
1. **MyAnimeList** — anime base data, ratings, reviews, user lists
2. **AniList** — tags, staff, relations, supplementary scores
3. **AniDB / Kitsu** — episode-level metadata, technical details

### Source inventory

#### MyAnimeList (MAL) — OAuth2 API
Rate limit: ~1 req/s, no bulk endpoint.

| Bronze table | Key fields |
|---|---|
| `bronze.mal_anime` | id, title, type, status, episodes, duration, start/end date, mean, rank, popularity, num_scoring_users, nsfw, genres[], studios[], related_anime[], recommendations[], synopsis, _raw_json |
| `bronze.mal_user_animelist` | user_name (hashed), anime_id, list_status, score, episodes_watched, is_rewatching, rewatch_count, tags, priority, start_date, finish_date |
| `bronze.mal_anime_reviews` | id, anime_id, user_name (hashed), score, body, num_helpful, is_spoiler, post_date |
| `bronze.mal_anime_statistics` | anime_id, watching, completed, on_hold, dropped, plan_to_watch, num_list_users, num_scoring_users, score_1…score_10 (histogram) |

#### AniList — GraphQL public API
Rate limit: 90 req/min. Supports batch pagination. Exposes `idMal` field (MAL ID) natively — critical for cross-source deduplication.

| Bronze table | Key fields |
|---|---|
| `bronze.anilist_media` | id, idMal, title{romaji/native/english}, type, format, status, episodes, duration, startDate, endDate, season, seasonYear, averageScore, meanScore, popularity, favourites, trending, genres[], tags[]{name, rank%, isAdult}, studios[], staff[], relations[], recommendations[], countryOfOrigin, isAdult, _raw_json |
| `bronze.anilist_user_medialist` | userId (hashed), mediaId, status, score, progress, repeat, priority, startedAt, completedAt, notes, customLists |
| `bronze.anilist_user_stats` | userId (hashed), count, meanScore, minutesWatched, genreStats[], tagStats[], staffStats[], studioStats[] |
| `bronze.anilist_reviews` | id, mediaId, userId (hashed), score, body, summary, rating, ratingAmount, createdAt, updatedAt |

#### Jikan — Unofficial MAL REST wrapper
Rate limit: 3 req/s, 24h cache. Useful for data MAL API doesn't expose directly.

| Bronze table | Key fields |
|---|---|
| `bronze.jikan_anime_full` | mal_id, titles[], type, relations[]{type, entry[]}, external_links[], streaming[], licensors[], opening_themes[], ending_themes[] |
| `bronze.jikan_anime_characters` | anime_id, character_id, name, role (main/supporting), voice_actors[]{person, language} |
| `bronze.jikan_top_anime` | rank, mal_id, score, type, filter, _snapshot_date |

#### AniDB — Monthly XML dump
Download once per month; no polling. Contains `anidb_id_map` which is the most complete cross-source ID mapping available — **ingest this first before any other source**.

| Bronze table | Key fields |
|---|---|
| `bronze.anidb_anime` | aid, type, episodecount, startdate, enddate, titles[]{lang, type}, ratings{temp, perm, review}, tags[]{name, weight, is_spoiler}, creators[]{role, name}, _dump_date |
| `bronze.anidb_episodes` | eid, aid, epno, type (episode/special/credit/trailer), length, airdate, title[]{lang}, rating, votecount |
| `bronze.anidb_id_map` | anidb_id, mal_id, anilist_id, kitsu_id, ann_id — **cross-reference table, foundation of silver deduplication** |

#### Kitsu — JSON:API
Key differentiator: `episodes.seasonNumber` — the only structured field that maps episodes to seasons when MAL lists each season as a separate anime entry.

| Bronze table | Key fields |
|---|---|
| `bronze.kitsu_anime` | id, slug, titles{}, episodeCount, episodeLength, averageRating, ratingFrequencies{}, userCount, favoritesCount, ageRating, ageRatingGuide, startDate, endDate, status |
| `bronze.kitsu_episodes` | id, anime_id, number, **seasonNumber**, title, synopsis, length, airdate, thumbnail_url |
| `bronze.kitsu_mappings` | kitsu_id, external_site, external_id |

#### MAL Forums — Scraping
| Bronze table | Key fields |
|---|---|
| `bronze.mal_forum_posts` | topic_id, post_id, anime_id (from URL), episode_number (from topic title), username_hash, body_raw (HTML), post_date, edit_date, _scraped_at, _page_url |

#### Reddit r/anime — Reddit API / Pushshift
Rate limit: 60 req/min with OAuth.

| Bronze table | Key fields |
|---|---|
| `bronze.reddit_episode_threads` | post_id, subreddit, title_raw, anime_title_parsed, episode_parsed, score, upvote_ratio, num_comments, awards, created_utc |
| `bronze.reddit_comments` | comment_id, post_id, parent_id, author_hash, body_raw, score, controversiality, created_utc, edited |

#### Supplementary sources (lower priority)
- `bronze.ann_anime` — AnimeNewsNetwork: Bayesian avg scores, detailed staff/cast
- `bronze.anisearch_anime` — European user base scores, certifications
- `bronze.livechart_calendar` — streaming platform availability, broadcast schedule

---

## Bronze Layer Rules

Every bronze table **must** include these metadata columns — no exceptions:

| Column | Type | Purpose |
|---|---|---|
| `_source` | varchar | Source system name (e.g. `mal`, `anilist`) |
| `_scraped_at` | timestamp | When this record was collected |
| `_raw_json` | text/json | Complete original payload — never modified |
| `_pipeline_run_id` | uuid | Airflow DAG run ID for lineage and rollback |

**Zero transformations in bronze.** No type casting, no field renaming, no nulls filled. Named columns beside `_raw_json` are projections for exploratory queries only — the raw JSON is the source of truth.

---

## Canonical Data Model — Silver (planned)

The silver layer introduces canonical UUIDs and cross-source deduplication. Key concept: one `anime_id` in the lake may map to multiple external IDs across sources.

### Core entities

| Entity | Description |
|---|---|
| `anime` | Canonical anime record with lake-generated UUID |
| `user` | Anonymized user profile (UUID derived from hash of external ID) |
| `rating` | Numeric score per user × anime |
| `review` | Long-form text review per user × anime |
| `user_list_entry` | Watch history entry (watching / completed / dropped / plan_to_watch) |
| `episode` | Per-episode metadata |
| `person` | Director, composer, voice actor |
| `studio` | Production company |
| `genre` / `tag` | Classification lookup |
| `anime_external_id` | Cross-source ID mapping table |
| `anime_relation` | Sequel, prequel, side story, adaptation relationships |
| `anime_staff` | N:N join between anime and person |
| `recommendation` | User-generated "if you liked X try Y" pairs (from MAL) |

### Deduplication strategy

1. Use `anidb_id_map` as the primary cross-reference (AniDB → MAL → AniList → Kitsu)
2. AniList's `idMal` field and Kitsu's `mappings` table provide additional anchors
3. Title-based fuzzy matching as fallback, with `confidence` score stored for manual review
4. `anime_external_id.is_primary` flags the authoritative source for each field
5. Fields populated following source priority order (MAL → AniList → AniDB/Kitsu)

### Season aggregation (key business rule)

MAL lists each season of a series as a separate anime entry. For the recommendation model, seasons of the same series should be treated as a single logical work. The silver layer will introduce a `series_id` grouping concept, using `anime_relation` (sequel/prequel chains) and Kitsu's `seasonNumber` to build these clusters. Individual `anime_id` records are preserved — `series_id` is an additional grouping key.

### User privacy rules

- Never store raw usernames — always hash before writing to bronze
- `user_id` in silver is a UUID derived from `sha256(source + external_user_id)`
- Store `join_year` only (not full join date)
- No PII fields under any circumstances

---

## Gold Layer — ML Feature Tables (planned)

Target: recommendation model. Approach: collaborative filtering + content-based hybrid.

Signals to aggregate:
- User × anime interaction matrix (ratings, list status, completion rate)
- Positive signals: completed, high score, favorited, rewatched
- Negative signals: dropped, low score
- Content features: genres, tags, studio, staff, season/year, source material, duration
- Temporal features: recency of watch, trending score at time of airing
- Social features: Reddit/forum engagement volume per episode

Series-level aggregation applies here — recommendation targets `series_id`, not individual season `anime_id`.

---

## Initial repository structure

The structure below was defined for a 100% local Kubernetes environment, running on the user's own machine, without notebooks, with a focus on declarative infrastructure, Spark pipelines, and Airflow DAGs. The intent is that this base can evolve toward a future cloud migration without requiring a complete restructure:

```text
anime-lakehouse/
├── dags/
├── docs/
├── infra/
│   └── kubernetes/
│       ├── base/
│       ├── overlays/
│       └── charts/
├── scripts/
├── src/
│   ├── pipelines/
│   │   ├── bronze/
│   │   ├── silver/
│   │   └── gold/
│   └── shared/
└── tests/
```

- `infra/kubernetes/base/` holds manifests for MinIO, Airflow, Spark Operator, ingress, and shared resources.
- `infra/kubernetes/overlays/` holds environment-specific variations such as development and production.
- `infra/kubernetes/charts/` can hold reusable Helm charts for ecosystem components.
- `dags/` holds Airflow DAGs and orchestration files.
- `src/` groups Python/Spark code for the bronze, silver, and gold layers.
- `scripts/` holds operational utilities and setup scripts.
- `tests/` will hold unit, integration, and regression tests.

This project will not use notebooks; exploration and validation are done via scripts, DAGs, and applications running on the local cluster.

---

## Project Decisions Log

| Decision | Rationale |
|---|---|
| MinIO over S3 | Full open-source stack, self-hosted |
| Airflow for orchestration | Mature ecosystem, good Spark integration |
| Spark Operator on K8s | Scalable processing, consistent with IaC approach |
| ArgoCD for GitOps | Declarative infra, version-controlled deployments |
| Medallion architecture | Clean separation of raw vs curated vs ML-ready data |
| `_raw_json` on every bronze table | Allows schema evolution without data loss |
| Hash usernames in bronze | Privacy by design from the first layer |
| AniDB dump as first ingest | Provides cross-source ID map that unblocks all deduplication |
| `series_id` concept in silver | Avoids recommending isolated seasons; aggregates franchise data |
| Sealed Secrets for all project secrets | Ensures secrets are encrypted, declarative, GitOps-friendly and reproducible across environments |
| All applications installed via Argo CD | Enforces the declarative GitOps model; nothing is applied manually outside of the bootstrap steps |
| English only for all documentation and code comments | Ensures consistency across the repository and makes the project accessible to any contributor |