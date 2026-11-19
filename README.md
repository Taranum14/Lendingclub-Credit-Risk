# Credit Risk Modeling on LendingClub Loans

**Default prediction, approval-policy testing, and what happens when the model drifts.**

An end-to-end analysis of 1.3M LendingClub loans (SQL, Python, Power BI). I built a default-risk model, tested whether it earns more than grade-only underwriting on a held-out year, and checked how reliable its predictions and loss estimates are when loan conditions change.

<img width="1248" height="740" alt="image" src="https://github.com/user-attachments/assets/cdac4d75-4ae2-4758-872f-ecc3e80596c8" /> <img width="1281" height="761" alt="image" src="https://github.com/user-attachments/assets/69488d8d-98f8-4a47-89cc-ca87619ed4d4" /> <img width="1316" height="737" alt="image" src="https://github.com/user-attachments/assets/a725dd50-765f-4064-bf0e-98d54c9c0654" /> <img width="1233" height="741" alt="image" src="https://github.com/user-attachments/assets/5792960c-929b-4243-9a3c-042098a75de2" />





---

## Headline results

| Question | Answer |
|---|---|
| Does the model rank risky borrowers well? | Yes. On 283K held-out 2015 loans, actual default rates rise from 3.8% (safest decile) to 29.2% (riskiest). |
| Does it beat grade-only underwriting? | Slightly. At equal dollars lent, it earned about 1% more profit (+$1.9M to +$2.1M at 60-80% of volume). The edge is statistically borderline and comes from one test year. |
| Is its default level reliable? | Not when it uses the interest rate. It predicted 12.1% defaults for 2015 against an actual 14.9%. |
| What would that cost? | A loss reserve built on it would have been about 18% short. A model without grade and rate came within about 3% (slightly high). |

**The main lesson:** simple rules tested on all loans looked profitable, but the result was biased by unfinished loans. Only a properly built, out-of-time test showed the model's small real edge, and its limits.

---

## Business problem

A lender loses money on defaults, but rejecting too many applicants also loses interest income. This project asks:

1. Can a model rank borrowers better than the lender's own grade?
2. If so, does that translate into more profit, measured fairly?
3. Can the model's probabilities be trusted for setting loss reserves when conditions change?

## Data

- **Source:** [LendingClub accepted loans 2007-2018 (Kaggle)](https://www.kaggle.com/datasets/wordsforthewise/lending-club), about 2.2M loans and 151 columns.
- **Used:** 27 columns, restricted to finished loans (Fully Paid, Charged Off, Default): 1,344,441 loans after cleaning, with a 19.96% default rate.
- **Target:** `Charged Off` or `Default` = 1, `Fully Paid` = 0.
- **Profit definition:** total payments received minus amount funded. It is undiscounted.

## Finding zero: naive analysis was biased

My first SQL analysis on all loans suggested that grades E-G lose money, that 60-month loans are the "hidden problem", and that a simple rule would add about $62M. Checking by loan age showed these results were mostly artifacts of **right-censoring**: the data only includes finished loans, so recent long-term loans that are still being repaid are missing, and the early defaulters are over-represented.

On fully matured loans (2010-2013), every grade was profitable, and 60-month loans earned more per loan than 36-month loans. I rebuilt the analysis on **36-month loans issued 2010-2015**, where every outcome is observed.

## Approach

1. **SQL (SQLite):** default rate, profit and ROI by grade, term, purpose, DTI, income, state and year.
2. **Python:** time-based split with no leakage (outcome columns are never used as features).
   - Fit: 2010-2012 (66,037 loans)
   - Calibrate and choose policy cutoffs: 2013-2014 (262,992 loans)
   - Test: 2015 (283,019 loans)
3. **Models:** grade-only baseline, logistic regression, gradient boosting. A model that predicts return directly, and a model without grade and interest rate, were tested as well.
4. **Policy comparison:** policies compared at equal **dollars lent** (not equal loan counts), with approximate 95% ranges for the profit differences.
5. **Monitoring:** AUC and predicted vs actual defaults by year, PSI on inputs and scores, expected loss versus actual loss.
6. **Power BI:** four-page dashboard.

## Results

### 1. Ranking quality (2015 holdout)

| Model | AUC |
|---|---|
| Grade only | 0.6675 |
| Logistic regression | 0.6667 |
| Gradient boosting | 0.6702 |
| Gradient boosting, without grade and rate | 0.6391 |

The gain over grade is small. Within a single grade, the model ranks defaults with AUC of about 0.56-0.61, which grade alone cannot do.

### 2. Profit at equal dollars lent (2015)

| Dollars lent | Grade-only ROI | Model ROI | Profit difference |
|---|---|---|---|
| 60% | 7.18% | 7.26% | +$1.9M |
| 70% | 7.10% | 7.19% | +$2.2M |
| 80% | 7.00% | 7.07% | +$2.1M |
| 90% | 6.87% | 6.85% | -$0.7M |

At 60% and 80% of dollars lent, the approximate 95% ranges for the profit difference were +$0.3M to +$3.4M and +$0.4M to +$3.8M. These ranges treat loans as independent, so the true uncertainty is wider. A model that predicted return directly did worse than grade-only (ROI 6.79% vs 7.18% at 60%), because it favored high-yield, high-risk loans.

An early comparison at equal loan counts showed a gain about seven times larger. Most of that came from the model funding more dollars, not from picking better loans. Matching on dollars removed it.

### 3. Drift and reliability

| Year | Actual default rate | Model with rate | Model without grade and rate |
|---|---|---|---|
| 2013 | 12.33% | 13.61% | 13.52% |
| 2014 | 13.73% | 13.18% | 14.39% |
| 2015 | 14.89% | 12.12% | 14.40% |

- Ranking stayed stable (AUC 0.66-0.67 every year), but predicted defaults fell while actual defaults rose.
- For grades A-E, interest rates fell each year after 2013 while borrower FICO fell (about 708 to 696) and DTI rose (about 15.3 to 18.5). A model that relies on the rate read these loans as safer.
- A loss reserve from the model with rate would have been **17.8% short** (165.7M predicted vs 201.5M actual). Without grade and rate it was **3.3% high** (208.3M).
- Input-drift monitoring (PSI) flagged only moderate shifts (FICO 0.17, DTI 0.17, rate 0.14) and the model score was stable (0.06), so it would not have warned of the deterioration. Outcome monitoring is needed, and defaults arrive late.

**Practical conclusion:** use the full model to rank applicants, and a price-free model to estimate losses. This is based on one test year.

## Dashboard

Four Power BI pages:

1. **Overview:** 2015 holdout KPIs, default rate by grade, predicted vs actual by risk decile.
2. **Drift:** interest rate, FICO, DTI and default rate by year and grade.
3. **Policy comparison:** ROI and profit at equal dollars lent.
4. **Model reliability:** predicted vs actual defaults by year, loss reserve, AUC by year.

![Overview page](dashboard/page1_overview.png)
![Model reliability page](dashboard/page4_reliability.png)

## Limitations

- Only accepted loans are visible, so rejected applicants cannot be evaluated (selection bias).
- Only 36-month loans issued 2010-2015 are modeled, to ensure fully observed outcomes. 60-month loans are not covered.
- One test year (2015). Some late-2015 loans may have been open when the data was collected.
- Profit is undiscounted and ignores funding and servicing costs. Returns are lifetime, not annualized.
- Loss severity is a grade-average proxy with interest already netted off, and exposure ignores amortization.
- Logistic reason codes need capped inputs. Uncapped, borrowers with 28-86 public records produced probabilities near 1.0.

## What I would do next

- Add 60-month loans using a survival or vintage-based approach.
- Test on more than one holdout year, with confidence ranges that account for monthly clustering.
- Discount cash flows and include funding costs.
- Add reject inference using the rejected-loans file.

## Repository structure

```
data/         raw and cleaned data (not included, see Data section)
sql/          analysis queries
python/       01_eda.ipynb, 02_model.ipynb
outputs/      scored holdout, policy table, calibration and reserve tables
dashboard/    loan_risk.pbix and screenshots
README.md
requirements.txt
```

## How to reproduce

1. Download `accepted_2007_to_2018Q4.csv` from the Kaggle link above into `data/`.
2. Install dependencies: `pip install -r requirements.txt`
3. Run `python/01_eda.ipynb` to trim, clean and save `loans_clean.csv`, and load it into SQLite.
4. Run `python/02_model.ipynb` to train the models and export the files in `outputs/`.
5. Open `dashboard/loan_risk.pbix` in Power BI Desktop and refresh the data source path.

## Tech stack

Python (pandas, scikit-learn, matplotlib), SQL (SQLite), Power BI (DAX), Jupyter.

## Author

[Your name](ADD-LINKEDIN) | [Email](mailto:ADD-EMAIL)
