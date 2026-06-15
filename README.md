# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

```bash
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```
## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

---

# FitFindr 🛍️ — Implementation

FitFindr is an AI-powered thrift shopping agent that takes a natural language query, searches a mock secondhand listings dataset, and generates personalized outfit suggestions and a shareable fit card — all in one automated planning loop.

## How to Run

```bash
# Install dependencies
uv pip install -r requirements.txt

# Add your Groq API key to .env
GROQ_API_KEY=your_key_here

# Run the app
python app.py
```

Then open http://127.0.0.1:7860 in your browser.

---

## Tool Inventory

### `search_listings(description, size, max_price)`
**Purpose:** Searches the mock listings dataset for thrifted items matching a natural-language description, optionally filtered by size and price ceiling.

**Inputs:**
- `description` (str): Keywords describing the item (e.g., "vintage graphic tee"). Scored against title, description, style_tags, and category fields.
- `size` (str | None): Size to filter by (e.g., "M"). Case-insensitive substring match. Pass `None` to skip.
- `max_price` (float | None): Maximum price in dollars, inclusive. Pass `None` to skip.

**Returns:** A list of listing dicts sorted by relevance score (highest first). Each dict contains: `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand`, `platform`. Returns `[]` if nothing matches — never raises an exception.

---

### `suggest_outfit(new_item, wardrobe)`
**Purpose:** Uses an LLM (Groq / llama-3.3-70b-versatile) to suggest 1–2 complete outfits pairing the thrifted item with the user's existing wardrobe. Falls back to general styling advice if the wardrobe is empty.

**Inputs:**
- `new_item` (dict): A listing dict from `search_listings`.
- `wardrobe` (dict): A wardrobe dict with an `'items'` key containing a list of wardrobe item dicts (fields: `name`, `type`, `color`, `style`). May be empty.

**Returns:** A non-empty string with 1–2 outfit suggestions, or general styling advice if the wardrobe is empty.

---

### `create_fit_card(outfit, new_item)`
**Purpose:** Uses an LLM to generate a 2–4 sentence casual OOTD caption suitable for Instagram or TikTok.

**Inputs:**
- `outfit` (str): The outfit suggestion string from `suggest_outfit`.
- `new_item` (dict): The listing dict for the thrifted item.

**Returns:** A 2–4 sentence caption string mentioning the item name, price, and platform naturally. If `outfit` is empty or whitespace-only, returns a descriptive error string instead of raising an exception.

---

## How the Planning Loop Works

The agent runs a **fixed-sequence, conditional-exit** planning loop inside `run_agent()`. It does not dynamically reorder steps, but checks state after each step and exits early if something fails.

1. **Parse the query** — regex extracts `max_price` (pattern: "under $N"), `size` (pattern: "size X"), and uses the remainder as `description`.
2. **Call `search_listings`** — if results are empty, the agent sets `session["error"]` to a specific, actionable message and **returns immediately**. `suggest_outfit` is never called with empty input.
3. **Select top result** — `search_results[0]` becomes `selected_item`.
4. **Call `suggest_outfit`** — always called if a `selected_item` exists. Handles empty wardrobe internally without early exit.
5. **Call `create_fit_card`** — called with the outfit string and selected item.
6. **Return session** — caller checks `session["error"]` first, then `session["fit_card"]`.

The agent behaves **differently** depending on search results: with matches it runs all 3 tools; with no matches it stops after step 2 and never calls the LLM tools.

---

## State Management

All state is stored in a single `session` dict initialized by `_new_session()`. No global variables. Each tool writes its output to the session and the next tool reads from it — the user never re-enters data between steps.

| Field | Set by | Read by |
|---|---|---|
| `session["query"]` | `run_agent` caller | Query parsing step |
| `session["parsed"]` | Query parsing step | `search_listings` call |
| `session["search_results"]` | `search_listings` | Item selection step |
| `session["selected_item"]` | Item selection step | `suggest_outfit`, `create_fit_card` |
| `session["wardrobe"]` | `run_agent` caller | `suggest_outfit` |
| `session["outfit_suggestion"]` | `suggest_outfit` | `create_fit_card` |
| `session["fit_card"]` | `create_fit_card` | Final output / caller |
| `session["error"]` | Any failing step | `run_agent` return gate |

---

## Error Handling

| Tool | Failure mode | Agent response |
|---|---|---|
| `search_listings` | No listings match filters | Sets `session["error"]` = "No listings found for '[description]' in size [X] under $[Y]. Try a broader description, a different size, or a higher price limit." Returns immediately — LLM tools never called. |
| `suggest_outfit` | Wardrobe is empty | Falls back to a different LLM prompt asking for general styling advice. Returns useful string either way. |
| `create_fit_card` | `outfit` is empty or whitespace-only | Returns "Cannot create fit card: outfit suggestion is missing. Please run suggest_outfit first." — no exception raised. |

**Concrete examples from testing:**

- `search_listings('designer ballgown', size='XXS', max_price=5)` returned `[]` with no exception. The full agent returned: *"No listings found for 'designer ballgown' in size XXS under $5.00. Try a broader description, a different size, or a higher price limit."*
- `suggest_outfit(item, get_empty_wardrobe())` returned general styling advice — no crash, no empty string.
- `create_fit_card('', item)` returned: *"Cannot create fit card: outfit suggestion is missing. Please run suggest_outfit first."* — no exception.

---

## Spec Reflection

**One way the spec helped:** Writing the State Management table in `planning.md` before coding made it immediately clear that `selected_item` needed to be stored separately from `search_results`. The table made the data flow obvious before a single line was written.

**One way implementation diverged from the spec:** The `planning.md` described using a "regex + LLM hybrid" for query parsing. In practice, regex alone handled all test cases reliably, so the LLM parsing step was skipped to keep latency low and avoid an extra API call.

---

## AI Usage

**Instance 1 — Implementing `tools.py`:**
I gave Claude the Tool specs from `planning.md` (inputs with types, return value fields, scoring logic, failure modes) and the starter file stubs, and asked it to implement all three tools. Claude produced correct implementations for `search_listings`, `suggest_outfit`, and `create_fit_card`. I reviewed each against the spec and ran `pytest tests/` to confirm all 9 tests passed before moving on.

**Instance 2 — Implementing `run_agent`:**
I gave Claude the Planning Loop, State Management table, and Architecture diagram from `planning.md` and asked it to implement `run_agent()`. I reviewed it to confirm `suggest_outfit` was never called when `search_results` was empty, all session fields were written in the right order, and the regex patterns matched the planning.md walkthrough. I overrode one part: Claude initially included LLM-based query parsing as a fallback, but I removed it since regex alone was sufficient and the extra API call added unnecessary latency.
