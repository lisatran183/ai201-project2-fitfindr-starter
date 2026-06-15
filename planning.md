# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Searches the mock listings dataset for thrifted items matching a natural-language description, optionally filtered by size and price ceiling. Returns a ranked list of matching listings sorted by keyword relevance.


**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): Keywords describing the item the user wants (e.g., "vintage graphic tee", "floral midi skirt"). Used for keyword scoring against listing titles, descriptions, style_tags, and category.
- `size` (str): Clothing size to filter by (e.g., "M", "S", "XL"). Matching is case-insensitive and substring-based so "M" matches "S/M". Pass `None` to skip size filtering.
- `max_price` (float): Maximum price in dollars, inclusive (e.g., 30.0). Pass `None` to skip price filtering.

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
A list of listing dicts, sorted highest-relevance first. Each dict contains:
- `id` (str): Unique listing ID
- `title` (str): Listing title
- `description` (str): Full item description
- `category` (str): Clothing category (e.g., "tops", "bottoms")
- `style_tags` (list[str]): Tags like ["vintage", "streetwear"]
- `size` (str): Size string (e.g., "M", "One Size")
- `condition` (str): Item condition (e.g., "Good", "Like New")
- `price` (float): Listed price in USD
- `colors` (list[str]): Color list (e.g., ["black", "white"])
- `brand` (str): Brand name or "Unknown"
- `platform` (str): Source platform (e.g., "Depop", "ThredUp")

Returns an empty list if nothing matches — does NOT raise an exception.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
The agent sets `session["error"]` to a helpful, actionable message (e.g., "No listings found for 'designer ballgown size XXS under $5'. Try a broader description, a different size, or a higher price limit.") and returns the session early. `suggest_outfit` and `create_fit_card` are NOT called.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Uses an LLM (via Groq) to suggest 1–2 complete outfits pairing the new thrifted item with pieces from the user's existing wardrobe. If the wardrobe is empty, it gives general styling advice for the item instead.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): A listing dict (the item the user is considering buying) — same structure as what `search_listings` returns.
- `wardrobe` (dict): A wardrobe dict with an `'items'` key holding a list of wardrobe item dicts (each with fields like `name`, `type`, `color`, `style`). The list may be empty.

**What it returns:**
<!-- Describe the return value -->
A non-empty string with 1–2 outfit suggestions. If the wardrobe is empty, returns general styling advice (what item types pair well, what aesthetic/vibe the piece suits, occasion ideas). Never returns an empty string.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
If the wardrobe is empty, the function falls back to general styling advice rather than raising an error. If the LLM call fails, the function catches the exception and returns a fallback string: "Unable to generate outfit suggestion. The item is a [category] in [color] — try pairing it with basics in neutral tones."

---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Uses an LLM to generate a 2–4 sentence casual, shareable outfit caption (OOTD-style) based on the outfit suggestion and item details. Suitable for Instagram or TikTok.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (str): The outfit suggestion string returned by `suggest_outfit`.
- `new_item` (dict): The listing dict for the thrifted item (provides title, price, platform, colors, style_tags).

**What it returns:**
<!-- Describe the return value -->
A 2–4 sentence caption string that:
- Feels casual and authentic (not a product description)
- Mentions the item name, price, and platform naturally, once each
- Captures the outfit vibe in specific sensory/aesthetic terms
- Sounds different each time (LLM called with temperature ~0.9)

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
If `outfit` is empty or whitespace-only, the function returns a descriptive error string: "Cannot create fit card: outfit suggestion is missing. Please run suggest_outfit first." — it does NOT raise an exception.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->
None required beyond the three above for the base implementation.

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

The agent runs a **fixed-sequence, conditional-exit** planning loop. It does not dynamically reorder steps — but it checks state at each step and exits early with an informative error if a required input is missing or a tool returns nothing useful.

**Decision logic:**

1. **Parse query** — Extract `description`, `size`, and `max_price` from the raw query string using a regex + LLM hybrid approach (regex for price/size patterns; LLM for description extraction if regex is ambiguous). Store in `session["parsed"]`.

2. **Call `search_listings`** — If `session["search_results"]` is empty after the call, set `session["error"]` and **return early**. Do not call `suggest_outfit` with empty input.

3. **Select top result** — Take `search_results[0]` as the `selected_item`. (Future stretch: score-weighted random selection for variety.) Store in `session["selected_item"]`.

4. **Call `suggest_outfit`** — Always called if a `selected_item` exists. Handles empty wardrobe internally. Stores result in `session["outfit_suggestion"]`.

5. **Call `create_fit_card`** — Only called if `outfit_suggestion` is non-empty. Stores result in `session["fit_card"]`.

6. **Return session** — The caller checks `session["error"]` first, then `session["fit_card"]`.

The loop **knows it's done** when either (a) `session["error"]` is set, or (b) `session["fit_card"]` is populated.

---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->

All state lives in a single `session` dict initialized by `_new_session()`. No global variables. State flows forward only — each tool writes to the session, and the next tool reads from it.

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

The user never re-enters data between steps. The item from `search_listings` flows directly into `suggest_outfit` via `session["selected_item"]`, and the outfit string from `suggest_outfit` flows directly into `create_fit_card` via `session["outfit_suggestion"]`.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No listings match description/size/price filters | Set `session["error"]` = "No listings found for '[description]'. Try broadening your description, adjusting your size, or raising your price limit." Return session immediately — do NOT call subsequent tools. |
| `suggest_outfit` | Wardrobe `items` list is empty | Fall back to LLM prompt for general styling advice (not an error — handled inside the tool). If LLM call itself fails, return a hardcoded fallback string describing basic pairing principles for the item's category and color. |
| `create_fit_card` | `outfit` argument is empty or whitespace-only | Return a descriptive error string: "Cannot create fit card: outfit suggestion is missing." Do NOT raise an exception. Agent logs this but still returns the session with `fit_card` set to the error string so the caller can inspect it. |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

```
User query
     │
     ▼
Planning Loop ──────────────────────────────────────────┐
     │                                                   │
     ├──► search_listings(description, size, max_price)  │
     │         │                                         │
     │         ├── results=[]                            │
     │         │      └──► [ERROR] "No listings found..."─► return
     │         │                                         │
     │         └── results=[item, ...]                   │
     │                  │                                │
     │         Session: selected_item = results[0]       │
     │                  │                                │
     ├──► suggest_outfit(selected_item, wardrobe)        │
     │         │                                         │
     │         Session: outfit_suggestion = "..."        │
     │                  │                                │
     └──► create_fit_card(outfit_suggestion, selected_item)
               │
               Session: fit_card = "..."
               │
               ▼
          Return session ◄──────────── error path returns here
```
---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**

**Tool 1 — `search_listings`:**
- **AI tool:** Claude (claude.ai)
- **Input to Claude:** This planning.md Tool 1 section (inputs, return value, failure mode) + the `load_listings()` function signature from `utils/data_loader.py`
- **Prompt:** "Implement `search_listings(description, size, max_price)` in Python. Use `load_listings()` to get all listings. Filter by `max_price` (inclusive) and `size` (case-insensitive substring match) if provided. Score each listing by counting how many words from `description` appear in its title, description, style_tags, and category fields combined. Drop listings with score 0. Sort by score descending and return the list. Do not raise exceptions — return [] on no match."
- **Verification:** Run 3 test queries: (1) "vintage graphic tee" with no filters — expect multiple results; (2) same query with `size="M"` — expect subset; (3) impossible query "zxqwerty" — expect `[]`.

**Tool 2 — `suggest_outfit`:**
- **AI tool:** Claude
- **Input to Claude:** Tool 2 spec from this planning.md + example wardrobe dict structure
- **Prompt:** "Implement `suggest_outfit(new_item, wardrobe)`. If `wardrobe['items']` is empty, call the Groq LLM asking for general styling ideas for the item (mention its category, color, style_tags). If not empty, format the wardrobe items as a list and ask the LLM to suggest 1–2 specific outfits combining the new item with named wardrobe pieces. Return the LLM response string. Never return empty string."
- **Verification:** Test with (1) populated wardrobe — check that response references actual wardrobe items by name; (2) empty wardrobe — check that response is non-empty and gives styling advice, not an error.

**Tool 3 — `create_fit_card`:**
- **AI tool:** Claude
- **Input to Claude:** Tool 3 spec from this planning.md
- **Prompt:** "Implement `create_fit_card(outfit, new_item)`. Guard against empty/whitespace-only `outfit` — return a descriptive error string. Otherwise, build a prompt asking the LLM for a 2–4 sentence casual OOTD caption that mentions the item's title, price, and platform once each. Use temperature=0.9 for variety. Return the LLM response."
- **Verification:** Test with (1) valid outfit + item — confirm caption is 2–4 sentences, mentions price and platform; (2) empty outfit string — confirm error string returned, no exception raised.

**Milestone 4 — Planning loop and state management:**

- **AI tool:** Claude
- **Input to Claude:** The Planning Loop, State Management, and Architecture sections of this planning.md (the full session dict schema + the flowchart above)
- **Prompt:** "Implement `run_agent(query, wardrobe)` following this state management plan. Parse the query with regex for price (e.g. 'under $30') and size (e.g. 'size M'), then use the remainder as description. Call each tool in order, storing results in the session dict. If `search_results` is empty, set `session['error']` and return early. Return the session dict."
- **Verification:** Run the two CLI test cases already in `agent.py`: happy path (graphic tee) should produce a non-None `fit_card`; no-results path (designer ballgown XXS under $5) should produce a non-None `error` and None `fit_card`.

---

## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30, size M. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1 — Parse the query**

The agent applies regex to extract:
- `max_price = 30.0` (matched from "under $30")
- `size = "M"` (matched from "size M")
- `description = "vintage graphic tee"` (remainder after stripping price and size tokens)

Stored in `session["parsed"] = {"description": "vintage graphic tee", "size": "M", "max_price": 30.0}`.

**Step 2 — Call `search_listings("vintage graphic tee", size="M", max_price=30.0)`**

The tool loads all listings from `data/listings.json`, drops any priced above $30, drops any where "M" is not a case-insensitive substring of the size field, then scores the rest by keyword overlap with "vintage graphic tee" across title, description, style_tags, and category. Returns a list of matches sorted by score. Suppose it returns 3 results; top result: `{"title": "Faded Band Tee — Rolling Stones", "price": 22.0, "platform": "Depop", "size": "M", "condition": "Good", ...}`.

Stored in `session["search_results"]` (3 items) and `session["selected_item"]` (top result dict).

**Step 3 — Call `suggest_outfit(selected_item, wardrobe)`**

The agent passes the Rolling Stones tee dict and the user's wardrobe (which contains: wide-leg jeans, chunky platform sneakers, an oversized denim jacket, a white ribbed tank). The wardrobe is not empty, so the LLM receives a prompt listing all wardrobe items and asks for outfit combinations.

LLM returns: "Outfit 1: Tuck the Rolling Stones tee into your wide-leg jeans and lace up the chunky platform sneakers — add the denim jacket open for a 90s grunge-meets-streetwear look. Outfit 2: Layer the tee over the white ribbed tank, knot the front, and wear with the wide-leg jeans and sneakers for a relaxed off-duty vibe."

Stored in `session["outfit_suggestion"]`.

**Step 4 — Call `create_fit_card(outfit_suggestion, selected_item)`**

The agent passes the outfit string and the tee's listing dict. The LLM generates a caption at temperature 0.9:

"Thrifted this faded Rolling Stones tee off Depop for $22 and honestly it's doing all the heavy lifting. Wide-leg jeans, chunky platforms, denim jacket open — 90s grunge activated. Some finds just make sense. 🎸"

Stored in `session["fit_card"]`. `session["error"]` remains `None`.

**Final output to user:**
<!-- What does the user actually see at the end? -->

Found: Faded Band Tee — Rolling Stones ($22, Depop, Good condition)

Outfit suggestion:
Outfit 1: Tuck the Rolling Stones tee into your wide-leg jeans and lace up
the chunky platform sneakers — add the denim jacket open for a 90s look.
Outfit 2: Layer the tee over the white ribbed tank, knot the front, and wear
with the wide-leg jeans and sneakers for a relaxed off-duty vibe.

Fit card:
Thrifted this faded Rolling Stones tee off Depop for $22 and honestly it's
doing all the heavy lifting. Wide-leg jeans, chunky platforms, denim jacket
open — 90s grunge activated. Some finds just make sense. 🎸

**Error path example:**
If the query were "designer ballgown size XXS under $5", `search_listings` would return `[]`. The agent sets `session["error"] = "No listings found for 'designer ballgown' in size XXS under $5.00. Try a broader description, a different size, or a higher price limit."` and returns immediately. `suggest_outfit` is never called.
