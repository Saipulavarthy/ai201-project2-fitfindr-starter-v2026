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
- **Returns:** A list of full listing dicts (id, title, description, category, style_tags, size, condition, price, colors, brand, platform), at most `config.SEARCH_RESULT_LIMIT` of them, sorted by number of matching description words (most first), then by whether the query's last keyword is in the title, then by price (lowest first). Listings that match no description words are dropped.
- **When it has nothing:** An empty list `[]`, never None and never an exception.

Matching rules: the description is lowercased and split into words. Price amounts like `$30`, single-letter words, and filler words such as "a", "the", "under", "size", "looking" and "want" are ignored. A listing scores one point for each distinct description word that appears as a whole word in its title, description, style_tags, or category. Brand is not searched. Because matching is whole-word only, plurals miss (`sneaker` will not match `sneakers`). Size is compared case-insensitively and matches if the requested size equals the whole listing size or equals one piece of it when split on spaces, "/", and parentheses, so "M" matches "M" and "S/M" but not "L", "XL (oversized)", or "US 9". Price passes when it is less than or equal to `max_price`. Ties on score go first to listings whose title contains the query's last keyword (the last non-filler word as typed, so "jacket" in "oversized corduroy jacket", matched as a whole word, case-insensitively), then to the lower price. This is a tie-breaker only and adds nothing to the score, so the first result is still not always the most literal match.

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

### Unit 4

**Moment 3**

- *What I asked for:* I asked my VS Code assistant to move search_listings behind MCP and compare the direct and MCP results before changing the agent, reporting differences without fixing them.
- *What came back:* It registered the tool, found no differences in values, order or field types across three queries, and noticed that my default python couldn't import mcp because the SDK is only in .venv.
- *What I changed:* Nothing in the code. I did notice that its trace still printed "search_listings" whether or not the call went over MCP, so I had it relabel step 1 to show "via MCP".

**Moment 4**

- *What I asked for:* I asked the assistant to build my scenarios and say which criteria the run_eval scorer could and couldn't check.
- *What came back:* One scenario per criterion, plus a finding that criterion 3 couldn't be measured as I wrote it. Its first PASS table, though, had invented a stricter reading of criterion 4 and a looser one of criterion 3 without quoting my criteria.
- *What I changed:* I made it print my criteria word for word next to each PASS check, then decided myself that case would be ignored for title words, and wrote criterion 3's revision under the original instead of editing it.

**Moment 5**

- *What I asked for:* I asked the assistant to rank near matches better for 'oversized corduroy jacket'.
- *What came back:* A check showed no corduroy jacket exists in the data and both jackets cost over $40, which made my first scenario unable to show any change, and one of my own earlier sentences about "loose word matching" gave the wrong reason.
- *What I changed:* I dropped the price cap on the diagnostic scenario, reworded my diagnosis, and made the new rule a tie-breaker only (not extra score) so the top item for my other scenarios would not change; the assistant checked that without calling the model before I kept it.

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

<!-- On the MCP move: what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->

**The MCP move.** I moved search_listings behind MCP: mcp_server.py registers it, and agent.py::run_agent now calls it through mcp_client.call_tool instead of directly (the other two tools stay direct calls). Nothing behaved differently. Before the swap I compared the direct and MCP results for three queries; they were equal in order and field types, and the empty case came back as [] rather than None. The step 1 trace now reads "search_listings (via MCP)".

---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed.** One change in `tools.py::search_listings`: ties on the number of matching words are now broken first by whether the query's last keyword (the last non-filler word as typed, e.g. "jacket" in "oversized corduroy jacket") is a whole word in the listing's title, then by lower price. It is a tie-breaker only and adds nothing to the score. I also updated the two places that describe the ranking (the Tool Inventory text in this README and the `search_listings` description in `mcp_server.py`) so they match; that is documentation, not a second change.

**Why I picked it.** My diagnosis found no missed criterion, but it did find a problem none of my criteria catch: for 'oversized corduroy jacket', the data has no corduroy jacket, and search_listings scored every listing 1, so price alone decided the order and a Tie-Dye Long Sleeve came first. I picked a ranking change over a "no exact match" message because it was the smaller change, and I checked without calling the model that it changed the top result for only that one query out of the six I tried.

**How I measured it.** I added one diagnostic scenario, "near-match ranking" (query `oversized corduroy jacket`, no price cap, because both jackets in the data cost over $40 and a $40 cap removes them before ranking). It is not one of my five criteria. I wrote the PASS rule before running: PASS = the selected item's title contains the word "Jacket". Because search_listings involves no model, its five tries are identical by design, so that is not a sign caching was on. The baseline for this comparison is `results/run_2026-10-11_0025_before-b.md`, run with the same seven scenarios as the after run, `results/run_2026-10-11_0034_after.md`. The "Run Log — Before" above is from my first run, which had six scenarios.

**Did it help?** Yes, on the scenario it targeted: near-match ranking went from 0/5 (Tie-Dye Long Sleeve every try) to 5/5 (Denim Jacket every try). Nothing else moved: criteria 1, 2, 3 and 5 are 5/5 in both runs, and the selected items for the denim jacket and leather jacket scenarios are unchanged. This shows the tie-breaker does what it says; it does not show search got better in general. The scenario was built to test this fix, a Denim Jacket is still not a corduroy jacket, and the agent still presents it without saying it is a near match. None of my five criteria measure this problem, so their numbers did not change. Side effect: lower ranks moved in two other queries (for "leather jacket under $80" the Denim Jacket and Track Jacket went from 5th and 6th to 2nd and 3rd, ahead of the Leather Belt), which no criterion covers. (That ranking comes from calling search_listings directly, not from the saved run logs, which only keep the top 3 titles.)

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. Full three-tool run returns a fit card | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. Empty search stops before suggest_outfit | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. Item logged for both tools equals session["selected_item"] | 5 of 5 | PASS* | PASS* | PASS* | PASS* | PASS* | see Verdicts section |
| 4. Fit card ≤3 sentences, has a title word, no [FALLBACK] | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. Query parsed to max_price 30 and size "M" | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| near-match ranking (diagnostic, not one of my five) | PASS = title contains "Jacket" | PASS | PASS | PASS | PASS | PASS | before: FAIL 0/5, after: PASS 5/5 |

\* Criterion 3 scored on title, price and platform only (see Verdicts and Diagnoses).

Criterion 4 note: counted at most 3 sentences per card in both runs, reading as prose (a decimal price like "$75.00" or an emoji after the last period is not a sentence break). In the after run, tries 2 and 4 have exactly 3 sentences, the limit; the baseline had at most 2. I attribute this to sampling, since the ranking change does not touch the fit card prompt.

Real output (copied verbatim from the results files; search_listings is `tools.py`, trace by `trace.py::step`, report by `run_eval.py::write_report`). Scenario "near-match ranking", try 1 of each run.

Before (`results/run_2026-10-11_0025_before-b.md`):

```text
- selected_item: Tie-Dye Long Sleeve — Pastel ($17.0, depop)
[1] search_listings (via MCP)
      in:  description='oversized corduroy jacket', size=None, max_price=None
      out: 10 items: Tie-Dye Long Sleeve — Pastel, Oversized Crewneck Sweatshirt — Vintage Navy, Oversized College Crewneck — Faded Red … +7 more
```

After (`results/run_2026-10-11_0034_after.md`):

```text
- selected_item: Denim Jacket — Light Wash, Cropped ($42.0, poshmark)
[1] search_listings (via MCP)
      in:  description='oversized corduroy jacket', size=None, max_price=None
      out: 10 items: Denim Jacket — Light Wash, Cropped, 90s Track Jacket — Navy/White Stripe, Tie-Dye Long Sleeve — Pastel … +7 more
```

---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->

**Criterion 3 can't be measured as written.** The trace stores a string, not a dictionary, and logs the session's own object, so the full-dictionary comparison can't fail. I revised it in criteria.md to a weaker check (title, price and platform). Fixing it properly would mean logging a copy of what each tool actually receives and comparing that, which is a code change I held back to keep this unit to the MCP move and one improvement.

**Near matches are still presented as finds.** For 'oversized corduroy jacket' the agent now picks the Denim Jacket, but it styles it and writes a fit card with no sign that no corduroy jacket exists. I'd add a check that compares the query's keywords to the selected item and says "closest match, not exact" in the final answer. I stopped at the ranking change because one improvement was the rule and the ranking was the smaller fix.

**No criterion covers ranking, voice or caveats.** One after-run fit card (leather jacket, try 4) is written as if the poster were the seller ("Don't sleep on my shop!"), and criterion 4 can't catch that. Criterion 4 is also easy to pass because the query contains the item name, and criterion 1's 4-of-5 target may be low, though two 5/5 runs are a small sample.

**Search matches only whole words.** "sneaker" doesn't match "sneakers" (already noted in my Tool Inventory). I haven't touched it because no scenario exercises it.

**Small message tidy-ups.** The bad-key message shows the exception class name (ModelUnavailable:) and the session counter says "1 model calls". Neither changes what the user can do next, so I left them.

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

## Capstone

**What This Extends**

This extends FitFindr, my AI 201 Project 2 thrifting agent. A user types a plain-language request; the query is parsed with regex, and `agent.py::run_agent` runs three tools in order: `search_listings` through a local MCP server, then `suggest_outfit` and `create_fit_card` through Gemini via `generate()`. What is new for the capstone is an operational layer on top of that loop: a documented run path that does not depend on my editor, structured JSON-lines logging of every run including failures (written inside `run_agent()`, already built), and a measured cost per run. The agent's behavior does not change.

**The Shape**

I picked Shape A (deploy and instrument). I am adding one structured log record per `run_agent()` call, with per-stage timings for the three tools, a cost figure for one run measured with the response cache turned off, and a written run path someone else can follow without living in this repo the way I do.

What is out of scope is everything that would change how the agent thinks or presents itself. I am not adding or changing any of the three tools, not changing the planning loop or the session state, not rebuilding the command-line interface or `serve.py`, and not rewriting any other part of the system.

**Running It**

I chose Path B (local with logs). FitFindr only calls the Gemini API and a local MCP search tool, so it runs on my machine from one documented command, and every run leaves a record in the log. The exact start command, the Python version, the dependency pins, and the cache flag are in [RUNNING.md](RUNNING.md). In an earlier assistant shell, the first run of a session twice took about 180 seconds with almost no CPU use, and I could not explain it. A controlled test (sixteen minutes idle, then a cold run in a plain Terminal with the cache off) did not reproduce it: the cold run took 4.1 seconds and the warm run right after took 3.9 seconds. DNS returned only IPv4 addresses, and curl over IPv4 and IPv6 both connected in under 0.2 seconds, so IPv6 is ruled out. I did not find the cause, so I treat first-request time as unverified.

**Instrumentation**

Each `run_agent()` call appends one JSON object as a line of `logs/runs.jsonl`. The record holds `timestamp` (UTC ISO 8601), `run_id`, the raw `query`, `status`, `error_type`, `error_message`, `session_error`, `output_summary` (the chosen item title plus the character lengths of the outfit and fit-card text), `total_seconds`, `stage_seconds` for `search_listings`, `suggest_outfit`, and `create_fit_card`, `model_calls_real`, `model_calls_cached`, `prompt_tokens`, `output_tokens`, `model`, and `cache_enabled`. The write happens in a `finally` block, so a run that raises still leaves a line. `status` is `ok` when the loop finishes without a handled failure, `handled_error` when `session["error"]` is set without an exception (for example an empty search), and `error` when an exception was raised and re-raised after logging. `logs/runs.jsonl` also contains a few cache-on test lines from development.

Ten cache-off cost runs:

```text
{"timestamp": "2026-11-16T02:57:00.011064+00:00", "run_id": "ecc17762", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 464, "fit_card_len": 204}, "total_seconds": 3.191930749999301, "stage_seconds": {"search_listings": 0.6538869169999089, "suggest_outfit": 1.32449912499942, "create_fit_card": 1.2130751669992605}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 578, "output_tokens": 134, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:04.530852+00:00", "run_id": "0040e690", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 474, "fit_card_len": 211}, "total_seconds": 2.270029582999996, "stage_seconds": {"search_listings": 0.5011738749999495, "suggest_outfit": 1.2505283749997034, "create_fit_card": 0.5177071250000154}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 587, "output_tokens": 147, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:09.292216+00:00", "run_id": "47247fbf", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 468, "fit_card_len": 231}, "total_seconds": 2.4545209579991933, "stage_seconds": {"search_listings": 0.505078666999907, "suggest_outfit": 1.3828950829993119, "create_fit_card": 0.5660345840005903}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 583, "output_tokens": 143, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:14.524824+00:00", "run_id": "7570f6d8", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 484, "fit_card_len": 226}, "total_seconds": 2.970970333000878, "stage_seconds": {"search_listings": 0.5034556249993329, "suggest_outfit": 1.188791417000175, "create_fit_card": 1.2783391659995686}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 591, "output_tokens": 153, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:19.036669+00:00", "run_id": "ef56d7ea", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 474, "fit_card_len": 193}, "total_seconds": 2.2338179999987915, "stage_seconds": {"search_listings": 0.5106049580008403, "suggest_outfit": 1.1852688750004745, "create_fit_card": 0.5374617080015014}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 590, "output_tokens": 150, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:23.942847+00:00", "run_id": "1e33ecc1", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 462, "fit_card_len": 238}, "total_seconds": 2.6415108330002113, "stage_seconds": {"search_listings": 0.48957029200028046, "suggest_outfit": 1.2997012499999983, "create_fit_card": 0.851894291001372}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 580, "output_tokens": 157, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:28.534659+00:00", "run_id": "8703f96d", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 488, "fit_card_len": 220}, "total_seconds": 2.3201457080012915, "stage_seconds": {"search_listings": 0.5103949999993347, "suggest_outfit": 1.296796832999462, "create_fit_card": 0.512465957999666}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 591, "output_tokens": 156, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:33.611259+00:00", "run_id": "fc0ef8bf", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 562, "fit_card_len": 199}, "total_seconds": 2.817936583000119, "stage_seconds": {"search_listings": 0.4878179999996064, "suggest_outfit": 1.071964917000514, "create_fit_card": 1.2577597500003321}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 608, "output_tokens": 173, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:38.156719+00:00", "run_id": "1d1cf9fe", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 453, "fit_card_len": 260}, "total_seconds": 2.2813707090008393, "stage_seconds": {"search_listings": 0.5094461250009772, "suggest_outfit": 1.1773629170002096, "create_fit_card": 0.5937263329997222}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 579, "output_tokens": 149, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T02:57:42.705639+00:00", "run_id": "30391b79", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 439, "fit_card_len": 182}, "total_seconds": 2.298408708998977, "stage_seconds": {"search_listings": 0.49816141599876573, "suggest_outfit": 1.1853380000011384, "create_fit_card": 0.6144284580004751}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 584, "output_tokens": 139, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

Failed run (`status` `error`, `ModelUnavailable`):

```text
{"timestamp": "2026-11-16T02:29:12.185791+00:00", "run_id": "2e3d1140", "query": "vintage graphic tee under $30", "status": "error", "error_type": "ModelUnavailable", "error_message": "The model rejected your API key. Check GEMINI_API_KEY in your .env file, or create a fresh key at aistudio.google.com.", "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": null, "fit_card_len": null}, "total_seconds": 1.1781818340004975, "stage_seconds": {"search_listings": 0.5968053750002582, "suggest_outfit": null, "create_fit_card": null}, "model_calls_real": 1, "model_calls_cached": 0, "prompt_tokens": 0, "output_tokens": 0, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

Handled empty-query run (`status` `handled_error`):

```text
{"timestamp": "2026-11-16T02:28:59.235437+00:00", "run_id": "13e8924b", "query": "", "status": "handled_error", "error_type": null, "error_message": null, "session_error": "No listings matched (description \"\", size any, max price none). Try to try different or fewer description words.", "output_summary": {"item_title": null, "outfit_len": null, "fit_card_len": null}, "total_seconds": 0.7013444999993226, "stage_seconds": {"search_listings": 0.7007438330001605, "suggest_outfit": null, "create_fit_card": null}, "model_calls_real": 0, "model_calls_cached": 0, "prompt_tokens": 0, "output_tokens": 0, "model": "gemini-3.5-flash-lite", "cache_enabled": true}
```

**What It Costs**

Ten sequential runs of `vintage graphic tee under $30` with `AI201_CACHE=0` and a two-second pause between them averaged 2.55 seconds of `total_seconds` in the log (min 2.23, max 3.19), with mean 587 prompt tokens and 150 output tokens, or 737 tokens total per run. Those token and timing figures are measured from the log fields themselves. At the published list price of $0.30 per million input tokens and $2.50 per million output tokens, one run is about $0.00055, so 100 runs a day (one per person) is about $0.055 a day, or roughly $1.65 for thirty days; the dollar figure uses that list price rather than a billed invoice. The logged `output_tokens` count only `candidates_token_count` from the API response, not thinking tokens, so the dollar cost may be slightly understated if the model bills thinking separately.

**Run Log and Verdicts**

I scored the five capstone criteria over three trials. Latency and token means come from the stage2 cold/warm series (`AI201_CACHE=0`, query `vintage graphic tee under $30`); the bad-key, empty-query, and overlap checks reuse the earlier helper runs under `~/capstone_results/`. Every log line below was written by `agent.py::run_agent` into `logs/runs.jsonl`. The stage2 driver and scorers live outside the repo in `~/capstone_scripts/`.

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Every run under 10 s (`total_seconds`; 3 cold + 10 warm) | 13/13 | 12/13 | 13/13 | 12/13 | MISSED |
| 2. Bad key leaves an `error` record | 3/3 | 3/3 | 3/3 | 3/3 | MET |
| 3. Empty query via `run_agent` leaves `handled_error` | 5/5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Overlapping `python serve.py` asks run one after the other | 3/3 pairs | 3/3\* | 3/3\* | 3/3\* | MET |
| 5. Mean prompt+output tokens over 10 warm runs in 500–1000 | in band | 747.6 | 742.6 | 746.1 | MET |

\*First attempt on default port 5000 scored 0/3 each trial because `serve.py` could not start (macOS AirPlay holds that port). Re-run with `PORT=8765` scored 3/3. Criterion 5’s band was set after I had already measured about 737, so it could only fail if token counting broke.

**Criterion 1 — real output.** Slow trial-1 warm08 (`7c120131`) and one normal run (`a0658af3`), from `logs/runs.jsonl` via `agent.py::run_agent`, plus the matching rate-limit line from `~/capstone_results/c1_trial_1_warm08.txt`:

```text
{"timestamp": "2026-11-16T04:08:01.795315+00:00", "run_id": "7c120131", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 483, "fit_card_len": 228}, "total_seconds": 28.542547250002826, "stage_seconds": {"search_listings": 0.4759977499998058, "suggest_outfit": 1.1666533749994414, "create_fit_card": 26.898396042000968}, "model_calls_real": 3, "model_calls_cached": 0, "prompt_tokens": 603, "output_tokens": 171, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

```text
{"timestamp": "2026-11-16T04:06:58.305203+00:00", "run_id": "a0658af3", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 569, "fit_card_len": 225}, "total_seconds": 2.6673536250018515, "stage_seconds": {"search_listings": 0.5938685829969472, "suggest_outfit": 1.402196458002436, "create_fit_card": 0.6707761249999749}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 603, "output_tokens": 169, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

```text
  [rate limit] service pushed back. Waiting 26s (attempt 1 of 5). This is the limiter doing its job, not a bug.
```

**Criterion 2 — real output** (`188f1a4e`, invalid-key ask → `run_agent`):

```text
{"timestamp": "2026-11-16T03:40:03.536645+00:00", "run_id": "188f1a4e", "query": "vintage graphic tee under $30", "status": "error", "error_type": "ModelUnavailable", "error_message": "The model rejected your API key. Check GEMINI_API_KEY in your .env file, or create a fresh key at aistudio.google.com.", "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": null, "fit_card_len": null}, "total_seconds": 1.2817572909989394, "stage_seconds": {"search_listings": 0.66081183300048, "suggest_outfit": null, "create_fit_card": null}, "model_calls_real": 1, "model_calls_cached": 0, "prompt_tokens": 0, "output_tokens": 0, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

**Criterion 3 — real output** (`6a9d6df5`, `run_agent("", …)`):

```text
{"timestamp": "2026-11-16T03:40:10.081983+00:00", "run_id": "6a9d6df5", "query": "", "status": "handled_error", "error_type": null, "error_message": null, "session_error": "No listings matched (description \"\", size any, max price none). Try to try different or fewer description words.", "output_summary": {"item_title": null, "outfit_len": null, "fit_card_len": null}, "total_seconds": 0.48879916700025206, "stage_seconds": {"search_listings": 0.48845754099966143, "suggest_outfit": null, "create_fit_card": null}, "model_calls_real": 0, "model_calls_cached": 0, "prompt_tokens": 0, "output_tokens": 0, "model": "gemini-3.5-flash-lite", "cache_enabled": true}
```

**Criterion 4 — real output** (overlap pair, `5993e215` then `c6763798`):

```text
{"timestamp": "2026-11-16T03:42:22.791864+00:00", "run_id": "5993e215", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 403, "fit_card_len": 223}, "total_seconds": 2.9062702090013772, "stage_seconds": {"search_listings": 0.6079412079998292, "suggest_outfit": 1.4983136669998203, "create_fit_card": 0.7994895829997404}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 572, "output_tokens": 133, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
{"timestamp": "2026-11-16T03:42:24.889268+00:00", "run_id": "c6763798", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 513, "fit_card_len": 272}, "total_seconds": 2.093659000000116, "stage_seconds": {"search_listings": 0.37967283299985866, "suggest_outfit": 0.9704770000007557, "create_fit_card": 0.7425342090009508}, "model_calls_real": 2, "model_calls_cached": 0, "prompt_tokens": 605, "output_tokens": 178, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

```text
earlier start = timestamp - total_seconds = 2026-11-16T03:42:22.791864+00:00 - 2.9062702090013772s
             = 2026-11-16T03:42:19.885594+00:00
later start   = 2026-11-16T03:42:24.889268+00:00 - 2.093659000000116s
             = 2026-11-16T03:42:22.795609+00:00
later start (03:42:22.795609) >= earlier timestamp - 0.1s (03:42:22.691864)
```

**Criterion 5 — real output.** Trial 1 warm prompt/output pairs from `logs/runs.jsonl` (labels via `~/capstone_results/stage2_output.txt` markers):

```text
warm01 7d15a4f0  prompt=566 output=131  sum=697
warm02 35dd347b  prompt=583 output=147  sum=730
warm03 56ef3cc3  prompt=604 output=171  sum=775
warm04 23a4efb4  prompt=571 output=145  sum=716
warm05 b80c0965  prompt=585 output=150  sum=735
warm06 be659fc9  prompt=605 output=171  sum=776
warm07 59ad5d35  prompt=607 output=174  sum=781
warm08 7c120131  prompt=603 output=171  sum=774
warm09 cfd785fa  prompt=579 output=144  sum=723
warm10 02ffc343  prompt=597 output=172  sum=769
mean sum = 747.6
```

**Verdict reasoning**

Criterion 1 missed because two of three trials had a warm ask over 10 seconds on `total_seconds`, so the score was 12/13, 13/13, 12/13 rather than 13/13. Criterion 2 met: every invalid-key try exited non-zero and logged `status` `error` with `ModelUnavailable`. Criterion 3 met: every direct empty `run_agent("")` call logged `handled_error` with a non-empty `session_error` and null model-stage timings. Criterion 4 met on the `PORT=8765` re-run: overlapping asks serialized and both left `ok` lines. Criterion 5 met: all three warm-batch means sat inside 500–1000.

**Diagnosis (criterion 1)**

All nine cold asks were under 10 seconds (about 2.6 to 8.5). The asks that broke the target were warm: trial 1 warm08 at 28.5 seconds (`7c120131`) and trial 3 warm07 at 56.3 seconds, both with `model_calls_real` 3 and almost all of the time in `create_fit_card`, each matching a `[rate limit] service pushed back. Waiting Ns` line in the per-ask log (trial 1 warm08 waited 26 seconds). Trial 2 warm06 reached 8.6 seconds with the same retry pattern but still passed the under-10 cut. The slow stage is the Gemini call inside `generate.py`, retried by its backoff when the service pushes back. The warm battery sends about twenty model calls in roughly forty-five seconds (about 27 a minute), above `REQUESTS_PER_MINUTE` of 15, so the service pushed back and `generate.py` slept for the hinted delay. Slow runs followed bursts of about 13 to 24 logged calls in the previous minute, versus about 11 on many normal runs, but trial 2 warm08 had a high count and stayed fast, so a burst goes with slowness without guaranteeing it. Those counts are approximate because log timestamps are end-of-run. The per-process pacer is not the whole story: the slow overlap-test runs happened inside one `serve.py` process. I did not confirm the API’s real limit. Trial 2 cold3 took 8.5 seconds with no retry line and no separate explanation. The cold asks ran as fresh processes after sixteen idle minutes, not in a literal new Terminal window. The earlier unexplained ~180-second stalls are not shown to be the same cause; those traces showed 2 model calls, while a retried run shows 3. Criterion 1 was measurable and missed, so I am not revising it.

**The Improvement**

What I changed was two module-level counters in `generate.py` — how many times the existing rate-limit retry branch ran, and how many seconds of backoff it actually slept — plus two new fields on every `agent.py` log record, `retry_count` and `retry_wait_seconds`, written as per-run deltas the same way `model_calls_real` is. The counters increment only in that retry branch. I did not change retry logic, pacing, any delay value, stage order, tools, console output, or any existing log field. I made the change because, in the first run, I could only explain slow asks by digging through stderr files: the log showed a slow stage and an extra model call, but not that a rate-limit backoff had caused them. The instrumentation commit was `capstone: log retry_count and retry_wait_seconds per run`, made before the second run.

I scored the five criteria again over three trials (`AI201_CACHE=0`, same query; Path B helpers under `~/capstone_results_after/`). Criterion 4 used `PORT=8765` from the start.

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Every run under 10 s (`total_seconds`; 3 cold + 10 warm) | 13/13 | 12/13 | 11/13 | 7/13 | MISSED |
| 2. Bad key leaves an `error` record | 3/3 | 3/3 | 3/3 | 3/3 | MET |
| 3. Empty query via `run_agent` leaves `handled_error` | 5/5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Overlapping `python serve.py` asks run one after the other | 3/3 pairs | 3/3 | 3/3 | 3/3 | MET |
| 5. Mean prompt+output tokens over 10 warm runs in 500–1000 | in band | 747.8 | 749.9 | 726.0 | MET |

Before vs after (same rows):

| Criterion | Before (R1 / R2 / R3) | After (R1 / R2 / R3) |
|---|---|---|
| 1. Under 10 s | 12/13, 13/13, 12/13 — MISSED | 12/13, 11/13, 7/13 — MISSED |
| 2. Bad key → `error` | 3/3, 3/3, 3/3 — MET | 3/3, 3/3, 3/3 — MET |
| 3. Empty → `handled_error` | 5/5, 5/5, 5/5 — MET | 5/5, 5/5, 5/5 — MET |
| 4. Overlap serialize | 3/3, 3/3, 3/3 — MET\* | 3/3, 3/3, 3/3 — MET |
| 5. Warm token mean in band | 747.6, 742.6, 746.1 — MET | 747.8, 749.9, 726.0 — MET |

\*Before: first attempt on port 5000 failed to start; re-run with `PORT=8765` scored 3/3. After: `PORT=8765` from the start.

**Real output.** Trial 2 warm06 (`fefa0325`) from `logs/runs.jsonl`, plus the matching rate-limit line from `~/capstone_results_after/c1_trial_2_warm06.txt`:

```text
{"timestamp": "2026-11-16T18:39:04.680552+00:00", "run_id": "fefa0325", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 535, "fit_card_len": 213}, "total_seconds": 33.09847579200141, "stage_seconds": {"search_listings": 0.5708085830010532, "suggest_outfit": 31.022095500000432, "create_fit_card": 1.503560582998034}, "model_calls_real": 3, "model_calls_cached": 0, "retry_count": 1, "retry_wait_seconds": 28.267438988, "prompt_tokens": 604, "output_tokens": 171, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

```text
  [rate limit] service pushed back. Waiting 28s (attempt 1 of 5). This is the limiter doing its job, not a bug.
```

Trial 3 warm09 (`5bde21ae`) — slow with no retry, almost all time in `suggest_outfit`:

```text
{"timestamp": "2026-11-16T19:47:24.445844+00:00", "run_id": "5bde21ae", "query": "vintage graphic tee under $30", "status": "ok", "error_type": null, "error_message": null, "session_error": null, "output_summary": {"item_title": "Y2K Baby Tee — Butterfly Print", "outfit_len": 481, "fit_card_len": 234}, "total_seconds": 43.36955516600574, "stage_seconds": {"search_listings": 1.0829289169996628, "suggest_outfit": 41.51164437499392, "create_fit_card": 0.7694141250030953}, "model_calls_real": 2, "model_calls_cached": 0, "retry_count": 0, "retry_wait_seconds": 0.0, "prompt_tokens": 594, "output_tokens": 165, "model": "gemini-3.5-flash-lite", "cache_enabled": false}
```

Did it help. Nine of the 39 runs took 10 seconds or more (1, 2, and 6 per trial). Only trial 2 warm06 involved a retry; its `total_seconds` minus `retry_wait_seconds` was 4.83 seconds, so the log now explains that ask fully. The other eight — trial 1 warm06 `08dc7b5b` (13.6 s), trial 2 warm07 `8f2c3a0f` (14.5 s), and trial 3 cold1 `81ebd189` (18.0 s), warm01 `1af07cd4` (15.5 s), warm03 `6f91ec04` (11.9 s), warm07 `760fa88b` (10.7 s), warm08 `3c6d3cf0` (20.4 s), warm09 `5bde21ae` (43.4 s) — had `retry_count` 0 and spent about 6 to 41 seconds inside a single Gemini call. So the change improved diagnosis, not the target: criterion 1 got worse (12/13, 13/13, 12/13 became 12/13, 11/13, 7/13), and the new fields show that my first diagnosis (rate-limit retries) covers the earlier slow runs and one of these but not most of them. Trial 3’s median `total_seconds` was 9.8 s against about 3.5 s in trials 1 and 2. The change only adds counters, so I do not think it caused the difference. In `generate.py`, `genai.Client(api_key=key)` and `generate_content` are created and called with no `http_options`, timeout, or SDK-level retry settings; the only intentional waits this file adds are local pacing in `_wait_for_slot` and the rate-limit backoff sleep. I have not confirmed whether the extra latency came from the API, from SDK-internal retries, or from time of day (a weak candidate, since trials 1 and 2 also ran at midday and were mostly fast) (the first run was late at night; trial 3 ran mid-afternoon). I am not stating a cause.

One note on sleep: the Mac slept during idle gaps in trials 1 and 2 (the first gap lasted 9.4 hours), but every cold ask still followed at least 16 minutes with no FitFindr run, and the warm bursts (65, 97, and 157 seconds) had continuous timestamps, so sleep did not fall inside any warm burst.
