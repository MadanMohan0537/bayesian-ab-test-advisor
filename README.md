# Bayesian A/B Test Advisor

**Turn experiment data into a ship / hold / keep-running decision that a launch review can act on: probability of improvement, expected lift with an interval, guardrail checks, and expected loss.**

[![CI](https://github.com/MadanMohan0537/bayesian-ab-test-advisor/actions/workflows/ci.yml/badge.svg)](https://github.com/MadanMohan0537/bayesian-ab-test-advisor/actions/workflows/ci.yml)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-3776AB.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

*Previously named `AB-advisor`. The app title, CLI, and module names are unchanged (`AB Advisor`, `python -m src.cli`, `src/`).*

![Static render of the checkout analysis: decision banner, conversion posteriors, lift, and a forest plot](screenshots/dashboard.png)

The figure is produced by `scripts/render_screenshots.py` (matplotlib), not a browser capture of the Streamlit app. That script draws 12,000 posterior samples, so the pictured intervals can differ slightly from the 20,000-draw table in [Sample data](#sample-data-known-effects).

## The product problem

Product teams run A/B tests to decide whether to ship, but the usual output, a p-value, answers "how surprising is this data if nothing changed?" rather than "should we ship, and what do we risk if we're wrong?" Teams also peek early, let a pretty secondary metric outvote a weak primary one, or miss that traffic didn't split as designed.

## Who it is for

Product managers, analysts, and engineers who present experiment results in launch reviews, especially people who aren't statisticians but must make and defend the call.

## Approach

- Fit a **closed-form conjugate Bayesian model per metric** (conversion, revenue with zeros, counts, continuous, skewed). Summaries use Monte Carlo draws from those posteriors, not an MCMC sampler.
- Make product judgments explicit controls: **prior, minimum useful lift (MDE), ROPE, P(better) ship threshold, and metric roles**. The P(beat MDE) bar (default 80%) and the expected-loss ratio (default 0.25) live in `DecisionThresholds`; they are not sidebar sliders.
- Roll per-metric results into one **experiment-level recommendation**: ship, hold, keep running, or investigate (sample-ratio mismatch).
- Write a plain-English summary from a deterministic template; an optional LLM may only rewrite the already-computed numbers.

## Outcomes so far

- Streamlit dashboard (analyze, sample-size planner, how-to-read guide), CLI for batch reports, notebook walkthrough, and example review memos.
- 33 automated tests, run in CI on Python 3.10, 3.11, and 3.12.
- Checked against **synthetic datasets with known effects** (below). It has not yet been used on a real product experiment.

---

The quote below uses made-up figures. It is not the bundled sample (that conversion lift is about +31%, not +24%) and it is not a result from your data:

> There is a **99.9%** probability the new checkout increases conversion, with an expected lift of **+24%**. The 90% highest-density interval is **[+17%, +32%]** — it does not include zero. Expected loss if you ship is tiny compared with holding.

That's not a p-value. A p-value answers *"how surprising is this data if nothing changed?"* Nobody walks into an exec review to discuss surprisingness — they walk in to decide whether to ship.


## Read a recommendation in context

Choose a primary metric, define the minimum useful effect and identify guardrails before looking at outcomes. Then inspect assignment quality, model choice, posterior uncertainty and expected loss together.

The sample datasets are synthetic demonstrations. Their results are not evidence for a real checkout launch. Optional model-written narratives summarize the analysis; the numerical engine and report remain the evidence to review.

For reproducibility, retain the input CSV, metric configuration, prior settings, thresholds and report. Begin with [CLI reporting](src/cli.py) for a repeatable batch run or [the dashboard](app.py) for interactive exploration.

## Why Bayesian (and why it's harder)

| Frequentist t-test / z-test | This tool |
|---|---|
| p = 0.04 | P(treatment > control) = 97% |
| "Significant" vs "not" | Expected lift, 90% HDI, chance of beating the MDE |
| Fixed-horizon p-values are invalid if you peek | Posterior and expected-loss rules can support sequential decisions under a predeclared policy (see caveats below) |
| Binary conversion only in the simple case | Conversion, revenue (including zeros), counts, continuous, skewed |

The paired numbers in the first row are a contrast, not an output of this repository.

The cost is real: you have to pick a **prior**, a **minimum useful lift (MDE)**, a **ROPE**, and a **P(better) ship threshold**. Those are product judgments. The sidebar exposes HDI mass, MDE, ROPE, the ship threshold, posterior draw count, and an optional historical-conversion prior. Metric role and direction are set on the analyze tab. The expected-loss ratio is a code default (0.25), not a sidebar control.

## Quick start

`python` below is the virtualenv's Python 3.10+ (`python3` on systems where `python` is not on `PATH`).

```bash
git clone https://github.com/MadanMohan0537/bayesian-ab-test-advisor.git
cd bayesian-ab-test-advisor
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python -m src.data_gen           # writes the three CSVs in data/
streamlit run app.py
```

Open the app, leave **Checkout redesign (clear win)** selected in the sidebar, and click **Run Bayesian analysis**.

### Headless / CI

```bash
python -m src.cli data/sample_experiment.csv --name "Checkout redesign" --out docs/experiment_review.md
pytest -q
```

`--out` creates that path; the checked-in memos are [`docs/sample_experiment_review.md`](docs/sample_experiment_review.md) and `docs/generated_experiment_review.md`.

### Optional LLM narrative

The written summary is a deterministic template by default — no external calls, no dependency on an API key. If you set an API key, the tool asks an LLM to *rewrite* the same computed numbers into a tighter narrative; the model only ever sees the structured JSON output of the Bayesian engine, never the raw data, and is instructed never to invent a figure. If the key is missing or the call fails for any reason, the app falls back to the template.

**DeepSeek** (OpenAI-compatible) is the default provider:

```bash
export DEEPSEEK_API_KEY=sk-...
streamlit run app.py
# or, from the CLI:
python -m src.cli data/sample_experiment.csv --name "Checkout redesign" --llm
```

Default endpoint `https://api.deepseek.com`, default model `deepseek-v4-flash`. When the model name contains `v4`, the client sends `thinking` disabled. Override the model with `DEEPSEEK_MODEL` or the sidebar's model field.

The dashboard can switch provider to `openai` (default model `gpt-4o-mini`) when `OPENAI_API_KEY` is set. The CLI `--llm` flag always calls the DeepSeek-compatible endpoint; `LLM_PROVIDER=openai` does not redirect it.

## Input format

The loader rejects unsafe inputs before fitting a model: the file must contain exactly two distinct arms, every user must belong to only one arm, wide data must have one row per user, and each analyzed metric must have at least one usable observation in both arms. Missing metric values are dropped independently within each arm when that metric is split; infinite values are rejected. `expected_split` must lie strictly between 0 and 1. The app and CLI do not expose that argument; they assume a 50/50 split.

**Wide** — one row per user:

```text
user_id,variant,converted,revenue,time_on_page_sec,events_count,bounce
c-00001,control,0,0.00,162.4,2,1
t-00018,treatment,1,54.20,171.1,4,0
```

**Long** — one row per user × metric:

```text
user_id,variant,metric_name,metric_value
c-00001,control,converted,0
c-00001,control,revenue,0.00
```

The two blocks above show the column layout. They are not rows copied from `data/sample_experiment.csv`.

User, variant, and metric columns are auto-detected from common aliases (`variant`/`group`/`arm`/`bucket`, `user_id`/`uid`/`visitor_id`, etc.). Variant labels `A`/`B`, `0`/`1`, `control`/`treatment` are normalized automatically. Multiple metrics run in one pass, and each gets its own conjugate model — see below.

## What the engine actually does

No PyMC sampling loop — each metric family has a closed-form conjugate posterior. Monte Carlo (20,000 draws by default) is used only to turn two posteriors into P(better), lift, HDI, and expected loss.

| Metric pattern | Model | Default prior |
|---|---|---|
| 0/1 conversion, bounce | Beta–Binomial | Beta(1, 1) — uniform |
| Continuous (time on page) | Unknown mean & variance → Student-t on the mean | Reference (flat) prior |
| Strictly positive & skewed | Log-normal mean | Inverse-χ² on log-variance |
| Counts (events) | Gamma–Poisson | Gamma(1, 1) |
| Revenue with many zeros (ARPU) | Hurdle: Beta conversion × log-normal spend given convert | Same as above |

Metric type is auto-detected from the values (binary, integer counts in a bounded range → Poisson, many zeros and skewed positive values → hurdle log-normal, strictly positive and skewed → log-normal, else Normal) and can be overridden per metric in the UI.

For each metric the report includes:

1. **P(treatment mean > control mean)**. The metrics table always reports that direction. For "lower is better" metrics (bounce, latency, and the other names in `default_higher_is_better`), the decision engine flips it and scores P(improvement), P(harm), and expected loss in the improving direction.
2. **Expected relative lift** — mean of the posterior on (treatment − control) / control.
3. **90% HDI** (configurable from 80% to 99% in the sidebar, or `--cred` on the CLI) — shortest interval containing that share of the posterior lift.
4. **P(lift > MDE)** and **P(harm beyond MDE)** — probability of clearing your minimum useful lift, or of a harmful move past it.
5. **ROPE probability** — share of posterior mass inside a "too small to care about" band. A metric is marked `stop_futility` when that share is at least 80% and P(lift > MDE) is under 20% (both defaults).
6. **Expected loss** of shipping vs. holding, on the metric's raw scale. For lower-is-better metrics the decision engine swaps the two.
7. A frequentist p-value in the dashboard metrics table, with a caption that it is only a translation aid. The Markdown report does not include it.

### Expected loss — the actual decision rule

If you ship the treatment, you lose whenever it is actually worse; expected loss is the average size of that loss across the posterior. For one metric, `src/decisions.py` returns `ship` when P(improvement) is at least the ship bar (default 95%) **and** either:

- P(beat MDE) is at least 80%, or
- expected loss of shipping is at most 0.25 × expected loss of holding (and holding's expected loss is positive).

The loss-ratio path still requires the ship bar. It does not ship on a small expected loss alone.

## Decision engine

Every metric gets a role — **primary**, **guardrail**, or **secondary** — inferred from its name (`conversion`/`converted`/`revenue`/`purchase`/`checkout`/`signup` → primary; `bounce`/`error`/`latency`/`refund`/`churn` and related tokens → guardrail; everything else → secondary) or set explicitly. Per-metric actions (`ship` / `hold` / `stop_futility` / `keep_running` / `block`) roll up into one experiment-level call:

- **Ship** — there is at least one primary metric, every primary action is `ship`, and no guardrail action is `block` or `hold`.
- **Hold** — a guardrail action is `block` or `hold` (this blocks shipping even if a primary metric won), or at least one primary action is `hold` or `stop_futility` and none is `ship` or `keep_running`.
- **Keep running** — the remaining case, including a mix of primary `ship` and primary `keep_running`, and the case with no primary metric. Secondary metrics are reported and do not change this call.
- **Investigate** — a sample-ratio-mismatch (SRM) check on the actual vs. expected control/treatment split fails a chi-square test (default flag when p < 0.001). This is checked first. The tool will not recommend shipping until it is cleared.

A guardrail is `block` when the chance of harm beyond the MDE is at least 5% and P(improvement) is below 50%.

## Dashboard

Three tabs:

1. **Analyze experiment** — upload a CSV or pick a bundled sample; confirm the user/variant/metric columns and per-metric type, role, and direction (all auto-detected, all overridable); set HDI mass, MDE, ROPE, the P(better) ship threshold, posterior draws, and an optional historical-conversion prior; run the analysis to get KPI cards, an overview chart, a forest plot across metrics, posterior density overlays, the lift distribution with shaded HDI, expected-loss bars, a probability gauge, the written narrative, and a downloadable Markdown/HTML report (the HTML button is labeled for printing to PDF).
2. **Sample size planner** — simulate binary experiments at a hypothesized relative lift and report how many users per arm you need before P(better) clears your ship threshold in at least the chosen share of simulations (the button runs 200 simulations per grid point).
3. **How to read results** — a plain-English guide to the launch-review question, metric roles, priors, peeking, and limitations. It discusses P(better), expected lift, the HDI, and expected loss. ROPE is a sidebar control and is defined in the downloadable report; that tab does not define it.

## Repository layout

```text
app.py                         Streamlit UI (3 tabs: analyze, sample size, guide)
src/bayesian.py                Conjugate posteriors + Monte Carlo summaries
src/metrics.py                 Wide/long loaders, column detection, type inference, SRM check
src/decisions.py               Ship / hold / keep-running / investigate rules
src/summarize.py               Deterministic templates + optional DeepSeek/OpenAI rewrite
src/visualize.py               Plotly posteriors, lift, forest plot, gauges
src/report.py                  Markdown/HTML experiment review + sample size planner
src/data_gen.py                Synthetic checkout + underpowered datasets (known effects)
src/cli.py                     Batch report generation from a CSV, no Streamlit required
scripts/render_screenshots.py  Rewrites the three sample CSVs, screenshots/*.png, and docs/generated_experiment_review.md
data/                          Bundled sample CSVs (wide, long, underpowered)
notebooks/bayesian_demo.ipynb  Walkthrough of the math, using the checkout generator
docs/                          Example experiment-review memos (see below)
tests/                         Engine, loaders, decision rules, validation, LLM fallback (pytest)
```

## For product managers

### Choosing metrics

- **North star / primary** — the reason the experiment exists (conversion, revenue). Pre-register one before launch. A secondary `ship` does not outvote a primary that is still `keep_running`, and a secondary `hold` does not block a primary `ship`.
- **Guardrails** — bounce, errors, refunds, latency. A guardrail action of `block` or `hold` **blocks shipping**, even if the primary metric won.
- **Secondary** — time on page, event counts. Learning only. An inconclusive secondary metric does not stall a clean primary win.

### Setting priors from history

If last quarter's checkout converted at 10% and you'd trust that as roughly 100 users' worth of information (not 100,000 — you don't want the past to dominate a live test), set:

- Prior mean = 0.10, strength = 100 → Beta(10, 90) in the sidebar.

The sidebar applies that mean and strength only when the mean is above 0 and the strength is at least 2. Otherwise the prior stays Beta(1, 1). The strength input defaults to 2 and is ignored while the historical mean stays at 0. The prior's influence shrinks as the experiment's own `n` grows.

### Talking to executives

Don't say "we got significance." Say:

1. **Probability the change is positive.**
2. **How big we think it is** — expected lift and its interval.
3. **What we risk if we're wrong** — expected loss, guardrail status.
4. **What we'll do next** — ramp, hold, or wait for the pre-registered bar.

[`docs/sample_experiment_review.md`](docs/sample_experiment_review.md) is the tone to copy for a real write-up.

### Peeking and sequential monitoring

Bayesian posterior updates can support sequential decisions under an appropriate model and predeclared decision policy. A 95% posterior threshold alone does not guarantee a frequentist false-positive rate or protect against selection, multiple metrics or misspecified priors. Validate stopping rules for your design. Do not interpret a repeatedly inspected fixed-sample p-value as having fixed-horizon guarantees.

## Sample data (known effects)

`data/sample_experiment.csv` matches `generate_checkout_experiment()` — 5,000 users per arm, seed 42. True rates in the generator are conversion 10% vs 12.5%, bounce 42% vs 40.5%, events 3.0 vs 3.05, and time on page 180s vs 180s. The table is the **observed** file plus the default analysis (20,000 posterior draws, seed 42, MDE 2%, ROPE ±1%, 90% HDI):

| Metric | Control | Treatment | What you should see |
|---|---|---|---|
| `converted` | 9.86% | 12.94% | `ship` — P(better) = 100% (all 20,000 draws), expected lift +31.4% |
| `revenue` (ARPU, zeros included) | $5.39 | $7.56 | `ship` via the hurdle model, expected lift +40.8% |
| `time_on_page_sec` | 179.7s | 179.9s | Null — 90% HDI [−0.64%, +0.82%], includes 0, action `stop_futility` |
| `events_count` | 3.03 | 3.08 | `keep_running` — P(better) = 92.7%, P(lift > MDE) = 39.7% |
| `bounce` | 41.6% | 40.3% | Lower is better. Action `keep_running`; it does not block shipping |

Run `python -m src.cli data/sample_experiment.csv --name "Checkout redesign"` to reproduce that run. The experiment-level call is **ship**.

`data/sample_underpowered.csv` matches `generate_underpowered_experiment()` — same columns, 400 users per arm, seed 7, true conversion 10% vs 10.8%. The other true means are not the checkout experiment's (time on page is centered near 120 seconds). With default thresholds the experiment-level call is **keep running**, because `converted` and `revenue` stay below the ship bar. A secondary can still look decisive on its own in this file (`time_on_page_sec` is `ship`); secondaries do not change the experiment-level call.

`data/sample_experiment_long.csv` is the same checkout data in long format. The default analysis also calls it **ship**.

## Limitations (read before you ship)

- **Independence / SUTVA.** Users are assumed not to interfere with each other. Marketplace, social, or shared-cache effects need a different design.
- **Assignment must be random and logged correctly.** A sample-ratio mismatch is flagged with a chi-square test (default p < 0.001); if it fires, the recommendation is *investigate*, not ship, regardless of how good the metrics look.
- **Model family must match the metric.** A skewed revenue column forced through a Normal model will lie. Auto-detection is a starting point — override the type in the UI if it looks wrong.
- **Multiple metrics inflate false discovery.** Watching a dozen KPIs raises the chance something looks good by chance. Pre-register the primary metric; treat the rest as guardrails or learning.
- **No CUPED / variance reduction, no switchback design, no clustered users.** Those are reasonable extensions, not in this version.
- **The LLM can't invent numbers, but it can still over-smooth a caveat.** Prefer the template narrative when the decision is contentious or the write-up is going somewhere high-stakes.
- **Thresholds you cannot see in the sidebar still decide the call.** P(beat MDE) ≥ 80%, ROPE futility ≥ 80% with P(beat MDE) < 20%, guardrail harm ≥ 5%, and the 0.25 loss ratio are defaults in `DecisionThresholds`. The sidebar's ship slider changes only the P(better) bar.

## Tests

```bash
pytest -q
```

33 tests, collected from `tests/`. They cover a known conversion lift with P(better) above 0.99, a true null whose HDI contains 0, a Poisson rate shift, the hurdle revenue model, wide and long CSV loading, rejection of a single arm, users in both arms, duplicate wide rows, and an invalid expected split, per-arm dropping of missing values, SRM detection, a ship call on a clear primary win, an investigate call when SRM is flagged, metric-role heuristics, and the template fallback when no API key is set. `test_underpowered_keeps_running` checks that a small conversion lift is not treated as near-certain; it allows `keep_running`, `hold`, or `ship`, so it does not lock the experiment-level action. There is no test that asserts an experiment-level `hold`.

GitHub Actions (`.github/workflows/ci.yml`) runs that suite on Python 3.10, 3.11, and 3.12 for pushes to `main` and for pull requests.

## What I would measure next

- Agreement between the tool's recommendation and an experienced analyst's call on real, permitted experiment data.
- Time from data export to a written launch-review memo.
- Frequency of SRM and guardrail blocks caught before a launch decision.

None of these has been measured yet.

## Deployment

Cloudflare Workers is the planned deployment target. It is not configured in this repository: there is no `wrangler.toml`, no Worker entry point, and no live `workers.dev` URL. Nothing here is deployed.

When a Worker exists, publish it with:

```bash
npx wrangler deploy
```

That command deploys the Worker script directly. It does not upload a version, roll out a percentage of traffic, or publish a tagged release.

Still needed before that command can succeed:

- a `wrangler.toml` (Worker name and settings)
- a Worker entry point
- for this Python code, either [Python Workers](https://developers.cloudflare.com/workers/languages/python/) or a JavaScript/TypeScript port — the Streamlit app does not run as a Worker as it stands

## License

Released under the [MIT License](LICENSE).
