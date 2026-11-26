# 🎌 AniData Lab — Observatoire Anime/Manga

Pipeline de données de bout en bout autour du dataset Kaggle *Anime Recommendation Database 2020* : audit et nettoyage des données, enrichissement, indexation dans **Elasticsearch**, visualisation dans **Grafana** et orchestration avec **Apache Airflow**, le tout dans des conteneurs **Docker**.

Projet réalisé dans le cadre d'une formation Data / DevOps (2026).

## Architecture

```
CSV bruts ─► Nettoyage ─► Validation ─► Feature engineering ─► Export "gold" ─► Elasticsearch ─► Grafana
 (Kaggle)      (pandas)                    (weighted_score,       (CSV + NDJSON      (indexation       (dashboard
                                            drop_ratio, …)         versionnés)         bulk)            provisionné)
                         ▲
                         └── orchestré par Airflow (2 DAGs) ◄── scraper d'un mock-site (anidata-scraper)
```

| Brique | Rôle |
|---|---|
| **Airflow** (`airflow/`) | `dag1_etl_pipeline` : extract → clean → validate → transform → export gold → indexation ES. `dag2_data_injection` : lit des fichiers JSON/XML et déclenche le DAG 1 en lui passant les données. |
| **Scripts** (`airflow/scripts/`) | Modules réutilisables : `extract`, `clean`, `transform`, `export`, `load`. |
| **Notebooks** (`notebooks/`) | Étapes de data refinement : audit complet, audit visuel, nettoyage, feature engineering, validation, indexation. |
| **Elasticsearch + Logstash** (`elk/`) | Stockage et recherche ; Logstash en option pour indexer `anime.csv` à la demande. |
| **Grafana** (`grafana/`) | Datasource et dashboard `anidata-overview` provisionnés automatiquement. |
| **Scraper** (`anidata-scraper/`) | Package Python (BeautifulSoup) qui scrape un mock-site et produit un JSON consommable par Airflow. Livré avec des tests `pytest`. |

Le projet est dimensionné pour une machine de 8 Go de RAM (~2,4 Go pour la stack permanente).

## Démarrage

**Prérequis** : Docker Desktop (Compose v2), Python 3.10+, 8 Go de RAM, ~10 Go de disque.

```bash
# 1. Configuration
cp .env.example .env          # puis remplis les mots de passe et les clés (instructions dans le fichier)

# 2. Données : télécharger le dataset Kaggle et placer les CSV dans data/
#    https://www.kaggle.com/datasets/hernan4444/anime-recommendation-database-2020
#    -> data/anime.csv, data/anime_with_synopsis.csv, data/rating_complete.csv (~700 Mo)

# 3. Lancer la stack
./start.sh                    # Windows : start.bat   (ou : docker compose up -d)
```

| Service | URL | Identifiants |
|---|---|---|
| Grafana | http://localhost:3000 | `admin` / `GRAFANA_ADMIN_PASSWORD` de ton `.env` |
| Airflow | http://localhost:8080 | `admin` / `AIRFLOW_ADMIN_PASSWORD` de ton `.env` |
| Elasticsearch | http://localhost:9200 | API directe |

Indexer avec Logstash (optionnel) : `docker compose --profile ingest up logstash`

## Tests du scraper

```bash
cd anidata-scraper
pip install -r requirements-dev.txt
pytest
```

## Commandes utiles

```bash
docker compose ps                         # état des services
docker compose logs -f airflow-webserver  # logs d'un service
docker compose down                       # arrêt (données conservées)
docker compose down -v                    # arrêt + suppression des données
```

**Elasticsearch ne démarre pas sous Linux ?** `sudo sysctl -w vm.max_map_count=262144`
**Airflow « Database not initialized » ?** `docker compose down && docker compose up airflow-init && docker compose up -d`

## Stack

Python · pandas · Apache Airflow 2.8 · Elasticsearch · Logstash · Grafana · PostgreSQL · Docker Compose · BeautifulSoup · pytest
