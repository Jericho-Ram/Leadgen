# Leadgen

A scenario-driven B2B lead scoring pipeline. It takes a raw company list and returns a ranked, verified, client-deliverable sheet — with a written reason behind every point awarded.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Jericho-Ram/Leadgen/blob/main/leadgen_pipeline_v2.ipynb)

Runs entirely in Google Colab. No local install, no paid APIs, no database.

---

## The problem it solves

Most lead lists are ranked by something nobody can explain. A client asks why a lead scored 82 and the answer is a shrug.

This pipeline scores on a fixed rubric where every point is traceable. The `score_reasons` column reads like this:

```
+30 business email, valid MX | +20 decision-maker title ('chief executive') |
+15 size 120 within ICP band | +15 3 sector signals on site |
+10 Mandarin-market lead (outreach edge) | +10 website reachable
```

That is the whole product. The ranking is secondary to being able to defend it.

---

## Quick start

1. Click the **Open in Colab** badge above.
2. Run Cell 1 (dependencies), then Cell 2 (config), then straight down.
3. It ships with `INPUT_MODE = "sample"`, so it runs immediately on demo data with no setup. The last cell downloads a formatted XLSX.

Once that works, switch `INPUT_MODE` to `"csv_upload"` and point `COLUMN_MAP` at your own columns.

---

## The six stages

| Stage | What it does | Output |
|---|---|---|
| 1. Ingest | Load from Sheets, Drive, or CSV upload | `df_raw` |
| 2. Clean | Normalise, extract root domain, dedupe | `df_clean` |
| 3. Enrich | Fetch site, detect language, match keyword families | `df_enriched` |
| 4. Verify | Email syntax + domain MX records | `df_verified` |
| 5. Score | Transparent 100-point rubric | `df_scored` |
| 6. Export | Multi-tab XLSX, optional Google Sheet | deliverable file |

Each stage checkpoints to Parquet. Colab disconnects mid-scrape cost you minutes, not the whole run.

---

## Four scenarios

Set `SCENARIO` on the first line of Cell 2. The ICP, weights, tier cutoffs, and keyword families all switch together.

| Scenario | You are looking for | Heaviest dimension |
|---|---|---|
| `client` | Companies who will buy from you | Verified email (30) |
| `member` | Organisations whose staff join the gym | Decision-maker + headcount (25 each) |
| `supplier` | Companies you will buy from | Credentials (25) |
| `distributor` | Companies who will resell for you | Size, role, email, contact (20 each) |

**The weights differ because the signals differ, not just the keywords.**

For `client`, a verified email *is* the deliverable, so it carries the most weight. For `supplier` you are the buyer — email is table stakes, and what matters is whether they can actually supply, so ISO / HACCP / halal / BPOM claims carry 25 points. For `member`, headcount drives revenue and a lead outside the service area is worth nothing at all.

### Sector and role are scored separately

A juice manufacturer and its distributor share every sector keyword. Only the role keywords (`manufacturer`, `OEM`, `distributor`, `authorized dealer`, `agen resmi`, `经销商`) tell them apart. Collapsing these into one dimension was the main flaw in v1.

### Hard gates

Some misses are disqualifying, not just low-scoring. Weighting can't express that.

A Taipei company with a verified CEO email and perfect headcount scored 70/100 as a gym lead — while sitting 3,500km from the gym. `member` now applies a hard gate: leads with a confirmed address outside the service area are marked **tier X**, excluded from delivery, and routed to their own "Out of Area" tab.

Gates fire only on positive evidence of a miss. An unknown location doesn't disqualify a lead; it lowers its confidence score.

---

## Scoring dimensions

Nine exist. Each scenario uses a subset — a dimension weighted `0` is skipped entirely and never appears in the reasons.

| Dimension | Measures |
|---|---|
| `email_verified` | Syntax valid, not disposable, domain has MX records |
| `decision_maker` | Title indicates budget authority (full) or influence (half) |
| `company_size_fit` | Headcount inside the scenario's band |
| `sector_match` | What industry they're in, from site text |
| `role_match` | Where they sit in the supply chain |
| `credential_match` | Certifications, capacity, export experience |
| `apac_mandarin_edge` | Mandarin-market lead, or Chinese-language site |
| `local_proximity` | Inside the physical service area |
| `website_live` | Site reachable and readable |

Titles are matched on **word boundaries**, so `ceo` doesn't fire inside "ceonomics" and `intern` doesn't disqualify an "Internal Audit Director". Chinese titles use substring matching, since CJK has no word boundaries. Indonesian and Chinese HR titles resolve correctly: `Kepala HRD`, `Manajer SDM`, `人力资源经理`, `总经理`.

### The confidence column

`lead_score` alone is misleading. Two leads can both score 60 — one hit most of what was checkable, the other was mostly unknowns. `confidence` reports the share of dimensions that had any data at all. Read them together.

---

## What "verified" means here

Stage 4 checks that an email is syntactically valid, not from a disposable provider, and that its domain publishes MX records — meaning the domain can receive mail.

**It does not prove the specific mailbox exists.** That needs SMTP probing, which gets your IP blacklisted, or a paid API such as NeverBounce or ZeroBounce.

Say **"MX valid"** to clients. Not "100% deliverable". Overclaiming here is how lead gen freelancers lose accounts.

---

## Validating the rubric

The weights are a hypothesis. Plausible reasoning is often wrong.

Cell 9 compares your score against a **named baseline: random ordering**. It bootstraps 1,000 shuffles, takes the same top-k from each, and reports whether your top-decile reply rate clears the 95th percentile of random.

It does nothing until you have outcome data. Run a campaign, join the replies back on `email`, then run it.

**Do not put a lift figure in a proposal before this passes.** Run 200 leads, measure, quote the real number.

---

## Known failure modes

| Failure mode | Effect | Mitigation |
|---|---|---|
| JS-rendered sites | Near-zero site text, sector score wrongly 0 | Check low `site_text_chars` rows by hand |
| Cloudflare / bot walls | `http_403`, website points lost | Treat 403 as unknown, not absent |
| Catch-all mail domains | `mx_valid` true, mailbox may not exist | Say "MX valid", never "deliverable" |
| MX timeout | `mx_unknown` scores 0, penalising a good lead | Re-run Cell 7; the cache makes it cheap |
| Parked domains | Site resolves, content is a placeholder | Low text + no keyword hits flags these |
| Sector/role confusion | Manufacturer and distributor share sector words | Role keywords scored separately |
| Self-declared credentials | Site claims ISO 9001, nobody verified it | This measures *claims*. Ask for the certificate |
| Proximity from a city string | "Bandung" may be a billing address | Half points for site-only evidence |
| `member` scope | Scores organisations, not individual walk-ins | Individuals need a different data source |
| Rubric never validated | Confident ranking, no evidence | Cell 9 before quoting any lift |

---

## Scraping responsibly

`RESPECT_ROBOTS` defaults to `True`. Leave it on for anything client-facing — a blacklisted client domain costs far more than the leads you'd gain.

Requests are throttled to one every 2 seconds by default. Free Colab IPs get blocked quickly without this.

For EU or UK contacts, GDPR applies to B2B personal data. Legitimate interest can cover B2B outreach, but you need a lawful basis and a working opt-out. The default `target_countries` is APAC-only, which sidesteps this — widen it deliberately, not by accident.

Check a site's terms before scraping it commercially.

---

## Output

A five-or-six tab XLSX:

- **Summary** — run metadata, tier counts, verification breakdown, active dimensions
- **Scored Leads** — the deliverable, ranked, with reasons
- **Priority (A–B)** — the leads worth contacting first
- **Out of Area** — gate-disqualified leads, `member` scenario only
- **Rejected** — rows with no usable domain or email
- **Raw Source** — the original input, unmodified

Headers are frozen and formatted. Internal scraping fields are stripped from the client-facing tabs.

---

## Configuration

Everything lives in Cell 2. For a new client you edit three things:

```python
SCENARIO = "client"          # client | member | supplier | distributor

BASE = {
    "client_name": "...",
    "campaign_name": "...",
    "INPUT_MODE": "csv_upload",
    "COLUMN_MAP": { ... },   # map your source columns to pipeline names
}
```

Cell 2 asserts on load that every scenario's weights total 100, that tier cutoffs descend, and that no dimension is weighted without the keywords needed to score it. It fails immediately rather than three stages later.

Set `ENRICH_LIMIT = 25` for a test slice before committing to a full run.

---

## Files

```
leadgen_pipeline_v2.ipynb    Current version — four scenarios, hard gates
README.md
```

---

## Roadmap

- Playwright fallback for JS-rendered sites
- Contact-page crawl, not just homepage
- Paid verification API as an optional Stage 4 upgrade
- Outcome logging that feeds Cell 9 automatically

---

Built by [Edo Sanjaya P](https://github.com/Jericho-Ram) — data work with leak-free splits, named baselines, real metrics, and documented failure modes.
