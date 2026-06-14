# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

# FitFindr – planning.md

---

## Tools

### Tool 1: search_listings

**What it does:**
Searches the mock listings dataset for items matching a text description, optional size, and optional price ceiling. Returns a ranked list of matching items sorted by keyword relevance.

**Input parameters:**
- `description` (str): Keywords describing what the user is looking for (e.g. "vintage graphic tee")
- `size` (str): Size string to filter by, or None to skip size filtering. Case-insensitive (e.g. "M" matches "S/M")
- `max_price` (float): Maximum price inclusive, or None to skip price filtering

**What it returns:**
A list of matching listing dicts sorted by relevance score (highest first). Each dict contains: id, title, description, category, style_tags, size, condition, price, colors, brand, platform. Returns an empty list if nothing matches — does NOT raise an exception.

**What happens if it fails or returns nothing:**
If the list is empty, the agent sets session["error"] to a helpful message telling the user no listings were found and suggesting they try a different description or relax the filters. The agent returns early and does NOT proceed to suggest_outfit.

---

### Tool 2: suggest_outfit

**What it does:**
Given a thrifted item the user is considering and their current wardrobe, uses the Groq LLM to suggest 1-2 complete outfit combinations. If the wardrobe is empty, it gives general styling advice instead.

**Input parameters:**
- `new_item` (dict): A listing dict — the item the user is considering buying
- `wardrobe` (dict): A wardrobe dict with an 'items' key containing a list of wardrobe item dicts. May be empty.

**What it returns:**
A non-empty string with outfit suggestions. If the wardrobe is empty, returns general styling ideas for the item rather than specific combinations.

**What happens if it fails or returns nothing:**
If wardrobe["items"] is empty, the agent calls the LLM with a general styling prompt instead of a specific outfit combination prompt — so it always returns something useful.

---

### Tool 3: create_fit_card

**What it does:**
Generates a short, shareable Instagram/TikTok-style caption for the thrifted outfit. Sounds casual and authentic — like a real OOTD post, not a product description.

**Input parameters:**
- `outfit` (str): The outfit suggestion string from suggest_outfit
- `new_item` (dict): The listing dict for the thrifted item (used for title, price, platform)

**What it returns:**
A 2-4 sentence caption string that mentions the item name, price, and platform naturally, and captures the outfit vibe. Returns a descriptive error string if outfit input is empty — does NOT raise an exception.

**What happens if it fails or returns nothing:**
If the outfit string is empty or whitespace-only, the function returns an error message string without calling the LLM. This prevents wasted API calls and gives the user a clear signal something went wrong upstream.

---

## Planning Loop

**How does your agent decide which tool to call next?**

The agent follows a fixed conditional sequence — each step only runs if the previous one succeeded:

1. Parse the user's natural language query using the LLM to extract description, size, and max_price
2. Call search_listings() with the parsed parameters
3. If search returns empty → set session["error"] and return early (skip steps 4-6)
4. If results found → select the top result as selected_item
5. Call suggest_outfit() with the selected item and the user's wardrobe
6. Call create_fit_card() with the outfit suggestion and selected item
7. Return the completed session

The loop is conditional: the agent does NOT always call all three tools. If search fails, it stops and reports the error rather than passing empty data to suggest_outfit.

---

## State Management

**How does information from one tool get passed to the next?**

All state is stored in a single session dict initialized at the start of each run. The dict tracks:

- `query`: the original user query string
- `parsed`: the extracted description, size, max_price from the LLM parser
- `search_results`: full list returned by search_listings
- `selected_item`: the top result, passed into suggest_outfit
- `wardrobe`: the user's wardrobe dict, provided at session start
- `outfit_suggestion`: string returned by suggest_outfit, passed into create_fit_card
- `fit_card`: final caption string returned by create_fit_card
- `error`: set if any step fails early; None on success

Each tool reads its inputs from the session and writes its output back to the session. No tool re-fetches data that a previous tool already retrieved.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets session["error"] with a helpful message, returns session early, does not call suggest_outfit |
| suggest_outfit | Wardrobe is empty | Switches to a general styling prompt instead of outfit combinations — always returns a useful string |
| create_fit_card | Outfit input is empty or whitespace | Returns a descriptive error string without calling the LLM — does not crash |

---

## Architecture

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | |
| suggest_outfit | Wardrobe is empty | |
| create_fit_card | Outfit input is missing or incomplete | |

---

## Architecture

Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
     
     User query + wardrobe
     ↓
     Parse query (LLM)
     ↓
     search_listings()
     ↓
     Results found? ──── NO ──→ session["error"] → return early
     │
     YES
     ↓
     suggest_outfit()
     ↓
     create_fit_card()
     ↓
     Return session
     (listing, outfit, fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool.

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

All steps read from and write to the session dict.

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**
Used Claude to implement each tool one at a time. For each tool, provided the function signature, the TODO steps from tools.py, and the data structure from load_listings(). Tested each tool in isolation before moving to the next. Verified search_listings with known queries, suggest_outfit with a real wardrobe, and create_fit_card with outfit text.

**Milestone 4 — Planning loop and state management:**
Used Claude to implement run_agent() in agent.py using the 7-step plan in the docstring. Provided the session dict structure and the tool signatures. Verified with the two built-in CLI tests (happy path and no-results path).

---

## A Complete Interaction (Step by Step)

**Example user query:** 

**Step 1:**
The agent initializes a session dict and calls the LLM to parse the query. The LLM extracts: description="vintage graphic tee", size=None, max_price=30.0. This is stored in session["parsed"].

**Step 2:**
search_listings("vintage graphic tee", size=None, max_price=30.0) is called. It loads all 40 listings, filters to those under $30, scores each by keyword overlap with "vintage graphic tee", and returns the ranked matches. The top result is stored in session["selected_item"].

**Step 3:**
suggest_outfit(selected_item, wardrobe) is called. The wardrobe has items, so the LLM receives a prompt listing the wardrobe pieces and asks for 1-2 outfit combinations using the new tee. The LLM returns two outfits as a string, stored in session["outfit_suggestion"].

**Step 4:**
create_fit_card(outfit_suggestion, selected_item) is called. The LLM receives the outfit text and item details and writes a casual Instagram-style caption. Stored in session["fit_card"].

**Final output to user:**
- Top listing panel: item title, price, size, condition, platform, description
- Outfit idea panel: two complete outfit combinations using wardrobe pieces
- Fit card panel: a shareable caption like "Just scored this sick vintage tee on Depop for $24 and I'm obsessed..."

---

