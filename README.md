# College Scorecard Program Value Analysis

Which undergraduate programs give students the most earnings for the debt they take on, and what do the best-value programs have in common?

**Tools:** SQL Server (T-SQL), Python (pandas, SQLAlchemy, statsmodels), Power BI
**Data:** U.S. Department of Education [College Scorecard](https://collegescorecard.ed.gov/data/), downloaded October 2026

## Summary

I analyzed 223,587 program-level records (school by field of study by credential) from two federal datasets, of which 49,931 report earnings after removing repeated campus rows. For bachelor's degrees, three results stand out:

1. **Field of study explains far more of the variation in earnings than school traits do.** Field alone explains about 62% to 67%, while school type, admission rate and Pell share together explain about 12% to 14%.
2. **Debt differs sharply by school type.** In the same field, public and private nonprofit programs carry about 36% and 28% less median debt than for-profit programs, and return roughly 1.7 to 1.9 times the earnings per dollar of debt.
3. **The best-value programs are computer science at selective universities** (for example Wellesley, Penn, Johns Hopkins and MIT), with debt under about $10K against earnings of roughly $177K to $241K. The typical bachelor's program returns about $2.88 of earnings per $1 of debt.

## Business questions

1. Which programs give the best earnings relative to debt?
2. What matters more for earnings: the field of study or the school?
3. For the same field and credential, do public, private nonprofit and for-profit schools differ?
4. Which programs beat or miss the national median for their own field?
5. Which programs combine low earnings with high debt?

## Data

| File | Description | Size |
| --- | --- | --- |
| Most Recent Data by Field of Study | Earnings and debt by school, field and credential | 223,587 rows, 230 columns |
| Most Recent Institution-Level Data | School traits (control, admission rate, Pell share) | 6,278 schools, trimmed from 3,379 columns to 10 |
| Data dictionary | Column definitions | n/a |

Raw files are not committed because of their size. Download them from the [College Scorecard data page](https://collegescorecard.ed.gov/data/) into `data/raw/`. The Scorecard site lists this release as updated June 10, 2026.

## Project structure

```text
data/raw/        downloaded CSVs (not committed)
sql/             02_profile.sql, 03_clean.sql, 04_views.sql, 05_dedupe.sql
notebooks/       data loading and trimming, then analysis and models
dashboard/       Power BI file and screenshots
README.md
```

## How to reproduce

1. Create a SQL Server database named `Scorecard`.
2. Run the loading notebook, which trims the institution file to 10 columns and loads both files as text into `stg_field_of_study` and `stg_institution`.
3. Run the SQL scripts in `sql/` in numeric order (profile, clean, views, de-duplicate).
4. Run the analysis notebook to reproduce the models and charts.
5. Open the Power BI file in `dashboard/` and refresh it against your local server.

## Method

- **Loading:** Loaded everything as text first, because numeric columns contain the marker `PS` (privacy suppressed) and the text `NULL`.
- **Cleaning:** Converted values with `TRY_CAST` so suppressed values become real NULLs, joined the two files on `UNITID`, and trimmed trailing periods from field names.
- **Value metric:** Median earnings four years after completion divided by median federal debt. Years to repay assumes 10% of annual earnings goes to repayment, with interest ignored.
- **Minimum group size:** Rankings and comparisons use programs with at least 30 students behind both earnings and debt. Results were checked at minimums of 20 and 50.
- **De-duplication:** Scorecard repeats an institution's figures on each of its campuses, so I kept one row per institution group, field and credential.
- **Modeling:** Regressions of log earnings on field, school type, admission rate and Pell share (bachelor's programs), with errors clustered by institution group.
- **High-risk flag:** Debt above half of one year's earnings. I tested cutoffs from 0.25 to 1.0; at 0.25 about three in four programs were flagged (measured before de-duplication), so it separated nothing.

### Cleaning log

| Step | Rows |
| --- | --- |
| Field of study records loaded | 223,587 |
| Earnings suppressed (`PS`) or missing | 167,963 (usable: 55,624) |
| Debt suppressed (`PS`) | 185,946 (usable: 37,641) |
| Rows with both earnings and debt, before de-duplication | 30,778 |
| Rows with no school ID (kept and flagged) | 5,573 |
| Repeated campus rows among rows with earnings | 7,940 rows in 2,247 groups, 5,693 removed |
| Earnings rows after de-duplication | 49,931 |
| Rows with both earnings and debt, after de-duplication | 26,436 |

The repeated campus rows had identical earnings in every group. For-profit programs were affected most: the for-profit bachelor's sample fell from 646 to 284 programs, and the debt gap between school types narrowed from about 39% and 46% to 28% and 36%.

## Key findings

### 1. Field of study dominates school traits
- **Finding:** Field of study explains about two-thirds of the variation in bachelor's program earnings (adjusted R-squared 0.62 to 0.67), versus 12% to 14% for school type, admission rate and Pell share combined.
- **Evidence:** Three models on 9,936 to 20,248 programs at minimum group sizes of 50, 30 and 20 students, with consistent results.
- **So what:** Students comparing programs should weigh field of study far more than school type or selectivity when it comes to earnings.
- **Caveat:** Field has many more categories than the school measures, and this shows association, not cause.

### 2. Debt is where school type matters
- **Finding:** For the same bachelor's field, public and private nonprofit programs carry about 36% and 28% less median debt than for-profit programs (intervals roughly 29% to 43% and 19% to 36%), with similar or higher earnings. The typical program returns $2.96 and $2.76 of earnings per $1 of debt versus $1.60.
- **Evidence:** Medians for 12,364 programs (8,686 public, 3,394 nonprofit, 284 for-profit) plus regressions within field.
- **So what:** Students should compare debt as carefully as earnings.
- **Caveat:** Only 284 for-profit programs, school type reflects who enrolls, and the for-profit earnings comparison changed with the model used, so the robust result is the debt gap.

### 3. The best-value programs are selective computer science
- **Finding:** The top of the bachelor's ranking is dominated by computer science at selective private and public flagship universities, where low debt and high earnings combine.
- **Evidence:** Top 25 programs by earnings per $1 of debt among programs with 30+ students.
- **So what:** High value at these programs reflects who attends and borrows, so it doesn't mean other programs should copy them.
- **Caveat:** Debt covers borrowers only, so programs where few students borrow can look better than they are.

### Data quality notes
- Repeated campus rows would have inflated the sample and the debt gap if left in.
- The lowest earnings relative to the national median were all in Puerto Rico, mostly nursing programs. National figures aren't adjusted for local wages, so I treat this as a geographic effect.

## Dashboard

A four-page Power BI report (`dashboard/scoreboard.pbix`) built on one SQL Server view, with slicers for degree type, field of study, state and school type.

| Page | Question | What it shows |
| --- | --- | --- |
| Degree Value | Which programs give the best earnings per dollar of debt? | Ranked table of top programs, plus a scatter plot of debt against earnings for every program |
| Watch List | Which programs combine high debt with low earnings? | High-risk programs ranked by debt-to-earnings, with the share of programs flagged by school type |
| Public vs Private | Does school type matter? | Median earnings, median debt and earnings per $1 of debt by school type, with program counts |
| About/Methodology | How was this built? | Question, data source, cleaning steps, metric definitions, key findings and limitations |

![Degree Value](dashboard\DegreeValue.png)
![Watch List](dashboard\WatchList.png)
![Public vs Private](dashboard\PublicvsPrivate.png)

Questions 2 (field versus school) and 4 (outliers against the national median) are answered in the Python notebook rather than the dashboard.
## Limitations

- Earnings and debt describe students who received federal financial aid, so they may not represent everyone in a program.
- Programs with few graduates are suppressed, so results lean toward larger programs.
- Debt reflects federal loans only, and debt and earnings are both only available for undergraduate credentials in this file.
- Earnings are measured a fixed number of years after completion and are not adjusted for cost of living.
- Results show associations, not causes.

## Next steps

- Rerun the state-level analysis on the de-duplicated data to measure how much location adds beyond field and school type.
- Check whether other debt columns cover graduate programs, which would extend the value analysis beyond undergraduate credentials.
- Add a page comparing programs to their national field median (question 4).
