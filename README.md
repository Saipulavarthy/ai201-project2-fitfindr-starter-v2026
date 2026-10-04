# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does
Searches the listings file for items matching a text description, an optional size, and an optional price ceiling, and returns the matches ranked best first.

<!-- Three or four sentences: what a user asks for, and what they get back. -->



---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Searches the listings file for items matching a text description, an optional size, and an optional price ceiling, and returns the matches ranked best first.
- **Inputs:** `description` (str), `size` (str | None), `max_price` (float | None). If `size` or `max_price` is None, that filter is skipped.
- **Returns:** A list of full listing dicts (id, title, description, category, style_tags, size, condition, price, colors, brand, platform), at most `config.SEARCH_RESULT_LIMIT` of them, sorted by number of matching description words (most first) and then by price (lowest first). Listings that match no description words are dropped.
- **When it has nothing:** An empty list `[]`, never None and never an exception.

**Matching rules.** The description is lowercased and split into words. Price amounts like `$30`, single-letter words, and filler words such as "a", "the", "under", "size", "looking" and "want" are ignored. A listing scores one point for each distinct description word that appears as a whole word in its title, description, style_tags, or category. Brand is not searched. Because matching is whole-word only, plurals miss (`sneaker` will not match `sneakers`). Size is compared case-insensitively and matches if the requested size equals the whole listing size or equals one piece of it when split on spaces, "/", and parentheses, so "M" matches "M" and "S/M" but not "L", "XL (oversized)", or "US 9". Price passes when it is less than or equal to `max_price`. Ties on score are broken by lower price, so the first result is not always the most literal match.


### `suggest_outfit`

- **What it does:** Takes one listing and the user's wardrobe and asks the model for one or two outfit ideas that combine the new item with pieces the user already owns.
- **Inputs:** `new_item` (dict, one full listing dict as returned by `search_listings`), `wardrobe` (dict with an `items` key holding a list of wardrobe item dicts, each with id, name, category, colors, style_tags, and notes, which may be null).
- **Returns:** A non-empty string of outfit advice in plain prose, naming specific wardrobe pieces.
- **When it has nothing:** If `wardrobe["items"]` is empty, it returns general styling advice for the item that names no wardrobe pieces. If the model call fails or returns an empty response, it returns the fallback string "Outfit ideas are unavailable right now for <title>." It never returns None or "" and never raises, except that `ModelUnavailable` and `QuotaGuard` are allowed through for the loop to handle.


### `create_fit_card`

- **What it does:** Takes the outfit advice and the new item and asks the model to write a short caption someone would actually post about the find.
- **Inputs:** `outfit` (str, the string returned by `suggest_outfit`), `new_item` (dict, one full listing dict as returned by `search_listings`).
- **Returns:** A string of two to three sentences that mentions the item, its price, and its platform once each, optionally with a hashtag. The wording differs between runs.
- **When it has nothing:** If `outfit` is empty or whitespace, or the model call fails, it returns a plain fallback caption that begins with exactly "[FALLBACK]" followed by a sentence built from the title, price, and platform. It never returns None or "" and never raises, except that `ModelUnavailable` and `QuotaGuard` are allowed through for the loop to handle.

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, the loop sets `session["error"]` to a message naming what the user could change (raise the price ceiling, drop the size, or try fewer or different description words), leaves `session["fit_card"]` as None, and returns the session without calling `suggest_outfit` or `create_fit_card`. Otherwise it stores the results in `session["search_results"]`, sets `session["selected_item"]` to the first result (the best match, since results are ranked), calls `suggest_outfit` with that item and the wardrobe, and then calls `create_fit_card` with the outfit and the same item.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** With regular expressions and plain string handling, not the model. One pattern pulls a price ceiling out of phrases like "under $30" or "below 30", and another pulls a size out of phrases like "size M". The text that is left becomes the description. If no price or size is found, that value is None and the filter is skipped. This is predictable, but it will miss unusual phrasings such as "less than thirty dollars". The result goes in `session["parsed"]`.

**What moves through the session:** The loop writes to the session after each tool call and reads from it before the next one, in this order: `parsed` (description, size, max_price), `search_results`, `selected_item`, `outfit_suggestion`, `fit_card`. Each tool gets its input from the session, never straight from the previous call's return value, so `selected_item` can be checked against what reached `suggest_outfit`. In the empty-search path the run ends with `search_results` empty, `error` set, and `fit_card` still None.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask '...'

```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}]
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
For an effortless, casual streetwear look, pair the vintage Levi's 501 jeans with the white ribbed tank top tucked in at the waist, cinched with the brown leather belt, and finished off with the chunky white sneakers. To complete the outfit on a cooler day, layer the oversized grey crewneck sweatshirt on top and carry the black crossbody bag.

Another great option leans into a more classic, grunge-inspired aesthetic by combining the vintage Levi's 501 jeans with the black cropped zip hoodie and the black combat boots. You can add the vintage black denim jacket as an outer layer for a cohesive double-denim look and accessorize with the black crossbody bag for a sharp, monochrome finish.
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
Literally obsessed with these vintage Levi's 501 jeans I just scored on depop for only $38.00. They have that slouchy, effortlessly cool 90s fit when you pair them with fresh white sneakers. Total thrift gold. #depopfinds
```

**Edge cases**

```
$ python -c "from tools import search_listings; print(search_listings('designer ballgown', size='XXS', max_price=5))"
[]

$ python -c "from tools import search_listings; print([(l['title'], l['size']) for l in search_listings('tee', size='M')])"
[('Mesh Long-Sleeve Top — Black', 'S/M'), ('Y2K Baby Tee — Butterfly Print', 'S/M')]

$ python -c "from tools import suggest_outfit; from utils.data_loader import load_listings; print(suggest_outfit(load_listings()[0], {'items': []}))"
A classic pair of vintage medium-wash Levi's 501 jeans can serve as the anchor for a casual, streetwear-inspired daytime look by pairing them with an oversized graphic t-shirt and a classic canvas sneaker. Alternatively, for a slightly sharper aesthetic, the jeans can be styled with a tucked-in ribbed tank top layered beneath an open button-down shirt, finished off with leather loafers or retro runners.
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
