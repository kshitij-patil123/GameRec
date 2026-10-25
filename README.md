# GameRec

**An NLP-based Discord gaming community analysis and hybrid personalized game recommendation system.**
*Engineering mini-project: Text / Web / Social Media Analysis + Recommendation Systems.*

> Honest status: the system is built and tested (140 backend tests, 19 frontend tests, PostgreSQL 16 verified), but it has only been run on **synthetic demo data**. Recommendation and sentiment results on real data are **pending execution**. Nothing in this repository reports invented numbers.

## Overview
Traditional game recommenders mostly use ratings and metadata. GameRec adds **community intelligence extracted from text**: what gaming communities are discussing, how they feel about each game and each aspect of it (story, graphics, combat, performance, ...), and what is trending. It combines that with the user's own taste to rank games and **explains every recommendation**.

*Community understanding, then user understanding, then recommendation.*

## Problem statement
Rating- and metadata-based recommenders ignore the rich opinion text communities produce, cannot say *why* a game fits a particular person, and struggle with new users and new games. GameRec asks: can sentiment, topics, aspect-level opinion and trends mined from community text make personalized recommendations more context-aware, and can that be shown with a defensible evaluation?

## Objectives
1. Ingest games, community text and ratings through an interchangeable, privacy-respecting layer.
2. Extract sentiment, keywords, topics, aspect-level sentiment and trends from text.
3. Build a user preference profile and three recommenders: content-based, collaborative, and an interpretable weighted hybrid that uses community signals.
4. Explain each recommendation using only evidence that supports it.
5. Evaluate with Precision/Recall/F1/NDCG@K, baselines, ablations, paired statistics, and without fabricating results.

## Architecture
![Architecture](docs/diagrams/architecture.png)

Pipeline: data sources, ingestion, cleaning and NLP (sentiment, keywords, topics, aspects, trends), community intelligence, user profile, content / collaborative / hybrid ranking, explanations, REST API, dashboard. Other diagrams (data flow, ER, use case, class, sequence) are in `docs/diagrams/` and the report.

## Features
- **Ingestion** with `demo` / `public` / `discord_export` sources, schema validation, game linking by name, author-ID hashing; real and synthetic data are labeled and shown in the UI.
- **NLP:** two text views (light for VADER/transformers, aggressive for classical models), VADER, TF-IDF + Logistic Regression, optional transformer; TF-IDF keywords (game, genre, emerging); NMF topics with trends; rule-based aspect sentiment over 10 aspects.
- **Scores:** windowed trend score, project-defined Community Intelligence Score with confidence.
- **Recommenders:** content (TF-IDF + cosine), item-item CF, user profile, configurable hybrid, cold start for new users and games, evidence-based explanations with caveats.
- **API:** 23 paths (FastAPI) with validation and error handling. **Dashboard:** six React pages and ten chart types.
- **Evaluation harness** with baselines, ablations, validation-tuned weights, cold-start experiment and bootstrap statistics.

## Technology stack
Python 3.11+, FastAPI, SQLAlchemy, PostgreSQL 16 (SQLite for quick local runs), pandas, scikit-learn, NLTK, VADER, optional Hugging Face Transformers; React 19, Vite, Tailwind CSS 4, Plotly; Docker Compose; pytest, Vitest.

## Dataset description
Three standard tables (games, messages, ratings); see [`docs/datasets.md`](docs/datasets.md) and [`docs/data_dictionary.md`](docs/data_dictionary.md). A seeded generator creates a **synthetic** demo set (24 fictional games, template-based messages, ratings). It is not real Discord or Steam data and must not be used as evidence.

## Installation
```bash
git clone <your repo> && cd gamerec
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # edit passwords and USER_HASH_SALT
```
Optional transformer sentiment model: `pip install -r requirements-transformer.txt`.

## Environment variables (`.env`)
| Variable | Purpose |
|---|---|
| `DATA_SOURCE` | `demo`, `public` or `discord_export` |
| `DATABASE_URL` | SQLAlchemy URL; default in `.env.example` is PostgreSQL; unset for a local SQLite file |
| `POSTGRES_USER/PASSWORD/DB` | used by Docker Compose for the database container |
| `CORS_ORIGINS` | comma-separated allowed browser origins |
| `USER_HASH_SALT` | salt for hashing Discord author IDs |
| `ADMIN_TOKEN` | enables `POST /system/refresh` when set |
| `VITE_API_URL` (frontend `.env`) | API base URL for the dashboard |

## Database setup
```bash
docker compose up -d db           # PostgreSQL 16 (published on localhost only)
python scripts/init_db.py         # create the 16 tables
```
Schema: users, games, genres, tags, game_genres, game_tags, messages, sentiment_results, aspect_sentiments, topics, game_topics, message_topics, ratings, user_preferences, game_scores, recommendations (ER diagram in `docs/diagrams/er_diagram.png`, generated from the models by `scripts/generate_er_diagram.py`).

## Run it
```bash
python scripts/generate_demo_data.py        # only if data/sample is missing
python scripts/run_pipeline.py demo         # load, sentiment, aspects, topics, trend + community scores
cd backend && uvicorn app.main:app --reload # API + docs at http://localhost:8000/docs
cd frontend && npm install && npm run dev   # dashboard at http://localhost:5173
```
Everything in Docker: `docker compose up --build` (web at http://localhost:8080; the API start-up script loads data only if the database is empty). Docker was **not** available when this was built, so run it once before your demo.

## ML pipeline
| Step | Command / module | Output |
|---|---|---|
| Load | `scripts/load_data.py` | games, messages, ratings (with `source_type`) |
| Sentiment | `scripts/run_sentiment.py vader` | `sentiment_results`, per-game positive/neutral/negative % |
| Keywords, topics, aspects | `scripts/run_analysis.py all` | keywords JSON, `topics`, `aspect_sentiments` |
| Trend + community score | `scripts/run_trends.py` | `game_scores` |
| Everything above | `scripts/run_pipeline.py` | |
Details: [`nlp_design.md`](docs/nlp_design.md), [`text_analysis.md`](docs/text_analysis.md), [`trends_and_content.md`](docs/trends_and_content.md).

## Recommendation methodology
`final = sum(w_c * norm_c) / sum(w_c)` over six components, each min-max normalized across candidates:

| Component | Default weight |
|---|---|
| Content similarity | 0.30 |
| Collaborative score | 0.25 |
| User preference match | 0.15 |
| Community sentiment | 0.15 |
| Trend | 0.10 |
| Aspect match | 0.05 |

Weights are configurable per request and are **starting values, not tuned truths**. Unavailable components (for example no CF for a new user) are dropped and the rest renormalized; with no signal at all the system falls back to popular/trending, then to catalog rating. `match %` is relative to the other candidates, not a probability. Full description: [`recommenders.md`](docs/recommenders.md).

## Evaluation methodology
Per-user chronological split (60/20/20), relevant = rating of at least 4, Precision/Recall/F1/NDCG/hit-rate @K, 95% bootstrap CIs and paired bootstrap differences, Random and Popularity baselines, ablations A to D, a hybrid without community signals, weights tuned on validation only, and a cold-start experiment. The harness was validated with a positive control (planted signal is found) and a null control (noise shows no gain, i.e. no leakage).
`python ml/evaluation/run_evaluation.py` after loading real data. **Results on real data: pending execution.** Details and caveats: [`evaluation.md`](docs/evaluation.md).

| Model | Precision@10 | Recall@10 | NDCG@10 |
|---|---|---|---|
| Content-only | Result pending execution | Result pending execution | Result pending execution |
| Collaborative-only | Result pending execution | Result pending execution | Result pending execution |
| Content + collaborative | Result pending execution | Result pending execution | Result pending execution |
| Full hybrid with community intelligence | Result pending execution | Result pending execution | Result pending execution |

## Screenshots
Add real screenshots after you run the app (the pages were not visually checked during the build):
`docs/screenshots/home.png`, `explorer.png`, `game-detail.png`, `for-you.png`, `community.png`, `profile.png`.

## Limitations
- Only synthetic data has been run; no real-data results exist yet.
- Aspect sentiment is rule-based (fixed aspect list, one score per clause, no sarcasm); sentiment accuracy is unmeasured until you label real messages.
- The default hybrid and community-score weights are not tuned; tuned weights must come from validation on real data.
- A small catalog makes top-K metrics unreliable; the harness warns when this happens.
- No authentication, HTTPS or rate limiting: local demo only.
- Not verified: pages in a real browser, `docker compose up`, the transformer model.

## Future scope
Matrix factorization / neural CF; fine-tuned ABSA transformer; embeddings or KeyBERT for keywords and content similarity; BERTopic; learning hybrid weights with a learning-to-rank model; implicit feedback (playtime); authentication and per-user privacy controls; incremental updates instead of rebuilds; a Discord bot for live ingestion with consent tooling.

## Ethical and privacy considerations
Collect Discord data only through a bot or export you are authorized to use (your own server or written permission) and tell members what is collected. Author IDs are hashed with a salt before storage; the API returns aggregates, never raw messages or identifiers. Synthetic data is always labeled. Recommendation explanations show caveats and uncertainty rather than hiding them. Community scores are project-defined metrics, not measures of game quality, and sentiment models can be biased by slang, sarcasm and community norms.

## Documentation index
[`docs/api.md`](docs/api.md), [`frontend.md`](docs/frontend.md), [`integration.md`](docs/integration.md), [`testing.md`](docs/testing.md), [`evaluation.md`](docs/evaluation.md), [`datasets.md`](docs/datasets.md), and the academic report in `docs/report/`.
