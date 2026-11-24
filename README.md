# Follow the Rupee: guide for reviewers and auditors

**Off Campus code:** 9QC5AJ

This page explains what the system concludes, how each number is produced, and how to check any of it yourself.
It is written for judges, auditors and anyone who wants to challenge a result.

> **All data is synthetic.** It is the hackathon dataset (`data/`), not BMC data. Contractors are fictional.
> Nothing here should be used to judge a real person, contractor or department.

---

## 1. What the system answers

| Question | Where | Answer on this dataset |
|---|---|---|
| Where did the money go? | View 1 | ₹13.43 Cr spent of a ₹14.91 Cr monsoon budget. 6 of 24 wards overspent. |
| Which bills need explaining? | View 1, View 2 | **527 bills** flagged, **₹1.71 Cr** at stake. ₹60.3 L of it was paid for surface repairs the original contractor owed under warranty (DLP). |
| Did the fixes last, and who is responsible? | View 3A | 3 contractors are clearly better than expected for their conditions, 2 within the normal range, 3 clearly worse. |
| Where should the next rupee go? | View 3B | With the ₹1.77 Cr wards had left, the recommended plan prevents **89% more harm** than "fix the most-complained roads first" on the same money. |

## 2. Principles you can hold us to

1. **Code computes every number; language models only explain.** Detectors, scores and the budget plan are deterministic code.
   Any text a model writes is generated from fields that already exist, and it cites their IDs.
2. **Every flag points to its evidence.** Each flag carries `evidence_row_ids` (work orders, road, rate-card row or rain date)
   and a plain `rule_text` saying which rule fired and with what values.
3. **Neutral labels.** Flags are labelled *Needs Explanation* or *Recoverable under DLP*. Never "fraud" or "corrupt".
   Every flag accepts a **Right-of-Reply** from the ward engineer or contractor (`flag_replies`), so a flag can be answered and closed.
4. **Fairness before blame.** Contractors are compared with what their working conditions predict, not with each other's raw numbers.
5. **Reproducible.** One command chain rebuilds everything from `data/`. The automated tests check the API end to end (Section 8).

## 3. How it fits together

```
data/*.csv ──> supabase/load_source.sql ──> Postgres (typed schema: supabase/migrations/)
     │
     ├─ pipeline/build.py ──> exports/flags.csv, ward_audit.csv ──> load_audit.sql ──> flags, ward_audit
     │                                         (Views 1, 2, 4: audit engine, standard-library Python)
     │
     └─ view3/view3_pipeline.py ──> v3_* tables                    (View 3: XGBoost + SHAP, MILP optimizer)

backend/api.py (FastAPI)  reads those tables/views  ──>  frontend
```

| Component | File | Role |
|---|---|---|
| Schema, security | `supabase/migrations/*.sql` | Typed tables, read views per screen, row-level security (public read, no anonymous edits) |
| Audit engine | `pipeline/build.py` | Detectors, ward audit, under-reporting |
| Scorecard + allocation | `view3/view3_pipeline.py`, `view3/allocator.py`, `view3/sql/01_features.sql` | 3A and 3B |
| API | `backend/api.py`, `backend/models.py`, `backend/openapi.json` | Endpoints with typed responses |
| Tests | `postman/backend.postman_collection.json` | 37 requests, 94 assertions |

## 4. View 1 and 2: the audit (detectors)

Run over all 2,499 work orders. A bill can trigger several detectors. Totals count each bill once,
using its largest amount at stake (`flagged_queue` view), so rupees are never double-counted.

| Detector | Tier | Bills | Rule | ₹ at stake |
|---|---|---|---|---|
| `DLP_PAID_TWICE` | 1 | 259 | A surface repair (cold mix, hot mix, mastic, paver relay) billed on a road still under its Defect Liability Period | Full bill |
| `OVERLAPPING_WORK_ORDER` | 1 | 47 | Same road and method, with the second job started before the first was finished | Full bill of the later job |
| `REPEAT_BILL_NO_COMPLAINT` | 1 | 45 | Same road and issue billed again within 7 days, with no citizen complaint linked. Every genuine job in this data has at least one | Full bill |
| `RATE_INFLATION` | 2 | 109 | Bill more than 1.5× the rate card. Normal jobs sit within 0.7–1.3×, with a clear break above that | Amount above the 1.3× normal band |
| `RAIN_DAY_WORK` | 2 | 116 | Hot mix or mastic completed on a day with at least 64.5 mm of rain (IMD "heavy"). Cold mix is excluded because it's made for wet weather | Full bill |

Tier 1 means a contractual breach is likely, Tier 2 means the bill is implausible on its face. All thresholds are in one block at the top of
`pipeline/build.py`, and the values used are echoed in `exports/summary.json → meta.rules`.

**Ward figures** (`ward_audit`, `GET /api/wards`):
- `dlp_leakage_rate` = warranty money paid ÷ ward spend. The map is coloured by this.
- `warranty_enforcement_rate` is 0 wherever there are warranty-eligible jobs, because no such repair was ever charged ₹0 to BMC.
  That is itself a finding.
- `under_reported_flag`: physical risk (flood-proneness, flood hotspots, accidents) in the top half, while complaints per
  lakh people are in the bottom quarter. On this data: **P/N**. These wards get extra weight in 3B, so silence isn't read as no need.

**Complaint clean-up:** the 7% of complaints with no ward are matched through locality names and aliases, including Devanagari
(426 resolved, 31 left unresolved). The complaint-to-work-order links are kept in their own table (`complaint_work_orders`).

## 5. View 3A: fair contractor scorecard

**Problem:** raw failure rates punish whoever was sent to the hardest roads in the worst weeks.

**Method:**
1. An XGBoost model estimates each job's chance of **failing within 30 days**, and a second one its chance of **missing the deadline (SLA)**.
   Both use only conditions outside the contractor's control: rain and big tides in the week after the job, rain during it,
   ward flood-proneness, distance to a flood hotspot, job type, road type, surface, road age and traffic.
   **The models never see the contractor.** Each job is scored by a model that didn't train on it (5-fold, out-of-fold predictions).
2. For each contractor: **actual failures vs expected failures**, summed over their jobs.
3. **Small-sample fairness:** every contractor starts with 10 imaginary jobs that went exactly as expected, so a handful of jobs can't
   produce an extreme score. A 90% confidence range is computed, and the verdict is *worse than expected* or *better than expected* only
   when the whole range sits on one side.
4. Score 0–100, where **50 = exactly as expected for those conditions**. The overall score is 60% durability + 40% speed.
5. SHAP compares each contractor's jobs with the city average to explain *why* their conditions were harder or easier (`drivers` field).

| Contractor | Jobs | Failed in 30 days | Expected | Durability | Verdict | Overall |
|---|---|---|---|---|---|---|
| Deccan Paving Solutions | 287 | 9.1% | 22.7% | 76.0 | better than expected | 69.3 |
| Sahyadri Infra Works | 337 | 13.6% | 23.8% | 69.0 | better than expected | 65.6 |
| Mithi Infra Projects | 319 | 11.6% | 24.0% | 72.9 | better than expected | 64.9 |
| Shivneri Construction | 289 | 26.3% | 23.2% | 44.1 | within the normal range | 46.3 |
| Konkan Roadcon Pvt Ltd | 337 | 26.7% | 23.6% | 44.2 | within the normal range | 45.9 |
| Arabian Sea Engineering | 328 | 32.3% | 24.9% | 36.6 | worse than expected | 38.7 |
| Western Coast Asphalt Co | 299 | 38.8% | 24.0% | 23.0 | worse than expected | 33.3 |
| Mahalaxmi Civil Contractors | 303 | 37.6% | 23.9% | 24.8 | worse than expected | 29.8 |

**What the models found, stated plainly:** conditions predict missed deadlines well (AUC 0.79), but barely predict failures (AUC 0.52).
So the deadline score needed the adjustment, while the differences in how long repairs last are mostly down to the contractors themselves.
**Design choice:** the choice of repair method is *not* adjusted for (`ADJUST_FOR_METHOD = False`). Laying cold mix in the monsoon is a decision the contractor made.
Billing is reported alongside the score (median bill vs rate card, jobs above 1.3×) but is **not part of the score**. Billing belongs to the audit.

## 6. View 3B: next-rupee budget allocation

**Problem:** with limited money, which repairs should be funded, and with which method, to prevent the most harm?

**Method:** an exact optimization (a multiple-choice knapsack, solved with SciPy/HiGHS), not a ranking heuristic.
- **Items:** one per road segment × issue type seen this season. Demand = repairs needed this season + open complaints.
- **Harm per occurrence** = severity × stakes.
  - **Severity** comes from BMC's own SLA class: 168 ÷ SLA hours, so cave-ins and open manholes count 7× a broken paver.
  - **Stakes** add weight for a flood hotspot within 800 m, an emergency-hospital ambulance road, schools, traffic,
    a BEST route, road-caused accidents, open complaints and an under-reported ward.
- **Per repair method:** harm avoided = demand × harm × (1 − the method's observed failure rate).
  Cost = demand × rate-card cost × (1 + failure rate), so the cost of redoing failed fixes is included.
- **Constraints:** one method per item; stay within the budget; every ward receives at least 50% of its proportional share if it has the demand.
- **Warranty items** (surface defects on roads under DLP) go to Bucket A at **₹0 to BMC**: the original contractor owes the fix.
  On this data that's 78 items, about ₹75.7 L at rate card.

**Result, harm avoided on the same money (₹1.77 Cr, what wards had left in 2025):**

| Approach | Harm avoided | Repairs funded |
|---|---|---|
| **Optimizer** | **4,694** | 196 |
| Most-complained roads first, cheapest patch | 2,489 | 16 |
| Highest-stakes roads first, cheapest patch | 2,267 | 97 |

The gain is largest when money is tight and shrinks as the budget approaches the ₹14.25 Cr needed to fix everything.
The full curve is in `GET /api/v3/summary`. Method choice comes out of the maths, not a hard-coded rule: durable methods on high-stakes roads, cheap patches on quiet ones.
Every item states why it was funded, not funded, or is free under warranty (`explanation`, `reason_codes`, `evidence_wo_ids`).

## 7. How to check a number yourself

With the API running (Section 8), or by running the same SQL against the database:

```bash
# A flag and the rows behind it
curl -s localhost:8000/api/flags/DLP_PAID_TWICE:WO25-01994
# The bill, its road and its warranty status
curl -s localhost:8000/api/roads/RD0058
# The connected trail for a road: complaint -> work order -> reopen -> accident
curl -s localhost:8000/api/roads/RD0058/timeline
```

```sql
-- Recompute RATE_INFLATION for one bill from first principles
select wo.work_order_id, wo.cost_inr, rc.standard_rate_inr,
       wo.cost_inr / (rc.standard_rate_inr * case when rc.unit = 'per sqm' then wo.area_sqm else 1 end) as ratio
from work_orders wo join rate_card rc using (repair_method)
where wo.work_order_id = 'WO25-01626';   -- ratio > 1.5 => RATE_INFLATION

-- City totals: each bill counted once
select count(*), sum(rupees_at_stake) from flagged_queue;
```

For a contractor score, `v3_job_expected_risk` holds the expected failure probability of every job, so actual vs expected can be re-added by hand.

## 8. Reproduce everything

```bash
pip install -r view3/requirements.txt                 # Python 3.12/3.13
cp .env.example .env                                  # fill in DATABASE_URL (Supabase Session pooler or local Postgres)
python3 pipeline/build.py
for f in supabase/migrations/*.sql; do psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f "$f"; done
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f supabase/load_source.sql
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f supabase/load_audit.sql
python view3/view3_pipeline.py
cd backend && STRICT_MODELS=1 uvicorn api:app --port 8000          # docs: http://127.0.0.1:8000/docs
newman run ../postman/backend.postman_collection.json              # 37 requests, 94 assertions
```

With `STRICT_MODELS=1`, any response field that isn't declared in `backend/models.py` is an error rather than being silently dropped.
Results are deterministic: models use a fixed seed, and the optimizer is exact.

## 9. Judgment calls and limitations

We state these openly so they can be challenged:

- **Thresholds are choices.** These include 1.5× the rate card, 7 days for a repeat bill, 64.5 mm for heavy rain, the stakes weights in 3B and the 50% equity floor.
  Each is set in one place and explained in a code comment.
- **"Harm points" are a relative unit**, used to compare options, not rupees or lives.
- **DLP scope:** only road-surface repairs count as owed by the original contractor. Drainage, pumping, manholes and cave-ins are utility work and stay with BMC.
  This is deliberately conservative, so the warranty figure (₹60.3 L) understates what a broader reading would give.
- **Demand in 3B comes from this season's history.** We tested a forecasting model and it did no better than "same as last time", so we didn't use it.
- **Complaints without GPS** are placed by locality name only (31 couldn't be resolved). A wrong placement affects only the open-complaint weight.
- **Not built yet:** the two Claude endpoints for drafting a defect notice (View 2) and answering free-text citizen queries (View 4).
  The deterministic lookup they would build on (`/api/lookup`) exists and is tested.
- **A flag is a reason to ask, not a verdict.** The Right-of-Reply exists because some flags will have good explanations (for example, utility trenching).

---

📺 **Video walkthrough:** [https://youtu.be/xVHGD5RvDek](https://youtu.be/xVHGD5RvDek)
