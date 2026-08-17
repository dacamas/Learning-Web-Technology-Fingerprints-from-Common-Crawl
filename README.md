# Learning Web Technology Fingerprints from Common Crawl

**Multi-label technology-stack detection, fingerprint-leakage ablation, and temporal generalisation analysis.**

Given the HTML of a web page, predict the technologies it was built with — as independent probabilities across 39
technologies spanning CMS, frontend frameworks, meta-frameworks, backends, CDNs, servers, analytics, payments and
CSS/UI libraries.

```
Input :  HTML + page-level evidence from a Common Crawl record
Output:  WordPress 0.98 | jQuery 0.86 | Cloudflare 0.79 | Google Fonts 0.77 | WooCommerce 0.75
```

The project's actual contribution is not the detector. It is the **measurement of how much of that detector's
accuracy is circular** — a problem most technology-fingerprinting work quietly inherits and does not report.

---

## Results

Trained on **96,487 deduplicated pages from 86,554 registrable domains** (Common Crawl `CC-MAIN-2026-30`),
evaluated on **13,001 domains never seen in training**. Full run: 80 minutes on a Colab T4.

| Model | micro-F1 | macro-F1 | micro-PR-AUC | Hamming ↓ | Subset acc. |
|---|---|---|---|---|---|
| Fingerprint rules, complete | **1.0000** | 1.0000 | 1.0000 | 0.0000 | 1.0000 |
| Logistic regression — regime A (all evidence) | **0.9987** | 0.9863 | 0.9997 | 0.0002 | 0.9904 |
| Logistic regression — regime B (signatures removed) | **0.7362** | 0.6215 | 0.8262 | 0.0499 | 0.2443 |
| Hybrid MLP — regime B, tuned thresholds | 0.7189 | 0.6059 | 0.7894 | 0.0539 | 0.1973 |
| Logistic regression — regime C (structure only) | 0.5309 | 0.3710 | 0.3811 | 0.1165 | 0.0074 |
| Frequency prior (tuned) | 0.3689 | 0.1323 | 0.3366 | 0.2768 | 0.0000 |
| Fingerprint rules, decisive rules removed | 0.2162 | 0.1446 | 0.1989 | 0.0777 | 0.0566 |
| Majority class | 0.0000 | 0.0000 | 0.3366 | 0.0884 | 0.0453 |

**The two rows at the top are not achievements.** They are the control condition — see below.

### Per-technology extremes (regime B, frozen thresholds)

| Easiest | F1 | | Hardest | F1 |
|---|---|---|---|---|
| Wix | 0.997 | | PayPal | 0.093 |
| WordPress | 0.982 | | Django | 0.093 |
| Next.js | 0.924 | | Netlify | 0.148 |
| Webflow | 0.920 | | Laravel | 0.215 |
| Joomla | 0.899 | | HubSpot | 0.253 |
| Shopify | 0.881 | | Stripe | 0.282 |

The split is not random. Technologies that emit **structural, repeated, page-wide artefacts** (a Wix asset host on
every resource, WordPress permalink and asset-path conventions, a Next.js hydration container) survive signature
removal. Technologies whose entire footprint is **one third-party script tag** (Stripe, PayPal, HubSpot) do not —
delete that tag and essentially nothing correlated remains.

---

## The central experiment: how much of this is circular?

A page is labelled `WordPress` because its HTML contains `/wp-content/`. If `/wp-content/` is then given to the
model as a feature, the model has not learned to detect WordPress — it has learned to reproduce a regex. The
resulting 0.99 F1 measures the reliability of the labelling function and nothing else.

Rather than hide this, the same models are trained over three nested feature regimes:

| Regime | Channels | What it measures |
|---|---|---|
| **A — full evidence** | text, resource/URL tokens, structure, **fingerprint rule hits** | Ceiling. Contains the label-generating signal, so it is a control, not a result. |
| **B — fingerprint-reduced** | text (redacted), resource/URL tokens (filtered), structure | The honest operating point. Signature stripped; can the rest of the page still identify the technology? |
| **C — structure-only** | DOM shape and resource-graph statistics only — no URLs, no domains, no text, no headers | Pure learned structure. |

```
A  0.9987  =  fingerprint echo  +  learnable signal  +  prior
B  0.7362  =                       learnable signal  +  prior
C  0.5309  =                       structural signal +  prior
prior 0.3689
```

**27.5% of regime A's micro-F1 is fingerprint echo** — the model reading back the rule that created its own label.
That is the number this project exists to produce. Benchmarks that train on rule-generated labels while feeding
the model those rules' signals are reporting somewhere in that band as if it were detection accuracy.

What remains is real. Regime B holds **0.7362** with every matched signature redacted from text, URL tokens and
resource domains, and with the header and cookie channels dropped entirely. Regime C — which never sees a single
vendor string — reaches **0.5309** against a tuned prior of 0.3689.

The most interesting sub-result: **server and CDN labels are derived from HTTP response headers, yet remain
partially predictable from client-side HTML alone.** Nginx reaches F1 0.555, Apache 0.525 and Cloudflare 0.787 in
regime B. Frameworks and hosting stacks impose a measurable morphology on the documents they emit.

---

## Technology co-occurrence

| Pair | Jaccard | P(B\|A) | Lift |
|---|---|---|---|
| Nuxt.js → Vue.js | 0.537 | 1.000 | 83.4 |
| ASP.NET ↔ Microsoft IIS | 0.495 | 0.560 | 22.8 |
| Next.js → React | 0.408 | 1.000 | 248.8 |
| Google Fonts ↔ WordPress | 0.427 | 0.630 | 1.49 |
| Font Awesome ↔ jQuery | 0.368 | 0.725 | 1.93 |

Two of these are artefacts of the labelling design — see limitation 3. ASP.NET ↔ IIS at lift 22.8 is a genuine
finding: two independently-derived labels (an `X-AspNet-Version` header and a `Server` header) confirming the same
Microsoft deployment pattern.

K-means over technology vectors yields 12 stack archetypes (silhouette 0.145), recovering recognisable ecosystems
without supervision: `Apache + WooCommerce + PHP + WordPress` (9.8% of pages), `LiteSpeed + WooCommerce +
WordPress + PHP` (7.6%), `Vercel + Next.js + Tailwind CSS` (4.3%).

---

## Temporal generalisation

A second, harder protocol: train on crawls up to 2021, validate on an intermediate crawl, test on 2026. Domain
disjointness is enforced *across time* as well, so a site captured in both periods cannot leak.

| Protocol | micro-F1 | macro-F1 | micro-PR-AUC |
|---|---|---|---|
| Domain-disjoint (same crawl) | 0.7362 | 0.6215 | 0.8262 |
| Temporal (train ≤ 2021 → test 2026) | 0.5695 | 0.2442 | 0.5644 |

Micro-F1 falls 22.6%. **Macro-F1 falls 61%** — the far more revealing number. The degradation is not uniform
decay; it concentrates in technologies scoring ~0.0 on the temporal test because their fingerprints did not exist
or were not detectable in the training period. Wix, Google Tag Manager, Tailwind CSS and Shopify all drop from
above 0.89 to 0.000.

Adoption trends across the sampled crawls (2013 → 2026) are directionally consistent with the known history of the
web — Font Awesome +1706%, Bootstrap +722%, Cloudflare +516%, Tailwind CSS from absent to 5.2% — which is a sanity
check on the labelling, not a discovery.

---

## Pipeline

```
Common Crawl index discovery (collinfo.json, live)
   ↓  threaded streaming of stride-sampled WARC files, per-domain caps, manifest checkpointing
evidence extraction  — one lxml extractor shared by training AND inference
   ↓  DOM stats, scripts, resource graph, class tokens, meta tags, whitelisted headers, cookie NAMES
fingerprint registry — 45 technologies, ~150 rules, HIGH/MEDIUM/LOW/ABSTAIN confidence ladder
   ↓  labels + a TRUST MASK: uncertain cells excluded from loss and from metrics
deduplication — exact URL → normalised URL → content hash → SimHash near-duplicates
   ↓  domain-disjoint 70/15/15 split, asserted disjoint
rule baselines → classical ML → hybrid neural model
   ↓  regime A/B/C ablation, per-label threshold tuning, isotonic calibration
error analysis · co-occurrence · stack clustering · temporal · adoption · transitions
   ↓
HTML-only Gradio inference interface
```

Deduplication removed 3,516 of 100,003 records (3.52%): 3 exact-URL, 17 normalised-URL, 532 content-hash and
2,964 SimHash near-duplicates.

### Design decisions worth noting

**WARC, not the columnar index or CDX.** WARC is the only source yielding HTTP response headers *and* full HTML in
one sequential pass, which lets the same extractor serve training and inference — eliminating train/serve skew by
construction. CDX would have meant targeted per-URL retrieval; the columnar index carries no page content.

**Trust masking.** Weak supervision produces a confidence ladder. Only `HIGH` becomes a positive; `MEDIUM`/`LOW`
cells are masked out of **both** the loss and the metrics (2.7% of cells) rather than forced into a binary.
`MaskedOvR` trains one classifier per technology on only that technology's trusted rows — standard
`OneVsRestClassifier` cannot express this.

**Domain-disjoint splitting.** Every page on a domain shares one stack, one CDN, one template. A random page-level
split would let the model memorise a domain in training and recognise it in test, reporting page recognition as
technology detection.

**Thresholds frozen before test.** Per-technology thresholds tuned on validation, then frozen. Tuning lifted
validation micro-F1 from 0.675 to 0.719.

---

## Running it

Upload `common_crawl_web_technology_fingerprinting.ipynb` to Google Colab, select a GPU runtime, and run top to
bottom. Drive mounts automatically for a persistent cache. The only cell intended for editing is `CONFIG`.

```python
CONFIG["DATASET_MODE"] = "quick"     # ~10k pages,  ~15 min
CONFIG["DATASET_MODE"] = "standard"  # ~100k pages, ~80 min   ← the run reported here
CONFIG["DATASET_MODE"] = "large"     # ~500k pages, 4-7 h, ~60 GB transfer
CONFIG["CRAWL"] = "CC-MAIN-2026-30"  # pin to reuse a cached corpus; "auto" takes the newest
CONFIG["OFFLINE_DEMO"] = True        # synthetic corpus, no network — validates the pipeline
```

Ingestion is resumable via a manifest, so a disconnect costs one archive file rather than the run.

**Outputs** land under the cache root: `datasets/` (train/validation/test Parquet with labels and masks),
`models/` (bundle, thresholds, registry), `reports/` (`final_report.md`, per-technology CSVs, ablation, temporal,
co-occurrence), `figures/` (13 plots), `experiment_summary.json`.

**Requirements:** ~13 GB RAM (peak RSS 6.25 GB), GPU optional. Every optional dependency (`torch`,
`sentence-transformers`, `gradio`, `warcio`) has a documented fallback.

---

## Limitations

Ordered by how much they should change your reading of the results.

**1. Weak supervision caps the ceiling, and the metrics cannot see it.** Labels come from rules, so where a rule is
wrong the model is trained to be wrong identically — and evaluation against those same labels is blind to it.
Reported scores measure agreement with the registry, not ground truth. No human-validated label set exists here.

**2. "Trusted negative" means "no visible evidence", not "absent".** A cell becomes a negative when no rule fires.
For technologies whose markers are easily stripped, proxied away, or bundled, this conflates absence with
invisibility. Recall is therefore optimistic and the true false-negative rate is higher than the tables show.

**3. Two headline co-occurrence figures are artefacts of the label design.** `P(React | Next.js) = 1.000` and
`P(Vue.js | Nuxt.js) = 1.000` are **not empirical findings**. Implication rules (Next.js ⇒ React) record the
implied label at `MEDIUM`, which the trust mask excludes. The only Next.js pages retaining a trusted React label
are those with independent direct React evidence — so the conditional is 1.000 by construction. The same applies
to WooCommerce ⇒ WordPress/PHP, Laravel ⇒ PHP, Drupal ⇒ PHP and Joomla ⇒ PHP. Lift values for these pairs are
meaningless. Non-implied pairs (ASP.NET ↔ IIS, Font Awesome ↔ jQuery) are unaffected.

**4. No JavaScript is executed.** Common Crawl archives pre-hydration HTML. Any technology manifesting only after
client-side rendering is systematically invisible. This is the single largest source of false negatives, and it
means **React's measured prevalence is a floor, not an estimate** — a production React build compiled through
Webpack or Vite often emits no framework-specific marker at all. React reaches only 0.743 F1 at 0.650 recall
partly for this reason.

**5. Regime B redaction is approximate.** It removes the matched signature, not everything correlated with it.
Permalink shapes, asset-path conventions and template idioms survive redaction, so regime B retains residual
leakage and its 0.7362 should be read as an upper bound on the honest operating point. Regime C is the clean
condition — which is precisely why it is included.

**6. Regime C's margin over the prior is real but modest.** micro-F1 0.5309 vs 0.3689 looks decisive, but the
threshold-independent comparison is much tighter: micro-PR-AUC 0.3811 vs 0.3366. Structure carries signal; it does
not carry a lot of it.

**7. The neural model lost to logistic regression.** The hybrid MLP (0.7189) underperformed the linear regime-B
model (0.7362) on every headline metric. Reported as-is. The likely cause is that the MLP consumes *compressed*
inputs (384-d MiniLM embeddings + 192-d resource SVD) while the linear model sees the full ~120k sparse feature
space, and technology fingerprinting rewards exact sparse token matches over dense semantics. **The neural
architecture is not doing useful work here**, and the linear model is the more honest headline.

**8. Calibration is reported in-sample and did not help.** Mean ECE 0.0526 raw; the isotonic figure of 0.000 is
meaningless because it is measured on the same validation data the calibrator was fitted to. On the held-out test
set calibration *hurt* — macro-F1 fell from 0.606 to 0.441 at frozen thresholds, since isotonic mapping shifts the
probability scale those thresholds were tuned against. Use the uncalibrated model.

**9. The transition analysis is anecdotal.** Only **72 domains** appear in both the 2013 and 2026 samples — far too
few to support claims about migration rates. Broad per-domain caps optimise for coverage, not for recapturing the
same sites across crawls. Treat that section as a demonstration of method, not a result.

**10. Adoption trends confound genuine change with sampling change.** Common Crawl's seed selection, host
allocation and frontier policy have shifted since 2013, and this project layers its own per-domain cap and
archive-file stride sample on top. A rise in measured prevalence may reflect adoption *or* a change in what the
crawler chose to fetch. Reported confidence intervals account only for binomial sampling noise, not for the much
larger composition effect.

**11. Header-derived technologies cannot be detected from pasted HTML.** Server, CDN and hosting labels come from
HTTP response headers present during training but absent at inference. The interface flags these; their
predictions rest entirely on learned structural correlates.

**12. Six technologies were dropped for insufficient support** (Gatsby, Hotjar, Matomo, Meta Pixel, Squarespace,
Svelte — each below 150 trusted positives). Their absence is a modelling decision, not a claim about prevalence.

**13. Common Crawl is a sample, not a census.** Every prevalence figure describes this ~96k-page sample.
"WordPress evidence appeared on 41% of pages in this sample" is supportable; "41% of the web runs WordPress" is
not.

**14. Evidence is not infrastructure.** A Cloudflare header means traffic passed through Cloudflare, not that the
origin is hosted there. One WordPress marker does not make a whole domain WordPress. Only observable client-side
evidence is measured.

---

## Ethics and scope

Entirely passive. All data comes from Common Crawl, a public archive of already-published pages.

**Not performed:** live crawling or fetching, port scanning, host discovery, vulnerability detection or
exploitation, authentication or access-control bypass, credential testing, brute forcing, secret or API-key
discovery, active reconnaissance of any target.

**Data minimisation:** `Set-Cookie` values are discarded at parse time — only cookie *names* are retained. Request
headers are never captured; only a whitelist of response headers is stored. Software **versions are deliberately
not parsed**, removing the natural bridge from fingerprinting to vulnerability mapping. Raw HTML is never written
to disk.

The inference interface accepts pasted or uploaded HTML and **has no URL field by design** — adding one would turn
a passive analysis tool into an active scanner.

---

## Repository contents

```
common_crawl_web_technology_fingerprinting.ipynb   the full pipeline (100 cells, ~4,300 lines)
README.md                                          this file
reports/                                           final_report.md, per-technology CSVs, ablation,
                                                   temporal, co-occurrence, calibration, errors
figures/                                           13 generated plots
experiment_summary.json                            reproducibility record
```

Run metadata: crawl `CC-MAIN-2026-30`, registry `v1.0.0`, seed 42, 96,487 pages / 86,554 domains, 39 of 45
technologies modelled, 80 minutes on a Colab T4.
