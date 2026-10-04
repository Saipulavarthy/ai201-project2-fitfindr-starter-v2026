# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**
Search is a plain keyword match on individual words, and the query parser is built from regular expressions, so some phrasings will be parsed wrongly or will miss a listing that a person would consider a match. Two of the three steps also call a model, which can fail or return something unusable on a given run. Four of 5 leaves room for those misses without being so loose that a broken happy path would pass.
---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
This path involves no model. It depends only on search returning an empty list and the loop checking for it, and both are deterministic code I control. If the branch works once it should work every time, so anything below 5 of 5 would mean the branch is broken, not that the path is unlucky.


---

## 3. Something about state

Given a query that matches at least one listing, the item dictionary logged by `trace.step` for `suggest_outfit` and for `create_fit_card` equals `session["selected_item"]`, in 5 of 5 tries.


**Why this target:**
Comparing the full dictionary, not just the ID, catches an item that arrives with fields altered or dropped. The check covers the hand-off from the loop to each tool, and because the loop reads the item from the session for both calls and involves no model, it should hold every time, so 5 of 5 is fair.


---

## 4. Something about the fit card

Given a matching query, the fit card is 3 sentences or fewer (hashtags do not count as sentences), contains at least one word of four letters or more from the item's title, and does not begin with the fixed fallback marker "[FALLBACK]", in 5 of 5 tries.

**Why this target:**
The fit card should be short enough to post and clearly about the right item. Starting the fallback caption with a fixed marker a model would never write lets me tell a real model caption from the hardcoded fallback. I picked 5 of 5 because these are loose checks that a reasonable caption passes however the model words it, so a miss would point to a real problem in my prompt or fallback handling and not to ordinary variation.



---

## 5. Your choice

Given the query "vintage graphic tee under $30, size M", the agent sets `session["parsed"]` to a `max_price` of 30 and a size of "M", in 5 of 5 tries.

**Why this target:**
Parsing uses regular expressions, so extraction should be reliable on a plain multi-constraint query before the values reach `search_listings`.


---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
