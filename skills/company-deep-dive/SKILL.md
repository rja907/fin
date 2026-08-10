---
name: company-deep-dive
description: Create a detailed MBI-style company deep dive with history, KPIs, financials, management, and valuation as a single self-contained HTML file.
---

# Company Deep Dive

Create a detailed, MBI Deep Dives-style equity research deep dive for any company. The output must be a single self-contained HTML file (inline CSS only, no external JS/CSS) that covers history, business model, key performance indicators, financials, competitive dynamics, management and incentives, and valuation.

## When to Use

Use this skill when the user asks for a deep dive, company analysis, equity research report, or valuation on a specific public company (e.g., "deep dive on Alphabet", "analyze GOOGL", "create a deep dive on Copart"). Also use when the user provides a ticker symbol ($GOOGL, GOOGL, CSU.TO) and requests detailed analysis.

## Core Workflow

Follow these steps in order. Do not skip steps. Adapt depth to data availability but always produce the full HTML.

### 1. Identify the Company

- Parse company name and ticker. Normalize ticker (strip $, uppercase). Handle dual-class (GOOGL/GOOG), foreign tickers (CSU.TO).
- Resolve CIK for SEC EDGAR: use `https://www.sec.gov/files/company_tickers.json` or `https://data.sec.gov/submissions/CIK{cik}.json`. If not found (non-US), note it and use SEDAR / local filings + Yahoo Finance / KoyFin as fallback.
- Confirm legal entity name (e.g., Alphabet Inc.), exchange, fiscal year end, and reporting currency.

### 2. Classify Business Archetype

Classify before choosing KPIs. Most companies are hybrids — pick primary + secondary:

| Archetype | Examples | Key Questions |
|---|---|---|
| **SaaS / Cloud Infra** | Datadog, Snowflake | NRR, gross margin, Rule of 40, CAC payback |
| **Transactional Marketplace (2/3-sided)** | DoorDash, Booking, Uber, Copart | GMV/GOV, take rate, orders/users, unit economics per order, liquidity |
| **Advertising / Attention** | Alphabet, Meta, Roblox | MAU/DAU, ARPU, ad load, engagement hours |
| **E-commerce / Retail** | Amazon retail | GMV, 1P vs 3P mix, take rate, shipping/fulfillment cost |
| **Vertical Market Software Serial Acquirer** | Constellation Software | Organic vs acquired growth, ROIC, hurdle rate, capital allocation system |
| **Subscription / Consumer Platform** | Spotify, Netflix | Subscribers, churn, ARPU, content cost |
| **Asset-Heavy / Operations** | Copart, airlines | Utilization, capacity, cost per unit, moats from physical assets |

Record the classification at the top of the HTML (e.g., "Archetype: Advertising + Cloud Infrastructure (hybrid) — Alphabet").

### 3. Gather Financials — SEC First, Then Supplements

**SEC is the source of truth.** Do not rely solely on model knowledge.

- **10-K (Item 1 Business, Item 1A Risk Factors, Item 7 MD&A, Item 8 Financials) and 10-Qs**: Pull last 3–5 years of revenue, segment revenue (e.g., Google Services vs Google Cloud vs Other Bets for Alphabet), operating income, margins, SBC, capex, FCF, share count. Use XBRL facts where possible: `https://data.sec.gov/api/xbrl/companyfacts/CIK{cik}.json` (needs `User-Agent` header).
- **Proxy DEF 14A**: Management history, compensation structure, incentives, insider ownership. Note founder-led vs professional CEO, tenure, capital allocation track record.
- **S-1 / Historical filings**: For origin story if company IPO'd recently.
- **Earnings calls / shareholder letters** (Alphabet: Sundar Pichai / Ruth Porat letters, 10-K letter): For narrative on strategy.
- **Supplement with**: Yahoo Finance / KoyFin for price history and market cap, but always cross-check revenue against 10-K. Cite sources inline (e.g., "Source: Alphabet 2024 10-K, p. 28, Company Filings").

If SEC fetch fails (rate limit/CORS), state the attempt, use the most recent 10-K numbers you can verify via web search, and flag as "as of [date] per filing" — never invent numbers. Every financial table must have a source line.

**Currency handling — detect, don't assume:**
- On every filing, explicitly extract functional currency, presentation currency, and any convenience-translation disclosure (e.g., "₸525.11 per $1 as of Dec 31, 2024"). Grep the filing for `functional currency`, `presentation currency`, `convenience translation`, and XBRL `unitRef="U_KZT"` / `iso4217:KZT`.
- **If the filing reports in USD (functional = USD and all XBRL units are `USD` / `iso4217:USD`), do NOT convert.** Display numbers as filed, cite "Reported in USD — no conversion."
- **If the filing reports in a non-USD currency (KZT, CAD, TRY, etc.), trigger full translation workflow:** Income statement & cash flows → period-average rate for that fiscal year (source: central bank / NBRK / Bank of Canada / ECB / exchange-rates.org — cite year + rate). Balance sheet snapshots → period-end spot. If a filing provides a convenience spot, show a 1-row reconciliation: `Local (as filed) ÷ avg rate = $ (this report) vs company convenience $ (spot)`.
- Never silently display raw local-currency numbers as USD. Never divide a multi-year history by today's spot — use year-specific averages. If a rate is unavailable, state the gap and use the filing's convenience spot with label "spot-converted."

**Fiscal-year dating — use today's date:**
- Resolve the *actual* current date first (`date` or system clock) — e.g., today is Aug 9, 2026. Compare to the company's fiscal year end (FYE). A fiscal year is **Actual (A)** if its period has ended *and* the filing / earnings release for that period exists (20-F / 10-K / 6-K). It is **Estimated (E)** only for current or future periods.
- Example: FYE Dec 31, today Aug 9, 2026 → FY2025 (ended Dec 31, 2025, 20-F filed Mar 2026) is **2025A**; FY2026 is 2026E (partial, H1 actual + H2 estimate). Never label a completed year as "E" because the model template defaults to next year.
- In HTML, suffix every column header explicitly: `2024A`, `2025A`, `2026E`, `2027E`... and in valuation, split historical actuals from forward estimates with a visual separator. Validate: before rendering, check `today >= FYE + ~75 days filing lag` — if FY has been filed, it must be "A".

### 4. Research History, Business Model, and KPIs

Cover in this order, adapting to archetype:

- **History (narrative, 400–700 words)**: Founding, near-death/bubble moments, pivots. For Alphabet: Stanford 1998, IPO 2004, restructuring to Alphabet 2015, Sundar CEO 2015/2019. Explain *why* history matters for culture/moat.
- **How the business actually works**: For each segment, explain the plumbing. Alphabet: Search auction, YouTube, Cloud, Network. Draw a simple diagram using HTML/CSS (no images needed) if helpful.
- **KPIs — the heart of the deep dive**: Select 3–6 archetype-specific KPIs and show 3–5 year trends in tables. Do not list generic metrics — explain why each KPI matters.
  - SaaS → NRR/GRR, customers >$100k, DBNR, Rule of 40
  - Marketplace → GOV/GMV, take rate, orders, frequency, contribution margin per order
  - Ads → Search TAC, YouTube ad revenue growth, Cloud backlog, YouTube hours
  - E-comm → GMV, 3P mix, shipping cost / GMV
  - VMS → # acquisitions/year, maintenance revenue %, ROIC
  - General → SBC as % of revenue, dilution, FCF conversion
- **Unit economics**: Build a per-unit bridge (e.g., per order for DoorDash, per search for Google) if applicable. Show take rate decomposition.

### 5. Competitive Dynamics, Moats, and Risks

- Identify 3–5 moat sources: network effects, scale, data/AI, brand, switching costs, cost advantage, culture. Explain durability.
- Map competition (explicit table): For Alphabet → Search vs Bing/ChatGPT/Perplexity, YouTube vs TikTok/Reels, Cloud vs AWS/Azure.
- Cover 2–4 key risks that could break the thesis: For Alphabet → AI disintermediation of Search, regulatory/antitrust, Cloud margin pressure, Other Bets drag.
- Include a "What would I need to believe to NOT own it?" counter-narrative.

### 6. Management, Incentives, and Capital Allocation

- CEO/founder background, tenure, prior track record.
- Compensation: salary vs RSU/PSU, performance metrics, alignment (use DEF 14A numbers).
- Insider ownership % and buying/selling.
- Capital allocation history: buybacks (Alphabet authorized $70B in 2024, ~$62B executed), M&A (Mandiant $5.4B, Fitbit), dividends (initiated 2024), Other Bets funding.
- Culture notes: e.g., Constellation's decentralized VMS, Copart's owner-operator, Booking's Google-SEM mastery.
- **Guidance track record**: Search the last 2–3 years of earnings releases and transcripts for company guidance vs actual outturn. Build a small table: Year | Guidance (as given at start of year) | Actual | Met / Miss / Beat | Source. Example for Kaspi: 2024 net income +25% guided → 25% actual (met, source Mar 2 2026 4Q FY2025); 2025 started at +20% ex-Türkiye → cut to +15% in Q1 → ended at +10-12% Kazakhstan / 10% consolidated ex-underlying (miss on smartphones + tax/reserve headwinds). State hit rate and what it says about management credibility.

### 7. Valuation — Model and Implied Expectations

This is not optional. Produce:

- **Revenue build**: Segment-level assumptions for 5–6 years (base case). Explain each assumption in 1 sentence (e.g., "Search grows at MSD as query growth offsets AI headwinds").
- **Cost structure**: Break out TAC, R&D, S&M, G&A, SBC separately. Show operating leverage thesis.
- **DCF or reverse DCF**: Show at least one of: (a) DCF with WACC/discount, terminal growth, or (b) reverse DCF: "At $190/share ($2.2T EV), market implies X% CAGR / Y% terminal FCF margin — is that reasonable?" Use a sensitivity table (e.g., 2×2 grid: terminal margin vs discount).
- **Multiples context**: EV/Sales, EV/EBIT, P/E vs peers and vs own history (use KoyFin/Yahoo as of date).
- **State your stance**: "I think earnings power is under/over-estimated because…" and list 2–3 model sensitivities the reader should play with.
- Include a downloadable assumptions summary in the HTML table — reader should be able to replicate.
- **Company Guidance vs Model**: Always search filings and earnings releases for the most recent company guidance (revenue, EBITDA, net income, GMV/TPV, or segment guidance). If guidance exists, render a dedicated comparison table `Company Guidance vs Model` in the Valuation section with: (a) metric, (b) company guidance (exact wording + KZT & USD + source link), (c) your model assumption, (d) delta. Cite the press release / 6-K / 20-F with a clickable link (e.g., `Source: Kaspi 4Q FY2025 Mar 2, 2026 — Adj. EBITDA +5% incl. Türkiye [GlobeNewswire]`). If no formal guidance exists, state "No formal guidance issued — model is management-commentary based" and cite the latest call.

Be explicit that this is not investment advice; includeDisclosure: "I may own shares / This is not a recommendation" line in the style of MBI.

### 8. Render Single-File HTML

Requirements:

- **One file only**: `deep-dives/<TICKER>_deep-dive.html` (e.g., `deep-dives/GOOGL_deep-dive.html`) or path user requests. Inline all CSS in `<style>`; no `<link>` to external CSS/JS, no CDN, no fonts that require network.
- **Inline CSS only**: Use a clean, MBI-inspired palette (dark header, serif headings, sans body, muted grays). All styles must be inside `<style>` in `<head>`.
- **No external JS**: If interactivity needed (TOC, collapsible), use pure CSS or minimal inline `<script>` — but prefer no JS at all. Charts must be HTML tables or inline SVG, not Chart.js.
- **Structure**:
  1. Header: Title ("Alphabet: The Attention and Compute Compound"), ticker, market cap as of date, archetype badge, author/date, disclosure.
  2. Sticky or top TOC with anchor links to each section (use `id` anchors).
  3. Sections in order: History → Business Model → KPIs & Unit Economics → Competitive Dynamics & Moats → Management & Incentives → Valuation & Model → Final Words (thesis summary + what to watch).
  4. Every table/chart must have `Source: ...` caption below it.
  5. Footer with data-as-of date and filing links (EDGAR URLs).
- **Quality bar**: Aim for 8,000–12,000 words equivalent (the PDFs average ~10k words, 35–53 pages). Dense but readable. Use tables, not walls of text, for numbers.
- **Print-friendly**: Use `@media print` to hide TOC sticky if needed.
- **Validation**: After writing, open the HTML locally (or via `python -m http.server`) to ensure it renders without network.
- **Auto-open**: After validation, automatically open the generated HTML in Google Chrome: `open -a "Google Chrome" "deep-dives/<TICKER>_deep-dive.html"` (macOS) or `google-chrome` / `xdg-open` on Linux. Do this as the final step so the user can immediately review the result.

## Business-Archetype KPI Quick Reference

Include this checklist when drafting KPIs — do not copy verbatim, use to ensure coverage:

- **All companies**: Revenue CAGR (3y/5y), gross margin trend, EBIT/FCF margin, SBC %, share count change, ROIC, net cash/debt.
- **If Ads**: TAC, ad revenue per MAU, Cloud backlog/RPO, YouTube hours or engagement proxy.
- **If Marketplace**: GOV/GMV, orders, take rate (revenue/GOV), AOV, contribution per order.
- **If SaaS**: ARR, NRR, customers $100k+, cac payback, Rule of 40.
- **If Serial Acquirer**: Organic growth, maintenance vs license, deals/year, price paid/EBIT.
- **If Platform/Gaming**: DAU/MAU, hours engaged, bookings, ABPDAU, DevEx payouts.

## Handling Different Business Types in Practice

When the deep dive subject could be analyzed as multiple archetypes, do not force one. For example:
- Alphabet is Ads (Search/YouTube) + Cloud (SaaS-like consumption) + Other Bets (venture). Model each segment separately.
- Amazon is E-comm + Cloud + Ads — same principle.
- DoorDash is 3-sided marketplace → lead with GOV/take rate, not NRR.

State the segmentation explicitly and weight the thesis by segment contribution (e.g., "Search is 57% of Alphabet revenue and ~75% of profit — the thesis lives or dies there").

## Data Integrity Rules

- Financials from 10-K/10-Q XBRL facts or the filing HTML itself beat any secondary source. If a web fetch fails, say so and show the last verified filing date.
- For valuation, always show base assumptions in a table; never hide them in prose.
- Cite every data table: "Source: Company Filings, MBI Deep Dives, Daloopa, KoyFin" style — even if you constructed it.
- Date everything: "As of Aug 8, 2026, market cap $2.1T at $175/share (KoyFin)".
- Currency: detect before converting — USD filers need no conversion; non-USD filers need year-specific avg (IS) vs spot (BS), reconciled to any convenience spot.
- Dating: suffix all financial columns `A` (actual) vs `E` (estimate) based on today's date vs FYE + filing lag. A completed fiscal year is never "E".

## Anti-Patterns

- Do not produce a 500-word summary and call it a deep dive.
- Do not skip valuation or say "valuation is left as an exercise."
- Do not load external fonts, styles, or scripts that break offline reading.
- Do not hallucinate filing numbers — if you cannot fetch SEC, explain the gap and use the nearest verifiable filing snapshot.
- Do not add extra files — one HTML only.

## Example Invocation

User: "Create a deep dive on Alphabet ($GOOGL)"
Agent: Runs steps 1–8, fetches Alphabet CIK 1652044, pulls 2024 10-K, builds segment model (Services/Cloud/Other Bets), writes `deep-dives/GOOGL_deep-dive.html`, validates it renders, and summarizes thesis in chat.

## References

The skill was designed by studying 7 MBI Deep Dives in `/Users/raj/Downloads/deep-dives`: Amazon 2025 Update (model-heavy), Booking (OTA marketplace), Constellation Software (serial acquirer/VMS), Copart (auction marketplace, owner-operator), Datadog (SaaS observability), DoorDash (3-sided local commerce), Roblox (attention/gaming platform). The common structure across them is: narrative history → business model & unit economics → KPIs → moats/competition → management/incentives → valuation/model → final words. Mirror that structure while adapting KPIs to the company's archetype.
