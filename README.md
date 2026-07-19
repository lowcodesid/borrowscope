# BorrowScope

**Estimated Borrowing Power Calculator — Australian Market**

A single-file, client-side web application that estimates residential borrowing power using deterministic, APRA-aligned serviceability logic. Designed for Australian borrowers who want a practical estimate before speaking with a broker or lender.

---

## Changelog

**Unreleased — realism & correctness pass**
- Employment type (per applicant) now applies a real shading factor to base salary (editable in Settings), instead of being collected and ignored.
- Living Situation → Renting now surfaces a Current Rent field (with frequency) and a "continues after settlement" toggle; only adds to expenses when checked.
- Loan Purpose → Investor now surfaces an Expected Rental Income field for the subject property, shaded like other rental income and added to assessable income.
- Removed the non-functional "Use benchmark expenses" toggle — the higher of actual/benchmark is now unconditionally applied (matches the toggle's own subtitle, which was already describing the actual behaviour).
- HELP Balance field relabelled as reference-only — compulsory repayment was always income-tested, not balance-tested; the field never affected the result and now says so.
- Added a frequency selector (weekly/fortnightly/monthly/annually) to rent, BNPL, personal loan, car loan, existing mortgage, other commitments, and insurance — all normalized to monthly via a shared `toMonthly()` helper.
- All numeric inputs now clamp to a non-negative floor via a shared `num()` helper — negative liability entries can no longer inflate borrowing power.
- Primary DTI metric and Position classification now use `targetDti` (loan actually being sought) instead of max-borrowing DTI, for consistency with LVR/surplus/Position, which were already target-loan-based. Max-borrowing DTI is retained as a secondary figure.
- AI Copilot now validates returned field names against a whitelist before applying changes, and surfaces any rejected fields instead of silently dropping them.
- AI model updated from a stale `claude-sonnet-4-20250514` reference to `claude-haiku-4-5-20251001`, appropriate for the short explanation/classification tasks this tool uses AI for.

---

## Overview

BorrowScope separates its calculation engine from its AI layer by design. The core serviceability math is fully deterministic and rules-based — no AI model influences the numbers. AI is used only for plain-English explanation, improvement recommendations, and natural-language scenario interpretation.

---

## Live Demo

Deploy instantly with no build step:

1. Rename `BorrowScope_MVP.html` → `index.html`
2. Drag and drop onto [netlify.com/drop](https://netlify.com/drop)
3. Live at `https://your-app.netlify.app` within seconds

---

## Features

### Applicant & Income
- Single or joint application
- Employment type per applicant (full-time, part-time, casual, contractor, self-employed) — applied as a shading factor on base salary, editable in Settings
- Lender-style income shading: base salary 100%, bonus/commission 80%, overtime 80%, rental income 75%, other income 100%
- HECS/HELP debt toggle with income-tested compulsory repayment deduction
- All shading percentages editable in Settings

### Expenses
- Actual monthly living expenses
- HEM (Household Expenditure Measure) benchmark by household composition — always applies the higher of actual or benchmark
- Childcare costs, private school fees, insurance and recurring commitments
- HEM benchmarks editable by household size in Settings

### Liabilities
- Credit card limits (assessed against full limit, not balance)
- Personal loan, car loan, existing mortgage monthly repayments
- Buy now pay later monthly commitments
- Credit card monthly assessment rate editable in Settings

### Loan Parameters
- Property value and deposit (used for LVR)
- Loan term (5–30 years)
- Actual interest rate, assessment buffer, floor rate
- Assessment rate = `max(actual rate + buffer, floor rate)` — APRA-aligned minimum 3% buffer
- Owner occupier or investor purpose
- Principal & interest or interest only repayment type

### Results Panel
- Estimated maximum borrowing power (binary search, 60 iterations, accurate to ~$1)
- Monthly repayment at actual rate (on target loan)
- Assessed repayment at assessment rate (on target loan)
- LVR = target loan / property value
- DTI = maximum borrowing power / assessable annual income
- Net monthly income after tax and HECS
- Monthly surplus at target loan — green if positive, red if negative
- Target loan feasibility banner — confirms whether the target property is within borrowing power or shows shortfall
- Position indicator: Stronger / Moderate / Constrained
- Income allocation bar: surplus / new loan / expenses / liabilities
- Top 3 factors helping and reducing borrowing power

### Scenario Comparison
- Save up to 2 named scenarios alongside the live base case
- Side-by-side comparison of borrowing power, LVR, DTI, surplus, repayments
- Scenario delta vs base case
- Chart.js bar chart comparing scenarios across key metrics
- Load saved scenario inputs back into the calculator

### AI Layer (requires Anthropic API key)
- **AI Explanation** — plain-English summary of the result and its key drivers
- **Recommendations** — 3–4 specific improvement suggestions based on constraint factors
- **Scenario Copilot** — natural-language what-if questions ("What if I pay off my car loan?") translated to input changes, recalculated, and explained

---

## Calculation Engine

All core calculations are deterministic JavaScript functions with no external dependencies.

### Tax Calculation
Australian resident income tax using FY2024–25 brackets:

| Taxable Income | Rate |
|---|---|
| $0 – $18,200 | Nil |
| $18,201 – $45,000 | 19% |
| $45,001 – $120,000 | 32.5% |
| $120,001 – $180,000 | 37% |
| $180,001+ | 45% |

Plus Low Income Tax Offset (LITO) and Medicare Levy (2%).

### HECS/HELP Repayment
18-tier income-tested repayment schedule (1%–10% of income depending on threshold). Deducted from net monthly income before serviceability assessment.

### Income Shading
```
Assessable income = base × 100%
                  + bonus/commission × 80%
                  + overtime × 80%
                  + rental income × 75%
                  + other income × 100%
```

### Benchmark Expenses (HEM)
Higher of actual declared expenses or HEM benchmark is always applied.

| Household | 0 dep | 1 dep | 2 dep | 3+ dep |
|---|---|---|---|---|
| Single | $2,000 | $2,700 | $3,200 | $3,700 |
| Couple | $3,200 | $3,900 | $4,400 | $4,900 |

All values editable in Settings.

### Credit Card Assessment
```
Monthly CC commitment = total CC limits × 3.8%
```
Full limit used regardless of current balance.

### Assessment Rate
```
Assessment rate = max(actual rate + buffer, floor rate)
Default: max(actual + 3.0%, 9.0%)
```

### Borrowing Power Search
Binary search over [0, $10,000,000] with 60 iterations. Finds the maximum loan where:
```
net monthly income
  − applied expenses
  − monthly liabilities
  − assessed repayment (at assessment rate)
  ≥ minimum surplus buffer ($200 default)
```

### Position Classification
Based on the target loan (not max borrowing):
- **Stronger** — target loan feasible AND monthly surplus ≥ $800 AND max DTI ≤ 6×
- **Moderate** — target loan feasible but surplus < $800, or DTI > 6×
- **Constrained** — target loan not feasible, or surplus < $200

---

## Editable Assumptions

All assumptions are accessible via the Settings modal (⚙ button in header):

| Assumption | Default |
|---|---|
| Base salary shading | 100% |
| Bonus/commission shading | 80% |
| Overtime shading | 80% |
| Rental income shading | 75% |
| Other income shading | 100% |
| CC monthly assessment rate | 3.8% |
| Minimum monthly surplus buffer | $200 |
| HEM benchmarks (8 values) | See above |

---

## AI Integration

AI features require an Anthropic API key. Add it via Settings ⚙ → AI API Key field.

The AI key is held only in the browser session and never persisted or transmitted anywhere other than the Anthropic API directly from your browser.

**For public deployment:** move the API key to a serverless backend proxy (e.g. Netlify Function, Cloudflare Worker) so it is not exposed client-side.

### Model
All AI calls use `claude-haiku-4-5-20251001` with `max_tokens: 800`.

### AI Guardrails
- AI never performs or overrides core calculations
- AI never implies loan approval or specific lender outcomes
- All AI output is labelled as explanatory guidance only
- The calculation engine is always the source of truth

---

## Deployment

### Netlify Drop (recommended, 2 minutes)
```
1. Rename file to index.html
2. Go to netlify.com/drop
3. Drag and drop the file
4. Share the generated URL
```

### GitHub Pages
```bash
# Create a repo, push index.html, enable Pages in Settings → Pages → main branch
```

### Cloudflare Pages
```
Connect GitHub repo or drag-drop via dashboard → deploys to .pages.dev
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Vanilla HTML/CSS/JavaScript (single file) |
| Charts | Chart.js 4.4.1 (CDN) |
| Fonts | Syne 800 (logo), Inter (body), DM Mono (numbers) |
| AI | Anthropic Claude API (`/v1/messages`) |
| Hosting | Any static host (Netlify, GitHub Pages, Cloudflare Pages) |
| Build | None — open the file |

---

## File Structure

```
BorrowScope_MVP.html          # Entire application — CSS, HTML, JS in one file
README.md                     # This file
BorrowScope_Icon_1024.png     # iOS app icon (1024×1024, RGBA, rounded rect masked)
```

---

## Compliance & Disclaimers

BorrowScope is an **estimate only**. It does not:
- Represent any specific lender's credit policy
- Constitute financial advice
- Guarantee or imply credit approval
- Replace a qualified mortgage broker or financial adviser

Actual borrowing power varies by lender, credit profile, employment documentation, property type and location, and applicable credit policy at the time of application.

Always recommend users consult a licensed mortgage broker or financial adviser for advice specific to their circumstances.

---

## Roadmap (Future Phases)

**Deferred from the realism pass above (by explicit scope decision, not oversight):**
- [ ] State selector + per-state HEM/stamp duty
- [ ] LMI (Lenders Mortgage Insurance) premium estimate
- [ ] Negative gearing addback for investment purchases
- [ ] Fixed/variable rate distinction, offset account modelling
- [ ] Guarantor / family equity LVR path
- [ ] Bridging / simultaneous-loan scenario
- [ ] Move AI API key to a serverless proxy (currently bring-your-own-key, client-side only — see Compliance & Disclaimers)

**Other:**
- [ ] Serverless API proxy for safe public deployment
- [ ] Document upload / income verification (OCR)
- [ ] Lender matching and policy comparison
- [ ] Broker handoff / lead capture
- [ ] User accounts and saved sessions
- [ ] Open banking income verification
- [ ] Stamp duty and upfront cost calculator
- [ ] Repayment schedule amortisation table

---

## Licence

Private / proprietary. Not licensed for redistribution or commercial use without permission.
