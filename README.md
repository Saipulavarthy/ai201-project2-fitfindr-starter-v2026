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
FitFindr is a thrifting agent. A user types what they want in plain language, such as "vintage graphic tee under $30, size M", and the agent searches a file of 40 secondhand listings, picks the best match, suggests outfits that combine it with pieces from the user's own wardrobe, and writes a short caption the user could post about the find. If nothing matches, the agent stops early and tells the user what to change instead of running the later steps on nothing.

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

Matching rules: the description is lowercased and split into words. Price amounts like `$30`, single-letter words, and filler words such as "a", "the", "under", "size", "looking" and "want" are ignored. A listing scores one point for each distinct description word that appears as a whole word in its title, description, style_tags, or category. Brand is not searched. Because matching is whole-word only, plurals miss (`sneaker` will not match `sneakers`). Size is compared case-insensitively and matches if the requested size equals the whole listing size or equals one piece of it when split on spaces, "/", and parentheses, so "M" matches "M" and "S/M" but not "L", "XL (oversized)", or "US 9". Price passes when it is less than or equal to `max_price`. Ties on score are broken by lower price, so the first result is not always the most literal match.

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

$ python app.py ask 'vintage graphic tee under $30, size M'
[1] search_listings
      in:  description='vintage graphic tee', size='M', max_price=30.0
      out: 8 items: Y2K Baby Tee — Butterfly Print, Mesh Long-Sleeve Top — Black, Henley Long Sleeve — Washed Burgundy … +5 more
[2] suggest_outfit
      in:  Y2K Baby Tee — Butterfly Print ($18.0, depop)
      out: For a quintessential Y2K-inspired streetwear look, you can pair the Y2K Baby Tee — Butterfly Print with the Ba…
[3] create_fit_card
      in:  Y2K Baby Tee — Butterfly Print ($18.0, depop)
      out: scored this cute little butterfly tee on depop for just $18 and I am obsessed with the early 2000s mall-rat en…

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   For a quintessential Y2K-inspired streetwear look, you can pair the Y2K Baby Tee — Butterfly Print with the Baggy straight-leg jeans, dark wash, layered underneath the Black cropped zip hoodie for a balanced proportion of fitted and relaxed silhouettes. Complete this outfit by stepping into the Chunky white sneakers and wearing the Black crossbody bag for an effortless everyday finish.

Alternatively, for a more contrasting style that blends the tee's delicate butterfly graphic with a grungier aesthetic, tuck the Y2K Baby Tee — Butterfly Print into the Wide-leg khaki trousers. Layer the Vintage black denim jacket on top and anchor the entire look with the Black combat boots, adding the Black crossbody bag to tie the accessories together.

  Fit card: scored this cute little butterfly tee on depop for just $18 and I am obsessed with the early 2000s mall-rat energy. styling it with baggy dark wash denim and chunky white sneakers for the ultimate nostalgic off-duty look. #y2kstyle

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

- *What I asked for:* I gave my VS Code assistant my Tool Inventory spec and asked it to implement `search_listings`, `suggest_outfit`, and `create_fit_card` in `tools.py`, with exact-piece size matching, an empty list on no match, and a `[FALLBACK]` caption when the model fails.
- *What came back:* Working tools, plus a report of conflicts. My spec said the tools "never raise," but `generate.py` and `agent.py` expect the loop to handle `ModelUnavailable`, so the two model tools let it through. It also caught `QuotaGuard` and turned it into a fallback, and its size rule could never match a value like "One Size".
- *What I changed:* I kept `ModelUnavailable` passing through and rewrote my spec to say "never raises, except `ModelUnavailable` and `QuotaGuard`." I made `QuotaGuard` propagate too, because a fallback caption would hide a runaway loop. I added a rule that a size also matches when the requested size equals the whole listing size.

**Moment 2**

- *What I asked for:* I asked an AI to draft my branch rule and Planning Loop section before I had seen `agent.py`.
- *What came back:* A reasonable rule, but with session keys it had guessed: `outfit` and a separate message field.
- *What I changed:* After reading `agent.py` I corrected the keys to `outfit_suggestion` and `session["error"]`, which are what the starter's session dictionary uses. I also made the empty-search message name the parsed query values, so the user can see what was searched.

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

From `results/run_2026-10-10_2132_before.md`, run with `python run_eval.py --label before`, five tries per scenario, caching off, after the `search_listings` move to MCP.

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. Full three-tool run returns a fit card | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. Empty search stops before suggest_outfit | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. Item logged for both tools equals session["selected_item"] | 5 of 5 | PASS* | PASS* | PASS* | PASS* | PASS* | see Verdicts section |
| 4. Fit card ≤3 sentences, has a title word, no [FALLBACK] | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. Query parsed to max_price 30 and size "M" | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

* Criterion 3 was scored on title, price and platform only, because the trace stores a string, not a dictionary.

**Real output from one try**, copied from `results/run_2026-10-10_2132_before.md`, try 1 of each scenario. The trace lines come from `trace.py::step`, called by `agent.py::run_agent`. The results file itself was written by `run_eval.py::write_report`.

**Criterion 1**: scenario "matching query completes", query `vintage graphic tee under $30`. Trace from `agent.py::run_agent` (via `trace.py::step`); fit card from `tools.py::create_fit_card`.

```text
[1] search_listings (via MCP)
      in:  description='vintage graphic tee', size=None, max_price=30.0
      out: 10 items: Y2K Baby Tee — Butterfly Print, Vintage Band Tee — Faded Grey, Graphic Tee — 2003 Tour Bootleg Style … +7 more
[2] suggest_outfit
      in:  Y2K Baby Tee — Butterfly Print ($18.0, depop)
      out: For a casual, streetwear-inspired look, pair the Y2K baby tee with the baggy straight-leg jeans, dark wash, an…
[3] create_fit_card
      in:  Y2K Baby Tee — Butterfly Print ($18.0, depop)
      out: Found this adorable butterfly baby tee on Depop for just $18 and I'm obsessed. It gives off the ultimate effor…
```

```text
Found this adorable butterfly baby tee on Depop for just $18 and I'm obsessed. It gives off the ultimate effortless streetwear vibe when you style it with baggy dark wash denim and chunky sneaks. 🦋✨ #y2kstyle
```

**Criterion 2**: scenario "impossible query stops early", query `designer ballgown size XXS under $5`. Branch step from `agent.py::run_agent` (via `trace.py::step`); stopped-early message from `agent.py::_no_results_message`, as recorded by `run_eval.py::write_report`.

```text
[2] branch
      in:  description='designer ballgown', size='XXS', max_price=5.0
      out: [] (empty)
      →    branch: empty results, stopping before suggest_outfit
```

```text
- stopped early: yes — No listings matched (description "designer ballgown", size XXS, max price $5.00). Try to raise the price ceiling, or remove the size, or try different or fewer description words.
```

**Criterion 3**: scenario "selected item reaches both tools", query `denim jacket under $50`. `selected_item` line from `run_eval.py::write_report`; trace from `agent.py::run_agent` (via `trace.py::step`).

```text
- selected_item: Denim Jacket — Light Wash, Cropped ($42.0, poshmark)
```

```text
[2] suggest_outfit
      in:  Denim Jacket — Light Wash, Cropped ($42.0, poshmark)
      out: For an effortless streetwear-inspired look, pair the denim jacket with the white ribbed tank top tucked into t…
[3] create_fit_card
      in:  Denim Jacket — Light Wash, Cropped ($42.0, poshmark)
      out: Score the ultimate streetwear look with this light wash cropped denim jacket I just snagged on Poshmark for on…
```

**Criterion 4**: scenario "fit card short and on-item", query `leather jacket under $80`. Trace from `agent.py::run_agent` (via `trace.py::step`); fit card from `tools.py::create_fit_card`.

```text
[1] search_listings (via MCP)
      in:  description='leather jacket', size=None, max_price=80.0
      out: 7 items: 90s Leather Bomber — Black, Leather Belt — Brown, Braided, Shacket — Olive Canvas … +4 more
[2] suggest_outfit
      in:  90s Leather Bomber — Black ($75.0, depop)
      out: For a classic grunge-inspired look, wear the thrifted 90s leather bomber over the white ribbed tank top paired…
[3] create_fit_card
      in:  90s Leather Bomber — Black ($75.0, depop)
      out: Finally scored this ultimate 90s Leather Bomber on depop for just $75.00, and it gives off the absolute best e…
```

```text
Finally scored this ultimate 90s Leather Bomber on depop for just $75.00, and it gives off the absolute best effortless grunge energy. I'm obsessed with throwing it over a basic tank and baggy denim for that slouchy, vintage-store-find aesthetic. #thriftfinds
```

Note on scoring criterion 4: case was ignored when checking for title words (Leather, Bomber, Black), and hashtags were not counted as sentences. The try 1 fit card contains "$75.00,", whose decimal point makes a plain split on `.`, `!` and `?` count 3 pieces, while reading it as prose counts 2 sentences. Both are within the limit of 3.

**Criterion 5**: scenario "price and size parsed", query `vintage graphic tee under $30, size M`. Step 1 trace line from `agent.py::run_agent` (via `trace.py::step`), built from `session["parsed"]`.

```text
[1] search_listings (via MCP)
      in:  description='vintage graphic tee', size='M', max_price=30.0
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

**Criterion 1: MET (5/5, target 4 of 5).** All five tries ran search_listings, suggest_outfit and create_fit_card and printed a fit card. I counted a try as passing when all three steps appeared in the trace, since trace.step only runs after a tool call returns. No misses, so nothing to diagnose. My original reason for 4 of 5 was that model calls can fail or return something unusable. One run of 5/5 doesn't show that reason was wrong, only that it didn't happen in these five tries, and five tries is a small sample, so I can't say the target was too low from this run alone.

**Criterion 2: MET (5/5, target 5 of 5).** The impossible query ("designer ballgown size XXS under $5") returned an empty list in every try, the loop's branch stopped before suggest_outfit, and the message named what to change. The empty list survived the MCP round trip as `[]`, which is what I was most worried about after moving search_listings behind MCP.

**Criterion 3: not measurable as written.** The criterion says the item dictionary logged by trace.step for suggest_outfit and create_fit_card equals session["selected_item"]. The trace stores a string (Title, price, platform), not a dictionary, and agent.py passes session["selected_item"] itself to trace.step, so the logged item is the session's own object. The comparison can't fail even in principle. What I could check, and did in all five tries, is that title, price and platform in the step 2 and step 3 `in:` lines match the selected item. That is a weaker check than my criterion, so I'm not calling it MET against the original wording. (Revision in criteria.md.)

**Criterion 4: MET (5/5, target 5 of 5).** Every fit card had 3 sentences or fewer, contained "Leather" and "Bomber" (case ignored), and none started with [FALLBACK]. The only ambiguous count was try 1, where the decimal point in "$75.00," makes a plain split give 3 pieces and a prose reading give 2; both are within the limit. This criterion is easy to pass: the query itself says "leather jacket", so a title word almost always shows up, and I'd tighten it to require the price or the item's full name.

**Criterion 5: MET (5/5, target 5 of 5).** The step 1 trace showed size='M' and max_price=30.0 in all five tries. The check is indirect because the report doesn't save session["parsed"], but the trace line is built from it.

**Pattern.** There were no misses, so there is no failing step to name. The weak spots are in the measuring, not the agent: criterion 3 can't be measured, criterion 4 is lenient, and criterion 1's 4-of-5 target may be low, though one 5/5 run can't show that. One problem none of my criteria catch: for the query 'oversized corduroy jacket under $40', the 40 listings contain no corduroy jacket, so the search can only return near matches. But search_listings scores each listing by how many description words it matches and breaks ties by lowest price, so the Corduroy Wide-Leg Pants rank 5th behind four 'oversized' tops (including the tie-dye one), and the agent then styled a tie-dye top (the top hit) and presented it with no sign it wasn't a corduroy jacket. (The corduroy query came from a build run in my terminal, not from results/, so it isn't in the saved run log.)

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
