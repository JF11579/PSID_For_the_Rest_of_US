# Chapter 8: Regression Report


This is the full Regression Report I promised back in Chapter 2. That chapter gave you the headline — kids whose parents own their homes get about one extra year of schooling. This chapter gives you every number behind that headline, plus the homeownership results I haven't shown you yet. If tables full of coefficients and p-values make your eyes glaze over, don't worry — each one comes with a plain-English walkthrough.



## Table 1 — Summary Statistics (White & Black Families)

## Table 1b — Means by Parent Homeownership Status

## Parent Homeownership Rate by Race

## Homeownership Transition Matrix — Counts

## Homeownership Transition Matrix — Row Percentages

## Table 2 — Education Regressions (OLS)

Dependent variable: Child's years of education Reference group: White male whose parents rented in 1968 Significance: *** p<0.001 ** p<0.01 * p<0.05

Standard errors in parentheses. Model 1 intercept = actual average education for renters' children. Models 2–3 intercepts are mathematical baselines (continuous controls set to 0).

### How to Read the Education Results

Model 1 (no controls): Children of renters averaged 13.2 years of education. Children of homeowners averaged 0.99 years more — about one extra year. This is the raw, unadjusted gap.

Model 2 (with controls): After controlling for parent education, parent income, race, and sex, parent homeownership is associated with +0.340 years of additional education. This is statistically significant (p < 0.001). The homeownership effect dropped from 0.99 to 0.34 years once controls are added. This means 65% of the raw gap was explained by other family characteristics.

Race: Black children received +0.05 years compared to White children, holding parent income, education, homeownership, and sex constant — not statistically significant.

Sex: Females received −0.01 years compared to Males — not statistically significant.

Model 3 (+ state fixed effects): Adding state fixed effects barely changes the homeownership coefficient (+0.319 vs. +0.340). The result is robust to geographic controls.

## Table 3 — Homeownership Regressions (Linear Probability Model)

Dependent variable: Child homeownership in 2023 (1 = owner, 0 = renter) Coefficients = change in probability (percentage points) Reference group: White male whose parents rented in 1968 Significance: *** p<0.001 ** p<0.01 * p<0.05

Standard errors in parentheses. Model 4 intercept = actual homeownership rate for renters' children. Models 5–6 intercepts are mathematical baselines (continuous controls set to 0).

### How to Read the Homeownership Results

Model 4 (no controls): 59.3% of children whose parents rented went on to own a home. Children of owners were +21.5 percentage points more likely to own — so about 81% of owners' children became homeowners.

Model 5 (with controls): After controlling for parent education, income, race, and sex, parent homeownership is associated with a +12.5 percentage points increase in the probability of child homeownership. This is statistically significant (p < 0.001). The effect shrank from 21.5 to 12.5 percentage points after adding controls.

Race: Black children were −18.9 percentage points less likely to own a home than White children, holding all else constant. This large gap even after controls points to structural barriers (lending discrimination, segregation, wealth gaps not captured by income).

Sex: Females were −5.5 percentage points less likely to own than Males, holding all else constant.

Parent education and income are not significant in the homeownership models — their effect is absorbed by the homeownership variable itself.

Model 6 (+ state fixed effects): The homeownership coefficient drops slightly further to +10.3 percentage points but remains highly significant. The race gap widens to −20.4 pp.

## Robustness: Full Sample (Including Hispanic & Other)

The main analysis uses only White and Black families because the PSID's 1968 sample was designed around these two groups. Hispanic (N about 24) and Other (N about 11) subsamples are too small for reliable inference.

Re-running Models 2 and 5 on the full sample confirms the results are substantively identical:

Dropping the smaller subgroups does not materially change the findings.

## Key Findings Summary

## Methods Note

Data: Panel Study of Income Dynamics (PSID), 1968–2023. The analytic sample comprises adult children linked to their 1968 household heads, restricted to White and Black families (N = 3,521; after listwise deletion, N = 3,491).

Education models (M1–M3): OLS. Dependent variable is child's completed years of education. Coefficients represent years of schooling.

Homeownership models (M4–M6): Linear Probability Model (OLS with binary dependent variable). Dependent variable is child homeownership in 2023 (1 = owner, 0 = renter). Coefficients represent changes in probability (percentage points).

Reference group: White males whose parents rented in 1968. In Models 2–3 and 5–6, the intercept represents a mathematical baseline (all continuous controls set to zero) and should not be interpreted as a real-world average. Model 1 and Model 4 intercepts give the actual group means for renters' children.





You've seen what the data says. Next up: what you can do with it — and why the PSID is waiting for your question, not just mine.

© 2026 Joe Foley. All rights reserved (text). MIT License (notebook code).
