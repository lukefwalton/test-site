# Site-Builder Retrospective — Cedar & Ember (test-site → test-site-snowy-sigma)

**Build:** `home`, single page, 200.4s wall time, 7/7 sections rendered, 0 hard failures, Astro build green.
**Input:** `lukefwalton/test-site` (`index.html`) — a hand-built, deliberately adversarial fixture.
**Pipeline:** `surmado/site-builder` (`capture → chunk → classify → componentize → ship`, with the refine quality-pass ON).
**Output:** `surmado-sites/test-site-snowy-sigma-vercel-app` (Astro + daisyUI rebuild).

> File/line references point at `surmado/site-builder` as of this build. Line numbers drift; treat them as
> "start here," not as permanent coordinates.

---

## TL;DR

The pipeline **shipped a buildable, self-contained, on-brand site** and handled several genuinely hard cases
gracefully — a broken image, a CSS-only carousel, missing alt text, an inline pixel-width, and a deliberately
low-contrast button. The brand/`design.md` extraction is excellent.

But it **silently rewrote and dropped customer copy**, **broke 100% of in-page navigation**, **flattened a
two-typeface brand into one**, and **ran a quality loop that never converged and whose two key auditors
(content-integrity + accessibility) never actually ran.** None of these surfaced as a build failure — the run
reported success.

The two systemic root causes:

1. **Body text fidelity is not guaranteed.** Capture caps every DOM text node at **200 characters**
   (`TEXT_CAP`), and that capped string is the *only* text source for the manifest. Anything longer — or
   anything wrapped in non-semantic `<div>` soup — is truncated or dropped, after which the LLM renderer
   **reconstructs the missing prose from the screenshot crop**. That is a hallucination surface sitting on top
   of paying customers' words, with no deterministic backstop on the default build path.
2. **The refine loop can detect but not enforce.** It correctly flagged invented colors and empty sections,
   then failed to fix them across 3 iterations and exited `converged=False`. The deterministic checks raise
   findings the LLM can't resolve, and there is no "stuck finding" detector to notice the loop is spinning.

This was a torture-test fixture and the pipeline survived it — but the failures it exposed are exactly the ones
that erode trust on real SMB sites (wrong words, dead nav, off-brand type).

---

## What this test actually was

`test-site/index.html` is **not** a normal SMB site — it's a purpose-built fixture for *Cedar & Ember Coffee
Roasters* with traps deliberately planted and labeled in the source comments:

- `(1)` a **low-contrast hero CTA** (mustard background + light-tan text — flagged in-source as intentional).
- `(2)` **inline-style traps**: an `clip-path` divider, a heading colored with inline `style="color:#b8862f"`,
  and a body block locked to `style="width:680px"`.
- **`<div>`-soup sections** ("Our Story" and "Testimonials" built entirely from `<div>`s, no semantic tags).
- `(4)` a product card carrying **~3× the body copy** of its siblings (forces height mismatch).
- a **CSS-only radio/`:checked` carousel** (one-of-N, the hard interactivity case).
- a **deliberately broken 404 image** and **three slides with intentionally omitted `alt` text**.

So this retrospective doubles as a scorecard against a known answer key.

---

## Scorecard — the planted traps

| # | Trap (from source) | Outcome | Verdict |
|---|---|---|---|
| 1 | Low-contrast CTA (mustard + tan) | Rebuilt as `btn btn-primary`; theme's `primary-content` is dark (`#1f2937`) → ~8:1 contrast | ✅ Fixed — but **incidentally** (see note) |
| 2a | Inline `clip-path` divider (`role=presentation`) | Dropped | ➖ Acceptable (decorative) |
| 2b | Inline-colored story title `#b8862f` | **Title dropped entirely** | ❌ |
| 2c | Inline `width:680px` body | Converted to responsive `max-w-2xl` | ✅ |
| 3 | "Our Story" `<div>` soup | **Fragmented; title + signature lost; body truncated then paraphrased** | ❌ Worst failure |
| 4 | "Testimonials" `<div>` soup | Rebuilt clean, copy verbatim | ✅ (survived via vision, not the manifest) |
| 5 | 3× body-copy card / height mismatch | Long copy preserved verbatim; `items-start` mismatch kept | ✅ |
| 6 | CSS-only radio carousel | Recognized as one-of-N → `data-panel-*` contract, validated | ✅ Clean win |
| 7 | Broken 404 image | Detected at download; rendered "Image unavailable" placeholder | ✅ |
| 8 | Missing `alt` on slides 2/3/5 | Synthesized from captions ("Dialing in the espresso…", "Steamed milk…") | ✅ Improvement |
| 9 | Sticky translucent nav + in-page anchors | Nav rebuilt; **every in-page anchor is dead** | ❌ |

> **Note on Trap 1:** the contrast fix is real but *unverified and unintentional*. daisyUI auto-derives a
> contrasting `primary-content`, so rendering "through the theme" fixes it as a side effect of Invariant 3 — the
> pipeline never *detected* the low contrast. The auditor that should have confirmed it (axe) failed to run on
> every iteration (see below), so we got lucky, not safe.

---

## What we did well

**1. The carousel — the hard one — was handled correctly (Invariant 2).**
A CSS-only radio/`:checked` carousel is exactly the case the `data-panel-*` contract exists for. Capture's
slide-driver couldn't reach every slide via the arrow-only controls and **correctly fell back to clone
slide-extraction** (`pipeline.log:4-5`), recovering all five slides. The renderer emitted the
`data-panel-group` / `data-panel` / `data-panel-control` contract, `wire_interactive_panels`
(`classify/render.py`) validated it (`pipeline.log:45`), and `panels.js` drives it. Output is a working
five-slide carousel with arrows + dots. No per-type Python branching, exactly as designed.

**2. The 404 image degraded gracefully (Invariant 4).**
The deliberately broken slide-3 URL was caught at download (`pipeline.log:19`, HTTP 404 skip) and the renderer
substituted a neutral `bg-base-300` "Image unavailable" tile while **keeping the caption** — instead of a broken
`<img>`. All other media (9 images) localized to content-addressed AVIF under `/assets`. Output references
nothing from the origin.

**3. Missing alt text was repaired.** Slides 2 and 5 shipped with no `alt` in the source; the rebuild
synthesized descriptive alt from the captions. The rebuild is *more* accessible than the original here.

**4. The brand extraction is genuinely strong.** `design.md` nailed the palette (amber `#cf9b2d`, espresso
`#271a10`/`#392716`, cream `#f5ecdc`), the pill buttons, 16px card radius, the two distinct shadow signatures,
and even the *voice* ("one ember per view… get it right than get it fast"). This became a clean daisyUI theme
that the sections render through with semantic tokens — Invariant 3 mostly holds, brand colors live in exactly
one place.

**5. Structured/repeating content rebuilt faithfully.** The three product cards (including the 3×-length
"Cedar Hollow" description, verbatim) and the testimonials grid came through cleanly with the intended
top-aligned height mismatch preserved.

**6. The output is shippable.** Self-contained, root-relative links, JSON-LD injected, sitemap + OG tags,
favicon fallback, `astro check` clean (0 errors/0 warnings, `pipeline.log:141`). For a one-shot capture of an
adversarial page, that's a real result.

**7. The quality pass caught real regressions — when it ran.** `pending_review.json` correctly flagged that the
story photo lost its framed-card treatment and became a full-bleed band, and `palette_adherence` correctly
flagged the invented hero color. The detectors work; the *fixers* are the problem (below).

---

## What we got wrong

Ordered by severity.

### P0 — Customer copy was silently truncated, then hallucinated back

This is the one that matters most: **we changed the words on a customer's page and reported success.**

The "Our Story" body in the source is two substantial paragraphs. In the output:

- Paragraph 1: "...learned the **rhythm of heat and time** one scorched batch at a time" became
  "...learned the **rhythms of heat and airflow** one scorched batch at a time." (**factually different**.)
- Paragraph 2: the brand's entire promise — *"rest each lot until its flavors settle into something honest — no
  shortcuts, no anonymous blends, just patient coffee from people who would rather get it right than get it
  fast"* — was **deleted** and replaced with *"rest each one long enough to matter."*

**Root cause (a precise chain):**

1. Capture serializes every DOM node's text with a hard `el.textContent.slice(0, 200)` —
   `const TEXT_CAP = 200` at `stages/capture/core.py:2268` (applied `:2387`). The comment says it exists so
   "Layer 1's text checks don't need more" (`:2257`) — i.e. it was sized for *classification*, not content.
2. The manifest's body text reads **that same truncated field** —
   `_pick_body_and_extras → n.text_content` at `stages/chunk/manifest.py:1901`. Confirmed in the artifact:
   the story section's `body` is **exactly 200 characters**, cut mid-word at "...we set up our first roaster i".
   OmniParser (Layer 3) carries **no text** (bbox-only, `chunk/layers/base.py:60`), so there is no vision text
   path to recover from — the manifest body is just the truncated string.
3. The renderer is handed the section **crop image *and*** the truncated body (`classify/render.py:10-18`). With
   the prose cut off, Sonnet **reconstructs the rest from the pixels** — verbatim where the crop is legible
   ("scorched batch" survived), invented where it had to guess ("heat and airflow"; the compressed paragraph 2).
4. The check that exists for exactly this — `check_content_integrity` — **skipped every iteration**
   (`pipeline.log:62`, `reason=no extraction/raw`). It only runs on *migrated* sites (it needs
   `extraction/raw/*.md`, which the capture→ship spine never produces — `refine/checks/content_integrity.py:238`),
   and even when present it compares list/table structure, **not prose**. So nothing compared rendered text to
   source text.

Net: **for any text node > 200 chars, content fidelity is delegated to non-deterministic OCR.** The 3× card got
lucky (legible, reconstructed verbatim); the story didn't. This violates the spirit of Invariant 5
(deterministic guardrails over best-effort LLM output) at the most important layer — the words themselves.

### P0 — `<div>`-soup headings and signatures were dropped on the floor

The story's title (`<div class="story-title">Roasted close to home, the slow way</div>`) and signature
(`<div class="story-signature">— Nora & Theo, founders</div>`) are **present in the captured `dom.json`** but
**absent from the manifest and the output** (`manifest 'Roasted close to home' → 0 hits`; section idx 8
`heading=''`).

**Root cause:** the manifest text extractors key on **semantic tags** — `_pick_heading` looks for `<h*>`,
`_pick_body_and_extras` for `<p>/<blockquote>/<address>`, `_pick_list_items` for `<li>`
(`chunk/manifest.py:1855/1901/1932`). Content wrapped in bare `<div>`s falls through all of them. The
"Our Story" eyebrow/title/signature and the "Testimonials" heading ("What our regulars say") were all
div-wrapped → none were extracted (testimonial section idx 27 also has `heading=''`).

The testimonials *looked* fine only because vision happened to reconstruct them from a legible crop — the same
non-deterministic crutch as P0 above. The story title sat on a section boundary and was simply lost. So
"div-soup is handled" is an illusion: **structured div-soup survives by luck; asymmetric div-soup gets shredded**
(here, "Our Story" fragmented into a text-block + a mis-labeled "notices" image band + dropped fragments).

### P0 — Every in-page link is dead

On a single-page site, the nav *is* the product. All 12 nav/footer anchors render as `href="/"`, and the one
surviving fragment (the hero's `/#coffees`) points at an id that doesn't exist — sections are emitted as
`id="section-10"`, never `id="coffees"`.

**Root cause:** `rewrite_nav_hrefs` (`classify/render.py:158`) → `_normalize_href_to_local`
(`chunk/manifest.py:3116`) rebuilds same-origin URLs from `path` only and **never reads or re-appends
`target.fragment`** (`:3137-3141`) — so `…/#coffees` collapses to `/`. Meanwhile section ids are sequential
`section-${index}` (`componentize/.../BlueprintSection.astro:49,73`), derived from the chunk index, **never from
a heading slug**. Nothing maps the old `#story`/`#coffees` anchors onto the new section ids. The remap feature
simply **does not exist**. Result: the whole nav and footer menu scroll nowhere.

### P1 — The brand's two typefaces were flattened into one

The source uses **Fraunces (serif) for headings** and a **system sans stack for body/eyebrows/nav**. The output
renders **everything in Fraunces serif**: `theme.css` aliases `--font-sans` to the *same* serif stack as
`--font-heading` (`stages/classify/design/theme.py:380-383`), and applies it to `body` (`:313`). The renders even
emit `font-sans` on body copy — but it's inert, because the token points at the serif.

**Root cause:** capture *does* observe two families (`capture/core.py:2417`, separate heading/body counters), but
the `design.md`-generation LLM collapsed them to one (`design/core.py:46-56` instructs Google-font substitution
with **no rule that heading ≠ body must stay distinct**), and nothing downstream detects "heading and body
resolved to the same family." A faithful two-typeface system silently becomes monotype. The whole site now reads
in a literary serif it was never meant to use.

### P1 — The refine loop never converges, and its two most important auditors never run

`refine` ran 3 iterations and exited **`converged=False`** with the *same* findings every pass
(`refine_result.json`). Mechanically:

- The two `empty_rendered_html` must-fixes are the page **header (idx 0) and footer (idx 39)** — chrome that is
  rendered via the separate chrome path but **left sitting in `manifest.sections`** with empty `rendered_html`.
  (`build_summary` says `chrome_sections_filtered: 2`, but that count means "skipped by the render loop," not
  "removed from the section list.")
- tier1 re-derives those must-fixes deterministically every iteration (`refine/checks/manifest.py:78-97`, no
  chrome guard). The LLM **cannot** clear them (it's told not to invent chrome content), and the deterministic
  cleanup `collapse_empty_sections` is **chrome-exempt** (`layer2/section_hygiene.py:162`). So they re-fire
  forever.
- There is **no "stuck finding" detector**: `detect_flapping` only catches present→absent→present *gaps*
  (`findings.py:181-211`), and these never gap, so `flapping=0`. A must-fix that's raised identically on every
  iteration is invisible to the convergence logic — the loop just burns iterations (stopped at 3 by a
  count-based no-progress guard, `core.py:696`) and quits.

And the auditors that this exact fixture was built to exercise didn't run:

- **axe (accessibility) skipped every iteration** — `status=skipped reason=empty-output rc=1`
  (`pipeline.log:72/95/117/139`). The axe CLI is on PATH but exits non-zero with empty stdout (characteristically
  a missing/incompatible headless Chrome for `@axe-core/cli`). On failure it returns `[]` and is **treated as a
  clean pass** — not even recorded in `skipped_tiers` (`auditors/axe.py:165-177`). So the contrast trap, the
  unlabeled inputs, etc., were never audited.
- **content-integrity skipped every iteration** (see P0). The two checks that would have caught the worst
  failures both no-op'd silently.

### P1 — Literal colors leak with no deterministic backstop (Invariant 3 hole)

The hero overlay shipped `style="background-color: rgba(39,26,16,0.62)"` — a literal espresso color. `refine`
flagged it `invented_color` on **every** iteration and never fixed it (`applied=1 rejected=2` each pass); it
survived to final output, alongside `rgba(0,0,0,0.45)` card shadows and `text-white`/`from-black` utilities.

**Root cause:** unlike non-local media (which has `strip_nonlocal_media_srcs` as a real guarantee), **there is no
deterministic literal-color → token rewriter** anywhere in `render.py`. Literal-color avoidance is enforced
*only* by the prompt (`design/prompt.py:64`, "NEVER emit literal colors") plus a refine check
(`palette_adherence.py:180`) that **flags but delegates the fix to the LLM** — which rejected it. Invariant 5 says
back the prompt with a deterministic post-processor; for colors, we didn't.

### P2 — Smaller cuts

- **Caption truncated:** the story photo caption lost its tail ("…still earning its keep **on Mill Street.**") —
  same vision-reconstruction-from-crop path (manifest idx 9 `body=''`).
- **Story photo reframed:** a rounded, shadowed card became a full-bleed band with the caption spilling onto the
  background (correctly caught in `pending_review.json`, never acted on).
- **Form semantics dropped:** `action`/`method`/`name`/`required`/`autocomplete` are gone (cosmetic form only,
  but it's lost fidelity and lost accessibility affordances).
- **Discovery can hallucinate diffs:** one `pending_review` proposal claims "the original full-bleed hero did not
  have a nav bar" — but the source *does* have a sticky translucent header. Acting on that would have been wrong.
- **Observability is inaccurate:** `build_summary.json` reports `design.generated: false` even though
  `pipeline.log:27-28` shows `design.md` was generated via Opus vision. A degraded build (`converged=false`, two
  auditors skipped, copy rewritten) reported overall success.

---

## The two patterns worth internalizing

1. **Text is being treated like a layout hint, but it's the product.** The `TEXT_CAP=200` cap, the
   semantic-tag-only extractors, and the vision-reconstructing renderer are all individually reasonable, but
   together they mean **the words on the page are best-effort OCR, not preserved data** — and the one guardrail
   that would catch divergence doesn't run on the default path. On a real café/dentist/plumber site, "we
   reworded your founder's story and deleted your guarantee" is the failure that loses the customer.

2. **Detection without enforcement is theater.** Refine *found* the invented color, the empty sections, and (via
   discovery) the reframed photo. It fixed none of them and couldn't tell it was stuck. A loop that re-derives
   the same must-fix three times and exits "not converged" while reporting success is worse than no loop — it
   spends tokens to manufacture false confidence.

---

## Recommendations for site-builder

Prioritized, each mapped to an invariant and an actual code site. These are directions, not patches.

### P0 — Make body text a preserved invariant, not an OCR guess
- **Stop truncating content at capture.** `TEXT_CAP=200` (`capture/core.py:2268`, and the sibling literal `200`
  at `:1877`) is sized for classification but is the *only* text source for the manifest. Either lift the cap for
  text that becomes section body, or — better — carry **full, untruncated `text_content` out-of-band** as the
  authoritative content channel, separate from the bounded classification payload. Layout is the token-budget
  problem; prose isn't.
- **Forbid the renderer from inventing prose.** The render prompt should treat manifest text as authoritative and
  **never synthesize body copy from the crop**. If text is missing, that's a capture bug to fix upstream — not a
  gap for vision to fill. Add a deterministic post-render check: every sentence in `rendered_html` must trace to
  captured source text (n-gram / fuzzy containment); flag novel sentences as a content-integrity finding.
- **Make content-integrity a non-skippable gate on the spine.** Produce the `extraction/raw` equivalent for
  fresh captures (it's just the untruncated captured text) so `check_content_integrity`
  (`refine/checks/content_integrity.py`) actually runs, and extend it from list/table structure to **prose
  fidelity**. A silent skip on the most important check should be impossible.

### P0 — Capture div-soup content, don't rely on tags
- The text extractors (`chunk/manifest.py:1855/1901/1932`) miss any heading/body/caption wrapped in bare `<div>`s.
  Fall back to **role/visual cues and computed typography** (font-size/weight/position) to recover
  heading-like and signature-like text when no semantic tag is present — this is the Invariant 6 ("behavior/style
  is ground truth, not tag names") principle the repo already applies to interactivity, extended to text.
  Regression-test against the "Our Story" / "Testimonials" div-soup fixtures.

### P0 — Keep intra-page navigation alive
- Assign section ids from **stable heading slugs** (not `section-${index}`) in
  `componentize/.../BlueprintSection.astro`, and have `rewrite_nav_hrefs`/`_normalize_href_to_local`
  (`render.py:158`, `manifest.py:3116`) **preserve and remap `#fragment`** to the matching section's slug instead
  of dropping it to `/`. Add a deterministic guardrail (Invariant 5): **no internal anchor may point at a
  fragment that has no matching `id`** — fail or auto-rewire, never ship a dead menu.

### P1 — Preserve multi-typeface brands
- The collapse is at `design.md` generation, not the theme writer. In `design/core.py`, keep heading and body as
  **distinct families** when the source uses two, and add a guard: if `tokens.heading_font == tokens.body_font`
  but capture observed two families (`capture/core.py:2417`), restore the body family (a clean system-sans stack
  is fine). Don't let `theme.py:380-383` alias `--font-sans` onto the heading serif unless the source really is
  single-family.

### P1 — Add the literal-color backstop (and fix it deterministically)
- Add a `strip_invented_colors` post-processor in `render.py` (sibling to `strip_nonlocal_media_srcs`) that maps
  hex/`rgb()`/`rgba()`/`bg-[#…]`/inline `style` colors to the **nearest theme token** (or a `color-mix()` of one).
  `palette_adherence` already *detects* these (`refine/checks/palette_adherence.py:180`) — make the fix
  deterministic instead of delegating to an LLM edit that gets rejected.

### P1 — Make refine converge, and fail loudly when it can't
- **Reconcile chrome vs body sections.** The header/footer should not remain in `manifest.sections` with empty
  `rendered_html` after being routed to the chrome path; either drop them from the structural-check surface or
  give the manifest check a chrome guard (`refine/checks/manifest.py:78`).
- **Add a "stuck finding" detector.** `detect_flapping` (`findings.py:181`) only catches gaps; add the
  complementary case — *the same finding present every iteration and never resolved* — and treat it as a terminal
  "cannot self-heal, escalate" condition rather than silently exiting `converged=False`.
- **A skipped auditor must not read as a pass.** axe returning `[]` on `rc=1` (`auditors/axe.py:165`) should
  record a real "auditor failed" status that shows up in the summary and ideally blocks a "clean" report. Make
  the a11y tier's headless-Chrome dependency a checked prerequisite. (Fixing axe also gives us automated
  verification of the contrast trap we currently only fix by luck.)
- **Tell the truth in `build_summary.json`.** Fix `design.generated` reporting, and surface
  `converged`, skipped checks, and content-integrity status prominently so a degraded build can't masquerade as a
  success.

### P2
- Carry caption text through the manifest so captions don't depend on crop OCR.
- Preserve benign form attributes (`action`/`method`/`name`/`required`) through componentize.
- Have discovery cite source evidence for each "regression" so it stops proposing fixes for differences that
  aren't real (the phantom "no nav bar" proposal).

---

## Appendix — evidence index

**Reproduce the headline findings** (from `surmado-sites/test-site-snowy-sigma-vercel-app/`):

```
# Body copy was altered before it ever reached the renderer (manifest already has the wrong words):
grep -c "rhythm of heat and time"   pages/home/content/manifest.json   # 0  (original wording)
grep -c "rhythms of heat and airflow" pages/home/content/manifest.json # 1  (altered wording)

# Story body truncated at exactly TEXT_CAP=200; title/signature never extracted:
#   manifest section idx 8: body_len == 200, heading == ''

# div-soup title/signature captured but dropped from the manifest:
grep -c "Roasted close to home" pages/home/capture/dom.json            # 1  (captured)
grep -c "Roasted close to home" pages/home/content/manifest.json       # 0  (dropped)

# Every in-page anchor collapsed to "/":
grep -oE 'href="[^"]*"' pages/home/render/page.refined.html | sort | uniq -c   # 12x href="/"

# Literal colors that survived refine:
grep -oE 'rgba?\([0-9, .]+\)' pages/home/render/page.refined.html | sort | uniq -c
```

**Key log lines** (`pipeline.log`): carousel clone-extraction fallback `:4-5`; 404 image skip `:19`;
design.md via Opus vision `:27-28`; interactive panels validated `:45`; content-integrity skipped `:62`;
axe skipped `rc=1` `:72`; refine `converged=false` `:156`.

**Key artifacts:** `refine_result.json` (`iterations:3, converged:false, flapping:0`),
`refine_findings.jsonl` (same 2–3 findings ×4 iterations), `pending_review.json` (story-photo reframe + phantom
nav-bar proposals), `design/design.md` (the strong brand extraction), `build/src/styles/theme.css`
(`--font-sans == --font-heading`, `--color-primary-content:#1f2937`).
