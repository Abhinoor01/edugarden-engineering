# Screenshots — what to capture

Four images, and then one paste into the README. This is the last manual step, and it's manual
for a good reason: the screenshots worth showing are the ones with **your real account's data** in
them. A guest session renders empty gauges and a bare garden, which looks worse than no screenshot
at all.

## Capture settings

- **Width: 1280px.** Narrower looks cramped in a README; wider gets downscaled and goes soft.
- **Dismiss the cookie banner first** — it floats over the lower third of the page.
- **Pick one theme and stay in it** across all four. Dark reads better for the tutor and garden;
  light reads better for the landing page. If you want them uniform, dark.
- Crop out browser chrome — the page only.

## The four

| File | Page | What it needs to show |
|---|---|---|
| `landing.png` | `/` | Scroll to the hero chat card. Capture with the **ENGLISH / HINGLISH** toggle visible — that pair is the product's first promise and the single most distinctive thing on the site. |
| `tutor.png` | `/tutor` | A real chapter open, mid-answer, with **maths rendered** (KaTeX) in the reply. A derivation or a numerical works best. Not the empty desk. |
| `home.png` | `/home` | The dashboard with a **live predicted score** — the gauge, the ± band and the coverage line all populated. This is the feature nothing else has; it deserves the slot more than the formula page. |
| `garden.png` | `/garden` | Plants at **mixed growth stages**, so it reads as earned progress rather than a decoration. |

## Then paste this into the README

Replace the `<!-- SCREENSHOTS ... -->` comment under **See it running** with:

```markdown
| | |
|---|---|
| ![Landing](assets/screenshots/landing.png) | ![Tutor](assets/screenshots/tutor.png) |
| **Landing** — every demo answer runs twice, English then Hinglish | **Tutor** — streaming chat, chapter-scoped, maths rendered |
| ![Home](assets/screenshots/home.png) | ![Garden](assets/screenshots/garden.png) |
| **Home** — the predicted board score, with its own error bar | **Garden** — one plant per subject, grown by real progress |
```

## Check before you push

Open the README on GitHub after pushing and confirm all four images resolve. Broken image icons
are worse than no images — which is exactly why this block ships commented out rather than
optimistic.
