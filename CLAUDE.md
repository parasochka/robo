# CLAUDE.md

Guidance for AI agents and developers working in this repository.

## What this repo is

A static, build-free multi-project site. Every project is a self-contained
`index.html` (plus assets) served as plain files. There is **no framework, no
bundler, no test suite, no CI build step**. Do not introduce one unless asked.

- `index.html` — landing page with a card per project.
- `roboforex-organic-analytics/index.html` — the analytics dashboard (see below).
- `compound-calculator/` — redesign concepts + production page for a compound-interest calculator.

Run locally with `PORT=3000 npm start` (uses `serve`) or any static server.

## Language & conventions

- UI copy and docs are in **Russian** — keep new copy Russian and match the
  existing tone.
- Font `Open Sans` is loaded from Google Fonts; pages degrade gracefully offline.
- Keep pages self-contained. Data is embedded directly in the HTML, not fetched.

## The RoboForex report — data model

`roboforex-organic-analytics/index.html` is a ~1.8 MB single file. Almost all of
its size is one embedded object literal:

```js
const DATA = { … };   // line ~385, a single very long line
const GSC  = { … };   // line ~386, Search Console layer (separate, hand-curated)
```

`DATA` is monthly organic-session data (55 months, `2022-01 … 2026-07`). Key fields:

| field | shape | meaning |
|---|---|---|
| `months` | `string[55]` | month labels |
| `total` | `number[55]` | site sessions per month |
| `dims` | `{country, host, section, locale, device, engine, os, topic: {label: series}}` | marginal breakdowns |
| `pages` | `{url: number[55]}` | top pages by traffic |
| `pages_other` | `number[55]` | long-tail = `total − Σ pages` |
| `pages_total_count` | `int` | universe of unique entry pages |
| `page_countries` | `{url: [[country, pct], …]}` | per-page country split |
| `dirs` | `{path: [startIdx, number[]]}` | URL-prefix index for the segment search |
| `cube_country_section` / `cube_countries` / `cube_sections` | country × section cube |
| `flags` | `{partial_last_month, anomaly_months}` | render caveats |

**Series encoding:** a series is either a plain `number[55]`, or a compressed
`[startIdx, number[]]` (zeros before `startIdx` omitted). Always expand before
math. See the `expand`/`dirSeries` helpers in the file.

**Partition invariants (must hold after any data edit):**
- `Σ dims.host[*] == total` (host is a MECE partition of total)
- `Σ dims.section[*] == total`
- `total[i] == Σ pages[*][i] + pages_other[i]`
- `dirs_covered == Σ total`

The other marginals (`country`, `locale`, `device`, `engine`, `os`, `topic`) are
independent marginals — there is **no host×dim cross-tab** in the data, so you
cannot exactly subtract a host from them.

`GSC` (clicks/impressions/ctr/position, position buckets, and the March→June 2026
loss tables) is a separate, hand-authored layer and is **not** partitioned by the
`DATA` structures. The prose in the "Search Console" panel references specific
numbers in those tables — keep them in sync if you edit either.

## Editing the embedded data

Because `DATA` is one giant line, edit it programmatically, not by hand:

1. Back up `index.html`.
2. In Python: `json.loads` the substring after `const DATA = ` up to the trailing `;`.
3. Mutate the object.
4. Re-serialize with `json.dumps(obj, ensure_ascii=False, separators=(',',':'))`,
   wrap as `const DATA = …;`, replace the single line.
5. Re-check the partition invariants above with asserts.
6. Validate: extract all `<script>` blocks and run `node --check`; then headless-
   render (Chromium is at `/opt/pw-browsers`, `playwright` under
   `/opt/node22/lib/node_modules`) and click every `#tabs button`, asserting no
   `pageerror`.

### Excluding a host (worked example)

The personal cabinet and web terminal are excluded from this report because they
are post-login traffic, not organic acquisition. Excluded host family:

```
my.roboforex.com  my.roboforex.pro  my.roboforex.org  my28.roboforex.org
webtrader.roboforex.com
```

To exclude a host cleanly:
- Subtract its host-dim series from `total`.
- Drop it from `dims.host`.
- Drop its own `section` entry if it has one; otherwise subtract its series from
  the `(other)` section so `Σ section` stays equal to `total`.
- Drop matching keys from `pages`, `page_countries`, `dirs`, and any
  `cube_sections` / `cube_country_section` entries.
- Recompute `pages_other = total − Σ pages` and `dirs_covered = Σ total`.
- Assert both partition invariants, then validate by rendering.

Hosts **not** excluded (auth-entry / other terminals / other product) but worth
flagging if the scope ever widens: `start*.roboforex.pro`, `start*.roboforex.org`,
`stockstrader.roboforex.com`, `my.roboinvesting.pro`.

The header stat, coverage %, and every share-mode number recompute live from
`DATA.total`, so they update automatically once the data is consistent. Static
tables in the GSC panel and prose are **not** data-driven — edit those by hand.

## Don't

- Don't add a build step, framework, or package that turns these into non-static pages.
- Don't hand-edit the `DATA` line character-by-character — you will corrupt the JSON.
- Don't change `contract`/model identifiers or leave temp scripts in the repo.
