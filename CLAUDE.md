# Project: Cairo (New Cairo) Restaurant Recommendation Chatbot — NLP Course Project

Handoff from a planning chat. Read this first. Do not start building models yet; the current focus is the **data preparation pipeline, Stages 1-2 (done/in progress) and Stage 3+ (next)**.

## Team / constraints
- University CS students, limited time and budget. Course rules: everything allowed (LLM APIs, pretrained transformers, scraping).
- Geographic scope: **New Cairo only**.
- Deliverables to plan around: problem statement/objectives, dataset collection + description, preprocessing, models, evaluation, error analysis, demo, GitHub repo.
- Course requirements (as summarized earlier in the chat, VERIFY against `NLP_Project_Description_1.pdf`): 5,000+ instances; 4+ EDA analyses; 4+ preprocessing steps; 2 traditional + 2 embedding representations; 3+ models (baseline + advanced); 4+ experiments; 4+ metrics; comparison table; 20+ error-analysis examples.

## Project idea
Chatbot that takes natural-language questions ("best pizza in Fifth Settlement") and returns top-5 restaurants with: name/location, cuisine, ranking score, rating, frequent positive/negative opinions from reviews, phone, Google Maps link.

## Chosen NLP task
- **Primary: review sentiment classification (pos/neu/neg), extended to aspect-based sentiment** (food, service, price, ambiance, delivery/wait).
- Secondary/optional: query understanding (extract cuisine + area) with rules/gazetteer first.
- Plan: classical baseline (TF-IDF + Logistic Regression / SVM / NB) then an advanced model (AraBERT/MARBERT/XLM-R). Do not default to deep learning.
- Languages: Arabic (Egyptian dialect) + English; Arabizi = stretch goal / error-analysis category.
- Chatbot = retrieval/ranking demo layer, not generative (avoid hallucinated phone numbers).

## Data strategy (decided)
- **Restaurant metadata + reviews: Google Maps via Apify scrapers** (Google Places API gives only 5 reviews/place and restricts storage, so not usable for a corpus). Facebook skipped (login walls, low yield).
- Optional supplements: Talabat Egypt reviews; public Arabic datasets for transfer experiments (ArRestReviews: 8,364 qaym.com reviews, license unknown; Mendeley Talabat dataset, 3-class, annotated). Check licenses; they are not New Cairo data.
- Terms caveat: scraping Google Maps violates Google's ToS even though the course allows it. Keep scale small, **do not publish raw reviews in the public repo** (publish scripts, schema, small sample, annotation guidelines), drop/hash reviewer identity, document the method in a data card.
- Weak labels from stars (1-2 neg, 3 neu, 4-5 pos); hand-annotate a **gold subset of 500-1,000 reviews** (sentiment + aspects), report Cohen's kappa. Split **by restaurant**, not by review. Gold set = untouched test set.
- Targets: 300-500 restaurants, 15-20k raw reviews, 8-10k usable text reviews.

## Apify actors used
- Stage 1 (places): `compass/crawler-google-places` — inputs: `searchStringsArray`, `locationQuery` (ONE location per run), `maxCrawledPlacesPerSearch`, `language`, `skipClosedPlaces`, `maxReviews`=0 here. ~$4/1,000 places. A single search is capped near ~120 results, hence many narrow searches (cuisine x district).
- Stage 2 (reviews): `compass/google-maps-reviews-scraper` — takes place URLs or place IDs; ~$0.30/1,000 reviews; set Reviews origin = **Google**; date filtering only works with sort = Newest. Output fields of interest: `reviewId, placeId, text, stars, publishedAtDate, originalLanguage, reviewOrigin, likesCount, responseFromOwnerText, scrapedAt` (+ `reviewerId` to hash). Drop names/URLs/photos. Keep original `text`, ignore `textTranslated` for training.
- Free Apify plan = $5/month credit, no rollover. Field names above are from docs/examples; confirm in each actor's Input tab.

## Search terms used (Stage 1)
Core EN: restaurant, pizza restaurant, burger restaurant, Italian restaurant, Egyptian restaurant, seafood restaurant, grill restaurant, shawarma, sushi restaurant, fast food restaurant, breakfast restaurant, cafe. Core AR: مطعم، مطعم بيتزا، مطعم برجر، مطعم إيطالي، مطعم مأكولات مصرية، مطعم سمك، مطعم مشويات، شاورما، مطعم سوشي، وجبات سريعة، مطعم فطار، كافيه. Extended lists (Chinese, Asian, Thai, Indian, Mexican, Lebanese, Syrian, Turkish, steakhouse, chicken, koshary, fatteh, crepe, pasta, healthy, dessert, bakery + Arabic equivalents) held back for budget. Language setting = English (categories come back in English; review text stays in original language).

## Current data state
- Ran Stage 1 for **Fifth Settlement** only: `fifth-settlement-discovery.json` -> 224 rows, 224 unique `placeId`, 0 closed. (Terms actually used included both English and Arabic: مطعم, بيتزا, برجر, فطار, etc.) 205 rows have city "New Cairo 1". Coordinates all within lat 29.99-30.02, lng 31.41-31.45.
- Reviews count per place: median 44; 143 places have >=10, 120 have >=30, 85 have >=100.
- Filter applied (`restaurants_raw.csv`, column `keep`): reviewsCount >= 30, not closed, inside bbox, not café/bakery/dessert/etc. -> **96 restaurants kept**, 34 excluded as cafés/non-food (column `is_cafe_or_nonfood`; reversible).
- `review_input_urls.txt` / `review_input_placeids.txt` = the 96 selected places, ready to paste into the Reviews Scraper. With cap 50 reviews/place expect up to ~4,700 raw reviews (~5,500 at cap 60); cost ~ $1.5-2.
- A `reviews_scraper_input.json` was generated but its field names are unverified guesses; prefer the actor's form.
- Stage 2 NOT yet run. Next: pilot with 5 URLs, check text rate, language mix (`originalLanguage`), star distribution, Arabizi share; then run all 96.

## Known concerns
- 96 restaurants is below the 300-500 target: repeat Stage 1 for the other districts (First/Third/Fourth Settlement, Rehab, Katameya, Mivida, Golden Square, ...), verify they return different places, merge by `placeId`.
- Many Google reviews have no text; plan ~2-3x raw collection. Ratings skew to 5 stars -> class imbalance (handling it counts as an experiment).
- Cap reviews per place so big restaurants do not dominate; consider an extra "most relevant" pass of ~20 reviews/place deduped on `reviewId`.
- Keep raw exports immutable in `data/raw/` with run date; keep a collection log (actor, version, dates, inputs, counts).

## Planned pipeline (each stage = separate script)
discover -> collect -> raw store -> clean -> entity resolution -> label -> split -> validate & document.
- Clean: drop empty/rating-only/very short (<~3 words) reviews, exact + near-duplicate removal, language ID per review (ar/en/mixed/Arabizi flag), Arabic normalization (alef variants, diacritics, tatweel, repeated letters), emoji handling (keep text-converted copy), strip URLs/phones/names. Keep both `text_raw` and `text_clean`.
- Entity resolution: each `placeId` = one branch; group into brands via fuzzy name match (RapidFuzz) + distance check; map Google categories to ~10-15 cuisine classes.
- Suggested tables: `restaurants`, `brands`, `reviews` (review_id, restaurant_id, source, text_raw, text_clean, language, stars, date, weak_label, gold_label, aspects, split).

## What to do next in Claude Code
1. Create repo structure: `data/raw`, `data/interim`, `data/processed`, `src/` (one script per stage), `docs/` (data card, annotation guidelines).
2. Put the three files from this chat in `data/` and write a loader for the Apify JSON.
3. After the Stage 2 pilot output exists, write Stage 3 cleaning (language ID, Arabic normalization, dedup) and a pilot EDA notebook.
4. Do not write model code until the cleaned dataset and gold-annotation plan exist.
