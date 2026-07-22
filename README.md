# NutriMatch — AI-Powered Diet & Recipe Recommendation System

A content-based recipe recommender that matches recipes to a person's nutritional
targets (calories, fat, protein, etc.) and preferred ingredients, served through a
FastAPI backend and a Streamlit front end, with recipe photos auto-fetched from the
web.

## How it works

You give it a 9-value nutrition profile (calories, fat, saturated fat, cholesterol,
sodium, carbs, fiber, sugar, protein) plus any ingredients you want included. It
filters the recipe dataset down to matches containing those ingredients, scales the
nutrition features, and uses a cosine-similarity k-Nearest-Neighbors search to return
the closest-matching recipes — each with cook time, instructions, and an image.

## Project structure

```
AI-Project/
├── dataset.csv              # gzip-compressed recipe dataset (name, times, ingredients, nutrition, instructions)
├── model.py                 # core recommendation engine (scaling + kNN pipeline)
├── main.py                  # FastAPI backend exposing the /predict/ endpoint
├── recomendation.py         # client-side request generator that calls the backend API
├── front1.py                # Streamlit landing page / UI entry point
├── imagefinder.py           # scrapes a representative image for a recipe/search term
└── docs/                    # literature review + presentation slides
```

## What each file does

### `model.py` — the recommendation engine
- **`scaling(dataframe)`** — standardizes the 9 nutrition columns (columns 6–15:
  Calories → ProteinContent) with `StandardScaler` so no single nutrient dominates
  the distance calculation.
- **`nn_predictor(prep_data)`** — fits a cosine-similarity `NearestNeighbors` model
  on the scaled nutrition data.
- **`build_pipeline(neigh, scaler, params)`** — wraps the scaler and the neighbor
  search into a single scikit-learn `Pipeline`, so a new nutrition vector can be
  scaled and matched in one call.
- **`extract_ingredient_filtered_data(dataframe, ingredients)`** — builds a regex
  that requires *all* requested ingredients to appear in `RecipeIngredientParts`,
  and filters the dataset down to only matching recipes.
- **`extract_data(dataframe, ingredients)`** — thin wrapper that applies the
  ingredient filter (kept separate so more filters can be added later).
- **`apply_pipeline(pipeline, _input, extracted_data)`** — runs the user's nutrition
  vector through the fitted pipeline and returns the matching rows from the
  filtered dataset.
- **`recommend(dataframe, _input, ingredients, params)`** — the main entry point:
  filters by ingredients, checks there are enough candidates for the requested
  number of neighbors, then scales + searches + returns the top matches (or `None`
  if too few recipes match the ingredient filter).
- **`extract_quoted_strings(s)`** — pulls the individual quoted items out of the
  dataset's packed `"item1" "item2"` string format (used for ingredients and
  instructions).
- **`output_recommended_recipes(dataframe)`** — converts the recommended rows into
  a list of clean dictionaries, unpacking ingredients and instructions into proper
  lists for the API response.

### `main.py` — FastAPI backend
- Loads `dataset.csv` once at startup.
- **`PredictionIn`** — request schema: a 9-value `nutrition_input`, an optional
  `ingredients` list, and optional kNN `params` (number of neighbors, whether to
  return distances).
- **`Recipe` / `PredictionOut`** — response schema describing a single recipe and
  the list returned to the client.
- **`GET /`** — health check endpoint.
- **`POST /predict/`** — takes a `PredictionIn` payload, calls `recommend()` and
  `output_recommended_recipes()` from `model.py`, and returns the matched recipes
  (or `None` if nothing matched).

### `recomendation.py` — API client wrapper
- **`Generator`** class — holds a nutrition profile, ingredient list, and kNN
  params.
- **`set_request(...)`** — updates those values (e.g. from user input in the UI).
- **`generate()`** — packages everything into JSON and POSTs it to the backend's
  `/predict/` endpoint, returning the raw response for the front end to render.

### `front1.py` — Streamlit entry page
- Configures the Streamlit page (title, icon).
- Displays the welcome message and a short description of the project, and points
  the sidebar to the actual recommendation app page(s).

### `imagefinder.py` — recipe image lookup
- **`get_images_links(searchTerm)`** — scrapes Google Images search results for a
  given term (e.g. a recipe name) with `requests` + `BeautifulSoup` and returns the
  first valid image URL it finds.
- **`Not_found_link`** — a base64-encoded placeholder image returned if the scrape
  fails or no image is found, so the UI never breaks on a missing photo.

## Setup

```bash
pip install fastapi uvicorn pandas scikit-learn streamlit requests beautifulsoup4
```

**Run the backend:**
```bash
uvicorn main:app --reload --port 8080
```

**Run the front end** (in another terminal):
```bash
streamlit run front1.py
```

## Notes / things to check before sharing this publicly
- `main.py` reads `dataset.csv` from `../Data/dataset.csv` — the CSV in this
  folder should either be moved there or the path updated to match.
- `imagefinder.py` scrapes Google Images directly, which is against Google's
  Terms of Service for production use and can break at any time if Google
  changes its markup — fine for a class project, but worth swapping for a proper
  image API (e.g. Unsplash, Pexels, or a recipe-image API) if this goes further.
- `recomendation.py` calls `http://backend:8080/predict/` — that hostname only
  resolves inside a Docker Compose network; running locally without Docker, this
  should point to `http://localhost:8080/predict/`.
