# Chapter 6: Exploration Robustness




## What This Notebook Does

In Notebook 02 we found that parental homeownership in 1968 is associated with roughly one additional year of education for children. That is a striking result, but a single regression is not enough. We need to know whether the finding holds up when we push on it.

This notebook asks the hard questions:

Does the effect apply equally across racial groups, or is it driven by one group?

Does it matter whether the child is male or female?

Has the effect changed over time -- is it stronger for older cohorts or younger ones?

Who is missing from our data, and could that missingness be distorting our results?

If we change the age cutoff, does the finding survive?

In short: is this real?

## Key Concepts

Robustness check: Testing whether your main finding still holds when you change how you analyze the data. If we restrict the sample to people aged 30 and older instead of 25 and older, does the coefficient stay roughly the same? If yes, the finding is robust.

Subgroup analysis: Checking whether the effect differs for different groups of people. If homeownership matters for White families but not Black families, that would change the story entirely.

## Part 1: Setup

Same libraries and data path as Notebook 02. We load two files from the Notebook 01.2 pipeline:

psid_intergenerational_clean.csv -- the main analytic sample (3,560 parent-child pairs, complete cases, children aged >= 25 in 2023)

psid_intergenerational_clean_prefilt.csv -- the pre-filter version (11,664 rows, before dropping incomplete cases), used for the missing data analysis in Part 4

We also derive two new variables from birth_year:

child_age -- age in 2023 (ranges from 56 to 89 in this sample)

birth_decade -- for cohort analysis (1930s, 1940s, 1950s, 1960s)

```
# -- Libraries --import pandas as pdimport numpy as npimport statsmodels.formula.api as smfimport matplotlib.pyplot as pltimport seaborn as snsimport os# -- Paths (same as Notebook 02) --BASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'OUTPUT_DIR = f'{BASE}/outputs'INPUT_FILE = f'{OUTPUT_DIR}/psid_intergenerational_clean.csv'PREFILT_FILE = f'{OUTPUT_DIR}/psid_intergenerational_clean_prefilt.csv'# -- Load --df = pd.read_csv(INPUT_FILE)# -- Derive age and birth decade --df['child_age'] = 2023 - df['birth_year']df['birth_decade'] = (df['birth_year'] // 10) * 10df['sex_label'] = df['child_sex'].map({1: 'Male', 2: 'Female'})
```

What loaded: 3,560 parent-child pairs with 22 columns. The sample is 2,195 White, 1,326 Black, 26 Hispanic, and 11 Other. Of the parents, 2,225 were homeowners and 1,335 were renters in 1968.

## Part 2: Family Structure Analysis

## The Question

"How many children did G1 families have?"

Understanding family size helps us check whether homeowners and renters are fundamentally different kinds of families -- different enough that it could confound our results.

### 2.1 Count Children per Parent

We group by parent_id and count how many children (rows) each parent has.

```
kids_per_parent = df.groupby('parent_id').size().reset_index(name='num_children')
```

Result: 2,106 unique G1 parents in the analytic sample.

Family size statistics:
Mean: 1.69 children
Median: 1 child
Standard deviation: 1.10
Range: 1 to 11 children

Distribution: Most families are small. 1,253 parents (59%) have just one child in our sample. 500 have two, 210 have three. Only a handful have six or more.

### 2.2 Family Size Histogram

The histogram shows a strong right skew -- the vast majority of families have one or two children, with a long tail stretching out to 11.

### 2.3 Compare Family Size by Homeownership

The question: Do homeowners have more or fewer children than renters?

The family sizes are virtually identical. Homeowners and renters in our sample had the same number of children on average. This means family size is not a confound -- we are not comparing large renter families against small owner families.

## Part 3: Subgroup Analyses

### The Goal

Test whether the homeownership effect differs across race, sex, and birth cohort. Following Notebook 02, we restrict the main subgroup analyses to White and Black families because the PSID's 1968 sample was designed around these two groups. The Hispanic (26) and Other (11) subsamples are too small for reliable subgroup regressions.

Analysis sample: 3,521 parent-child pairs (White + Black, non-missing on key variables). Of these, 2,200 had parents who owned and 1,321 had parents who rented.

```
df_wb = df[df['race'].isin(['White', 'Black'])].copy()analysis_sample = df_wb.dropna(subset=['child_educ_yrs', 'parent_homeowner', 'child_sex', 'race']).copy()
```

### 3.1 Subgroup Analysis by Race

The question: Does homeownership matter more for some racial groups than others?

We run separate bivariate regressions (child_educ_yrs ~ parent_homeowner) for each race group.

The effect is remarkably consistent across race. White children of homeowners got 0.59 more years of education; Black children of homeowners got 0.58 more years. The difference between these coefficients is trivial. Homeownership mattered equally for both groups.

### 3.2 Subgroup Analysis by Sex

The question: Does homeownership matter more for boys or girls?

Males: +0.99 years of education for children of homeowners vs. renters.
Females: +0.97 years of education for children of homeowners vs. renters.

Again, nearly identical. The homeownership effect does not depend on the child's sex. Boys and girls benefited equally.

### 3.3 Cohort Analysis by Birth Decade

The question: Did the homeownership effect change over time?

This is where the data gets interesting. We run separate regressions for each birth decade. The 1930s cohort (only 11 observations) is too small and is skipped.

The effect is significant in every decade -- but it is shrinking over time. For children born in the 1940s, having parents who owned their home was associated with over two additional years of education. By the 1960s cohort, the association had dropped to about two-thirds of a year.

What this might mean: As access to education expanded across the mid-20th century, the advantage conferred by homeownership may have diminished. Or it could reflect changing characteristics of homeowners over time. Either way, the trend is clear and consistent: the homeownership-education link is weakening across cohorts.

## Part 4: Missing Data Analysis

### The Goal

Understand who is missing from our analysis and whether missingness is systematic. We load the pre-filter file for this section -- 11,664 parent-child pairs before dropping incomplete cases.

How much was dropped? Of the 11,664 rows in the pre-filter file, 8,104 (69.5%) were dropped to reach the 3,560-row analytic sample. That is a lot of data.

### 4.1 Missing Data Patterns

The parent variables are nearly complete. The problem is on the child side: almost 70% of children are missing education and homeownership outcomes. This is expected -- ER35152 (education years) and ER82032 (homeownership in 2023) are only available for individuals who were tracked through the most recent PSID waves. Many children in the 1968 sample were lost to follow-up over the decades.

### 4.2 Does Missingness Relate to Homeownership?

This is the critical question. If education data is equally likely to be missing for children of owners and renters, our results are probably fine. If not, we may have selection bias.

The difference is 10.8 percentage points. Children of renters are more likely to be missing education data than children of owners. This is a caution flag -- it means our analytic sample may overrepresent children of homeowners relative to the full population of PSID children. The estimates could be biased if the missing renter children would have had different education outcomes than the observed renter children.

This does not invalidate our findings, but it is something a careful reader should know about.

## Part 5: Robustness Checks

### 5.1 Alternative Age Cutoffs

The question: Does our finding change if we use different age cutoffs?

The main analysis (Notebook 01.2) restricts to children aged >= 25 in 2023. Here we test whether different cutoffs change the result. Because everyone in our analytic sample is already aged 56-89 (born 1934-1967), all four cutoffs produce the same sample.

Identical across all cutoffs. This makes sense: the youngest person in our sample is 56 years old, so any cutoff below 56 retains everyone. The finding is trivially robust to age restrictions -- they simply do not bind in this sample.

### 5.2 Full Sample Including Hispanic & Other

The main analysis uses only White and Black families. Here we check whether including all races changes the estimate.

The difference is 0.024 years -- negligible. The finding is robust to sample composition. Including the small Hispanic and Other subgroups does not meaningfully change the estimate.

## Summary: What We Learned

Family structure: Average family size is 1.69 children per G1 parent. Owners and renters have virtually identical family sizes (1.69 vs 1.70). Family size is not a confound.

By race: The homeownership effect is nearly identical for White families (+0.59 years) and Black families (+0.58 years). The effect is universal across race.

By sex: The effect is nearly identical for males (+0.99 years) and females (+0.97 years). The effect is universal across sex.

By cohort: The effect is significant in every decade but shrinks over time -- from +2.16 years for the 1940s cohort to +0.66 years for the 1960s cohort. The homeownership advantage in education appears to be fading.

Missing data: 69.5% of child education data is missing in the pre-filter file. Missingness is not random -- children of renters are missing at a higher rate (75.3%) than children of owners (64.5%). This differential attrition is a limitation worth noting.

Age cutoffs: The finding is robust -- identical coefficients across all age cutoffs tested.

Full sample: Including Hispanic and Other families changes the coefficient by only 0.024 years. The finding is robust to sample composition.

Overall assessment: The homeownership-education association is real and robust. It holds across race, sex, age cutoffs, and sample definitions. The one nuance is temporal: the effect appears to weaken across birth cohorts, and the differential missing data rates between owners and renters warrant caution in interpretation.

Next up: we zoom out to three generations. If homeownership shaped the children, what happened to the grandchildren?












