# Tool PriceTracker

Tool PriceTracker collects Harbor Freight product prices and promotional history, stores
observations in SQLite, and generates static HTML reports. Data comes from
Harbor Freight's public product catalog and deal pages, plus archived copies of
those pages in the Internet Archive's Wayback Machine.

This README describes the code as it exists today. The planning document
[`Tool PriceTracker_HF_Price_Tracker_Spec.docx.md`](./ToolPulse_HF_Price_Tracker_Spec.docx.md)
also discusses future ideas that are not implemented here.

## What it does

- Discovers product URLs and item numbers from the Harbor Freight sitemap,
  Wayback CDX, and optionally saved promotional email.
- Scrapes current deals from `go.harborfreight.com`.
- Extracts historical product prices and deal observations from Wayback
  snapshots.
- Optionally queries current product prices through Harbor Freight's GraphQL
  endpoint using Playwright.
- Stores products, price observations, and deals in `data/toolpulse.db`.
- Generates three static pages: a product and price-history viewer, a deals
  explorer, and a current-sales page.
- Optionally runs scheduled collection and publishes the generated pages using
  GitHub Actions. A Cloudflare Worker is available for a separate Wayback
  backfill path and proxy.

## How the pieces fit together

```text
Harbor Freight sitemap ─┐
Wayback CDX ────────────┼─> discover_products.py ─> product URL JSON + products
Saved email links ─────┘

go.harborfreight.com ─────> go_hf_scraper.py ───────────────> deals
Wayback product snapshots ─> wayback_batch.py ──────────────> price history
Wayback deal snapshots ────> wayback_go_hf.py ──────────────> deal history
Harbor Freight GraphQL ────> live_hf_scraper.py (optional) ─> current prices
                                                               │
                                                               v
                                                     data/toolpulse.db
                                                               │
                     ┌─────────────────────────────────────────┼──────────────┐
                     v                                         v              v
             generate_viewer.py                       generate_deals.py  generate_current_sales.py
                     │                                         │              │
                     v                                         v              v
                viewer.html                                  deals.html  current-sales.html
```

The database layer in [`db.py`](./db.py) creates three tables:

- `products`: item number, product metadata, and Harbor Freight URL.
- `price_snapshots`: dated price observations and their source.
- `deals`: deal price, coupon/promotion data, validity dates, and source.

The database is created on demand. A fresh checkout does not include
`data/toolpulse.db`; the database and most generated JSON data are ignored by
Git. GitHub Actions workflows pass the database between runs using Actions
artifacts, so a fork starts with its own empty data and artifact history.

## Quick start: run locally

Python 3.12 is used by the GitHub Actions workflows and is recommended. Python
3.10 or later is needed for the type syntax used in the scripts.

Create and activate a virtual environment, then install the dependencies used
by the scheduled workflows:

```bash
python -m venv .venv
```

On macOS/Linux:

```bash
source .venv/bin/activate
```

On Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the scraper dependencies:

```bash
python -m pip install requests beautifulsoup4 lxml
```

Initialize an empty database, collect current deal-page data, and generate the
HTML reports:

```bash
python db.py
python scrapers/go_hf_scraper.py --detail --db
python generate_viewer.py
python generate_deals.py
python generate_current_sales.py
```

The scraper needs an Internet connection. The generated HTML files are written
to the repository root and can be opened directly in a browser. They use
Chart.js from a CDN, so chart-related functionality requires an Internet
connection when viewing them.

To discover product URLs from the live sitemap and save products to the
database:

```bash
python scrapers/discover_products.py --sitemap-only --db
```

To discover both current and archived catalog URLs, use
`python scrapers/discover_products.py --db`. That also makes a broad Wayback
CDX request and can take longer. Sitemap access may be blocked from some
networks; the script reports when the live sitemap is unavailable.

### Optional collection commands

Run a small product Wayback batch after product discovery:

```bash
python scrapers/wayback_batch.py --db --batch-size 5
```

Repeat the command to process further pending products. Progress and CDX cache
files are written under `data/` and are not tracked by Git.

For the historical `go.harborfreight.com` deal pages:

```bash
python scrapers/wayback_go_hf.py --db --batch-size 5
```

The repository also includes a pre-discovered URL list at
[`data/go_hf_wayback_urls.json`](./data/go_hf_wayback_urls.json) for this
backfill.

To query current product prices, install Playwright and its Chromium browser:

```bash
python -m pip install playwright
python -m playwright install chromium
python scrapers/live_hf_scraper.py --limit 100
```

The live scraper uses a browser session and Harbor Freight's GraphQL endpoint.
It is a separate, optional run; it is **not** included in the daily GitHub
Actions scrape workflow.

The local batch runner can also coordinate these operations:

```bash
python scrapers/local_batch_run.py --limit 25
python scrapers/local_batch_run.py --go-hf --limit 25
python scrapers/local_batch_run.py --live --limit 100
```

Run it without a limit to process the full pending queue. For available modes,
worker counts, and other options, see the usage text at the top of
[`scrapers/local_batch_run.py`](./scrapers/local_batch_run.py).

## Optional Gmail email collection

Email collection is not required for the rest of the project. To use it:

1. Enable IMAP for a Gmail account, create a Gmail label/filter for Harbor
   Freight emails, and create a Google App Password (2-Step Verification is
   required by Google).
2. Create a local `.env` file in the repository root; do not commit it:

   ```dotenv
   GMAIL_ADDRESS=you@example.com
   GMAIL_APP_PASSWORD=your-app-password
   GMAIL_LABEL=HF-Deals
   ```

   The fetcher also accepts these values as environment variables.
3. Fetch and save the emails, then extract deals:

   ```bash
   python scrapers/email_fetcher.py --save-raw
   python scrapers/extract_email_deals.py --db
   ```

Raw email files go in `emails/`; resolved links and extracted data go in
`data/`. Both locations are ignored by Git.

## Running the scheduled workflows on a fork

The workflows are in [`.github/workflows/`](./.github/workflows). To run them
on a fork:

1. Fork the repository and enable GitHub Actions for the fork.
2. Review **Settings → Actions → General → Workflow permissions**. Workflows
   that publish or update artifacts need the repository permissions shown in
   their YAML files, including write access to contents where configured.
3. Run **Discover Products (Sitemap)** manually first to establish a product
   catalog and database artifact. It is also scheduled weekly.
4. Run **Daily Deal Scrape** to collect current `go.harborfreight.com` deals.
   The current workflow runs daily at 13:13 UTC.
5. The product and go.hf Wayback backfills run hourly in parallel slices.
   Each can also be started manually; both accept a batch-size input.
6. To publish pages, enable GitHub Pages for the fork and use the `gh-pages`
   branch as the publishing source. **Deploy Viewer to GitHub Pages** regenerates
   the pages after a backfill workflow completes, or when manually dispatched.

The database and progress files are restored from and uploaded to GitHub Actions
artifacts. Check the workflow retention settings and your fork's artifact
history before relying on them as long-term backups. Generated files in a
workflow run are not automatically committed to the default branch.

The live product price scraper is not scheduled by these workflows. The Gmail
fetcher is also not scheduled and requires private account credentials; do not
add those credentials to source control.

## Optional Cloudflare Worker

The Worker in [`worker/src/index.js`](./worker/src/index.js) is an alternative
Wayback processing service. It stores a queue and results in Cloudflare KV and
processes small batches on a cron trigger. It has HTTP endpoints for seeding
URLs, checking status, reading or exporting results, and proxying Wayback
requests. The Python backfill workflows do not require it.

To run or deploy it:

1. Install Node.js, then run `npm ci` in `worker/`.
2. Create a Cloudflare KV namespace named `TOOLPULSE_KV`.
3. Replace the namespace `id` in [`worker/wrangler.toml`](./worker/wrangler.toml)
   with the ID created for your account.
4. From `worker/`, use `npm run dev` for local development or `npm run deploy`
   to deploy it.

The committed KV namespace ID is configuration from the original deployment;
forks must use a namespace they control. To enable the optional Worker seed step
in the product-discovery workflow, configure the fork's
`TOOLPULSE_WORKER_URL` repository variable.

## Repository map

### Python data pipeline

| File | Purpose |
| --- | --- |
| [`db.py`](./db.py) | SQLite schema and database operations: products, price snapshots, deals, imports, and summary statistics. |
| [`scrapers/discover_products.py`](./scrapers/discover_products.py) | Discovers product URLs from the live sitemap, Wayback CDX, and saved emails; writes URL JSON and optionally updates SQLite. |
| [`scrapers/go_hf_scraper.py`](./scrapers/go_hf_scraper.py) | Scrapes current deal grids and optional coupon detail pages from `go.harborfreight.com`; writes JSON and optionally SQLite. |
| [`scrapers/wayback_backfill.py`](./scrapers/wayback_backfill.py) | Shared Wayback helpers for locating product snapshots, extracting historical prices, and one-off discovery/backfill operations. |
| [`scrapers/wayback_batch.py`](./scrapers/wayback_batch.py) | Resumable, sliced product-price backfill used by GitHub Actions; builds a priority queue and tracks progress/cache files. |
| [`scrapers/wayback_go_hf.py`](./scrapers/wayback_go_hf.py) | Resumable, sliced Wayback backfill for archived deal and coupon pages. |
| [`scrapers/local_batch_run.py`](./scrapers/local_batch_run.py) | Runs product backfill, go.hf backfill, or live price collection locally; archive backfills support a configurable worker count. |
| [`scrapers/live_hf_scraper.py`](./scrapers/live_hf_scraper.py) | Optional Playwright-based current-price scraper using Harbor Freight's GraphQL API. |
| [`scrapers/email_fetcher.py`](./scrapers/email_fetcher.py) | Connects to Gmail over IMAP, parses promotional messages, and optionally saves raw `.eml` files. |
| [`scrapers/extract_email_deals.py`](./scrapers/extract_email_deals.py) | Resolves links from saved emails, scrapes linked deal pages, and optionally imports deals into SQLite. |

### Reports and static data

| File | Purpose |
| --- | --- |
| [`generate_viewer.py`](./generate_viewer.py) | Builds the product table, product price-history charts, deal history, and summary statistics. |
| [`generate_deals.py`](./generate_deals.py) | Builds the deal explorer, including sale/event summaries, filters, and discount comparisons. |
| [`generate_current_sales.py`](./generate_current_sales.py) | Builds the current-sales report with expiration information and buy-signal comparisons. |
| [`viewer.html`](./viewer.html) | Checked-in generated viewer snapshot; regenerate from the current database with `generate_viewer.py`. |
| [`deals.html`](./deals.html) | Checked-in generated deal explorer snapshot; regenerate with `generate_deals.py`. |
| [`current-sales.html`](./current-sales.html) | Checked-in generated current-sales snapshot; regenerate with `generate_current_sales.py`. |
| [`data/go_hf_wayback_urls.json`](./data/go_hf_wayback_urls.json) | Tracked list of discovered archived go.hf URLs used by the historical deal backfill. |
| [`ToolPulse_HF_Price_Tracker_Spec.docx.md`](./ToolPulse_HF_Price_Tracker_Spec.docx.md) | Original project plan and roadmap; some technologies and goals described there are not implemented in the current repository. |

### Automation and Worker

| File | Purpose |
| --- | --- |
| [`.github/workflows/daily-scrape.yml`](./.github/workflows/daily-scrape.yml) | Daily current deal scrape; restores and updates the database artifact. |
| [`.github/workflows/discover-products.yml`](./.github/workflows/discover-products.yml) | Weekly/manual product discovery; stores catalog artifacts and can seed the Worker. |
| [`.github/workflows/wayback-backfill.yml`](./.github/workflows/wayback-backfill.yml) | Hourly/manual product-price backfill in three slices. |
| [`.github/workflows/wayback-go-hf-backfill.yml`](./.github/workflows/wayback-go-hf-backfill.yml) | Hourly/manual go.hf deal backfill in four slices. |
| [`.github/workflows/deploy-viewer.yml`](./.github/workflows/deploy-viewer.yml) | Regenerates all reports from the database artifact and publishes them to `gh-pages`. |
| [`run_daily.sh`](./run_daily.sh) | Bash helper for running the go.hf deal scraper, printing DB stats, and rotating local logs. |
| [`worker/src/index.js`](./worker/src/index.js) | Cloudflare Worker for scheduled Wayback processing, KV-backed state/results, and HTTP endpoints. |
| [`worker/wrangler.toml`](./worker/wrangler.toml) | Wrangler Worker name, entry point, compatibility date, cron, and KV binding. |
| [`worker/package.json`](./worker/package.json) | Worker development and deployment scripts and Wrangler dependency. |
| [`worker/package-lock.json`](./worker/package-lock.json) | Locked Node dependency versions for reproducible Worker installs. |
| [`.gitignore`](./.gitignore) | Excludes local databases, generated data, email files, logs, credentials, and build/runtime files. |

## Fork and customization checklist

- Decide whether to use GitHub Actions, local runs, or both. Each fork has its
  own database/artifact history; it does not inherit the original project's
  collected database.
- Review scraper schedules, batch sizes, and concurrency before enabling
  recurring collection. Respect the source sites' terms, robots guidance, and
  rate limits.
- If deploying the Worker, create your own KV namespace and update its
  configuration as described above.
- If publishing with Pages, confirm the deploy workflow can write the
  `gh-pages` branch and configure Pages to publish from it.
- If using Gmail collection, use a dedicated account/app password and keep
  `.env` and saved emails private.
- Keep generated pages and data artifacts in sync by regenerating reports after
  the database changes.

## Current implementation notes

- SQLite is the database. PostgreSQL/TimescaleDB, an API server, and object
  storage mentioned in the planning spec are not present in this repository.
- The scheduled daily scrape collects deal-page observations; it does not
  perform a daily full-catalog current-price scrape.
- The GitHub Actions backfill workflows rely on artifacts for persistence.
  Running the Python scripts locally instead uses the local ignored files under
  `data/`.
- There is no top-level `requirements.txt`, automated test suite, or license
  file in the current repository. The Python dependencies for standard
  collection are listed in the install command above; Playwright is only
  needed for live price collection.
