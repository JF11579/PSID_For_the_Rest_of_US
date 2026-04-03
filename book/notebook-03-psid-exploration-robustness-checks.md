# Notebook 03 PSID Exploration & Robustness Checks


# š PSID Exploration & Robustness Checks

## What This Notebook Does

After finding that homeownership is associated with ~1 year more child education, we naturally wonder:

"Is this finding robust? Does it hold across different groups? What else can we learn from the data?"

### The Goal

This notebook explores the data more deeply and tests the robustness of our main finding through:
1. 
Family Structure Analysis ā How many children did G1 families have?
2. 
Subgroup Analyses ā Does the effect vary by race? By sex?
3. 
Cohort Analysis ā Does the effect differ by birth decade?
4. 
Missing Data Patterns ā Who's missing from our analysis?
5. 
Alternative Specifications ā Different age cutoffs, model variations

Why This Matters:
- 
Robustness ā Does our finding hold under different assumptions?
- 
Generalizability ā Does the effect apply to all groups?
- 
Understanding ā What factors moderate or mediate the effect?

## š Key Concepts

### What is a "Robustness Check"?

Simple explanation: "Testing whether your main finding still holds when you change how you analyze the data."

Example:
- 
Main analysis: Age ā„ 25
- 
Robustness: Try age ā„ 21, 23, 30
- 
If results stay similar ā Finding is "robust"

### What is a "Subgroup Analysis"?

Simple explanation: "Checking if the effect differs for different groups of people."

Example:
- 
Does homeownership matter more for Black families?
- 
Is the effect stronger for children born in certain decades?

# Part 1: Setup

Same libraries and data path as Notebook 02. We load two files:
- 
psid_intergenerational_clean.csv ā the main analytic sample (complete cases, age ā„ 25)
- 
psid_intergenerational_clean_prefilt.csv ā the pre-filter version (before dropping incomplete cases), used for the missing data analysis in Part 4

```
# āā Libraries āāimport pandas as pdimport numpy as npimport statsmodels.formula.api as smfimport matplotlib.pyplot as pltimport seaborn as snsimport os# āā Style āāsns.set_style('whitegrid')plt.rcParams.update({    'figure.figsize': (10, 6),    'font.size': 12,    'axes.titlesize': 14,    'axes.labelsize': 12})print("ā Libraries loaded!")
```

```
ā Libraries loaded!
```

```
# āā Mount Google Drive (Colab only) āāfrom google.colab import drivedrive.mount('/content/drive')# āā Paths (same as Notebook 02) āāBASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'DATA_DIR = f'{BASE}/data'OUTPUT_DIR = f'{BASE}/outputs'INPUT_FILE = f'{OUTPUT_DIR}/psid_intergenerational_clean.csv'PREFILT_FILE = f'{OUTPUT_DIR}/psid_intergenerational_clean_prefilt.csv'# āā Verify āāif not os.path.exists(BASE):    print(f'ERROR: BASE path not found: {BASE}')else:    print(f'BASE path OK: {BASE}')    print(f'Files in outputs/: {os.listdir(OUTPUT_DIR)}')
```

```
Mounted at /content/driveBASE path OK: /content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2Files in outputs/: ['psid_intergenerational_clean.csv', 'psid_intergenerational_clean_prefilt.csv', 'table2_education_regressions.csv', 'table3_homeownership_regressions.csv', 'fig2_homeownership_by_race.png', 'fig1_homeownership_by_parent_status.png', 'table3_homeownership_regression_plain.csv', 'table1_summary_stats.csv', 'table2_education_regression_plain.csv', 'fig3_coefficient_plots.png', 'fig4_education_by_race.png']
```

```
# āā Load the main analytic sample āādf = pd.read_csv(INPUT_FILE)print(f"ā Data loaded: {len(df):,} parent-child pairs")print(f"   Columns: {list(df.columns)}")# āā Quick check: confirm key columns are present āāexpected = ['child_id', 'parent_id', 'birth_year', 'child_sex',            'child_educ_yrs', 'child_homeowner_2023',            'parent_homeowner', 'parent_educ_yrs', 'race']missing = [c for c in expected if c not in df.columns]if missing:    print(f"ā ļø  Missing expected columns: {missing}")else:    print("ā All expected columns present")# āā Derive child_age (age in 2023) and birth_decade āādf['child_age'] = 2023 - df['birth_year']df['birth_decade'] = (df['birth_year'] // 10) * 10# āā Sex label for readability āādf['sex_label'] = df['child_sex'].map({1: 'Male', 2: 'Female'})print(f"\n   Age range: {df['child_age'].min():.0f} ā {df['child_age'].max():.0f}")print(f"   Birth decades: {sorted(df['birth_decade'].dropna().unique().astype(int))}")print(f"\n   Race distribution:")print(df['race'].value_counts())print(f"\n   Parent homeowner:")print(df['parent_homeowner'].value_counts())
```

```
ā Data loaded: 3,560 parent-child pairs   Columns: ['child_id', 'parent_id', 'birth_year', 'child_sex', 'child_educ_yrs', 'child_homeowner_2023', 'parent_homeowner', 'parent_educ_yrs', 'parent_income_1968', 'parent_income_1968_log', 'race', 'state_1968', 'head_sex_1968', 'V103', 'V313', 'V81', 'V181', 'V93', 'V119', 'ER35152', 'ER82032', 'ER32000']ā All expected columns present   Age range: 56 ā 89   Birth decades: [np.int64(1930), np.int64(1940), np.int64(1950), np.int64(1960)]   Race distribution:raceWhite       2195Black       1326Hispanic      26Other         11Name: count, dtype: int64   Parent homeowner:parent_homeowner1.0    22250.0    1335Name: count, dtype: int64
```

# Part 2: Family Structure Analysis

## The Question

"How many children did G1 families have?"

Why This Matters:

Understanding family size helps us:
- 
Assess data completeness (Are we capturing all children?)
- 
Understand context (Small families? Large families?)
- 
Spot potential issues (Missing children?)

What We'll Calculate:
- 
Number of children per G1 parent
- 
Distribution of family sizes
- 
Average family size by homeownership status

## 2.1 Count Children per Parent

How this works:

We group by parent_id and count how many rows (children) each parent has.

What .groupby() does: "Collect all rows with the same parent_id and count them."

```
# Count children per parentkids_per_parent = df.groupby('parent_id').size().reset_index(name='num_children')print(f"ā Analyzed {len(kids_per_parent):,} unique G1 parents")# Summary statisticsprint("\nš Family Size Statistics:")print(kids_per_parent['num_children'].describe())# Distributionprint("\nš Distribution of Family Sizes:")print(kids_per_parent['num_children'].value_counts().sort_index().head(15))
```

```
ā Analyzed 2,106 unique G1 parentsš Family Size Statistics:count    2106.000000mean        1.690408std         1.104116min         1.00000025%         1.00000050%         1.00000075%         2.000000max        11.000000Name: num_children, dtype: float64š Distribution of Family Sizes:num_children1     12532      5003      2104       905       256       187        48        29        211       2Name: count, dtype: int64
```

## 2.2 Visualize Family Size Distribution

What we're creating: A histogram showing how common different family sizes are.

What to look for:
- 
What's the most common family size?
- 
Are there many very large families?
- 
Any unusual patterns?

```
# Create family size histogramfig, ax = plt.subplots(figsize=(12, 6))# Histogramax.hist(kids_per_parent['num_children'],        bins=range(1, kids_per_parent['num_children'].max() + 2),        edgecolor='black', alpha=0.7, color='steelblue')# Add mean linemean_kids = kids_per_parent['num_children'].mean()ax.axvline(mean_kids, color='red', linestyle='--', linewidth=2,           label=f'Mean: {mean_kids:.2f} children')# Labelsax.set_xlabel('Number of Children per G1 Parent', fontsize=12)ax.set_ylabel('Number of Parents', fontsize=12)ax.set_title('Distribution of Family Sizes (G1 ā G2)', fontsize=14, fontweight='bold')ax.legend()ax.grid(axis='y', alpha=0.3)plt.tight_layout()plt.savefig(f'{OUTPUT_DIR}/family_size_distribution.png', dpi=300, bbox_inches='tight')print("ā Plot saved")plt.show()
```

```
ā Plot saved
```

## 2.3 Compare Family Size by Homeownership

The Question: "Do homeowners have more or fewer children than renters?"

Why This Matters: If homeowners have systematically different family sizes, that could influence our education findings.

```
# Get homeownership status for each parentparent_hw = df[['parent_id', 'parent_homeowner']].drop_duplicates()kids_with_owner = kids_per_parent.merge(parent_hw, on='parent_id', how='left')# Compare family sizesprint("š Family Size by Homeownership Status:\n")for status, label in [(0, "Renters"), (1, "Owners")]:    group = kids_with_owner[kids_with_owner['parent_homeowner'] == status]['num_children']    print(f"{label}:")    print(f"  Mean: {group.mean():.2f} children")    print(f"  Median: {group.median():.0f} children")    print(f"  Std Dev: {group.std():.2f}\n")# Visual comparisonfig, ax = plt.subplots(figsize=(10, 6))owners  = kids_with_owner[kids_with_owner['parent_homeowner'] == 1]['num_children']renters = kids_with_owner[kids_with_owner['parent_homeowner'] == 0]['num_children']ax.hist([renters, owners], label=['Renters', 'Owners'], bins=range(1, 15),        alpha=0.6, color=['coral', 'steelblue'])ax.set_xlabel('Number of Children', fontsize=12)ax.set_ylabel('Frequency', fontsize=12)ax.set_title('Family Size by Homeownership Status', fontsize=14, fontweight='bold')ax.legend()ax.grid(axis='y', alpha=0.3)plt.tight_layout()plt.show()
```

```
š Family Size by Homeownership Status:Renters:  Mean: 1.70 children  Median: 1 children  Std Dev: 1.20Owners:  Mean: 1.69 children  Median: 1 children  Std Dev: 1.04
```

# Part 3: Subgroup Analyses

## The Goal

Test whether the homeownership effect differs across:
- 
Race groups (White, Black)
- 
Sex (Male vs. Female children)
- 
Birth cohorts (Different decades)

Why This Matters:

If the effect is much stronger (or only present) in certain groups, that tells us:
- 
Who benefits most from homeownership
- 
Whether the effect is universal
- 
Potential mechanisms

Note: Notebook 02 restricts the main analysis to White and Black families because the PSID's 1968 sample was designed around these two groups. Hispanic and Other subsamples have fewer than 20 observations each ā too small for reliable subgroup regressions. We follow the same approach here.

## 3.1 Prepare Analysis Sample

Use the same sample restrictions as Notebook 02: White and Black families only, with non-missing education, homeownership, sex, and race.

```
# āā Create analysis sample (same as Notebook 02) āā# Restrict to White + Black (Hispanic/Other subsamples too small)df_wb = df[df['race'].isin(['White', 'Black'])].copy()# Drop rows missing key analysis variablesanalysis_vars = ['child_educ_yrs', 'parent_homeowner', 'child_sex', 'race']analysis_sample = df_wb.dropna(subset=analysis_vars).copy()print(f"Full dataset:        {len(df):,}")print(f"White + Black only:  {len(df_wb):,}")print(f"Analysis sample:     {len(analysis_sample):,}  (non-missing on key vars)")print(f"\nRace breakdown:")print(analysis_sample['race'].value_counts())print(f"\nParent homeowner:")print(analysis_sample['parent_homeowner'].value_counts())
```

```
Full dataset:        3,560White + Black only:  3,521Analysis sample:     3,521  (non-missing on key vars)Race breakdown:raceWhite    2195Black    1326Name: count, dtype: int64Parent homeowner:parent_homeowner1.0    22000.0    1321Name: count, dtype: int64
```

## 3.2 Subgroup Analysis by Race

The Question: "Does homeownership matter more for some racial groups than others?"

How we test this: Run separate regressions for each race group.

What to look for:
- 
Are coefficients similar across groups?
- 
Is the effect significant in all groups?
- 
Which group benefits most?

```
# āā Subgroup analysis by race āāprint("=" * 80)print("š SUBGROUP ANALYSIS: BY RACE")print("=" * 80)results = []for race_name in ['White', 'Black']:    subgroup = analysis_sample[analysis_sample['race'] == race_name]    if len(subgroup) < 30:        print(f"\nā ļø  {race_name}: Too few observations ({len(subgroup)})")        continue    # Run regression: education ~ homeownership    model = smf.ols('child_educ_yrs ~ parent_homeowner', data=subgroup).fit()    coef = model.params['parent_homeowner']    pval = model.pvalues['parent_homeowner']    n = len(subgroup)    results.append({        'Race': race_name,        'N': n,        'Coefficient': round(coef, 3),        'P-value': round(pval, 4)    })    sig = "***" if pval < 0.001 else "**" if pval < 0.01 else "*" if pval < 0.05 else "n.s."    print(f"\n{race_name}:")    print(f"  N: {n:,}")    print(f"  Homeownership Effect: {coef:.3f} years {sig}")    print(f"  P-value: {pval:.4f}")# Summary comparisonif results:    results_df = pd.DataFrame(results)    print("\n" + "=" * 80)    print("š SUMMARY COMPARISON:")    print(results_df.to_string(index=False))    print("\nš” Interpretation: Compare the coefficients across groups.")    print("   Larger differences suggest the effect varies by race.")print("\n" + "=" * 80)
```

```
================================================================================š SUBGROUP ANALYSIS: BY RACE================================================================================White:  N: 2,195  Homeownership Effect: 0.590 years ***  P-value: 0.0000Black:  N: 1,326  Homeownership Effect: 0.580 years ***  P-value: 0.0000================================================================================š SUMMARY COMPARISON: Race    N  Coefficient  P-valueWhite 2195         0.59      0.0Black 1326         0.58      0.0š” Interpretation: Compare the coefficients across groups.   Larger differences suggest the effect varies by race.================================================================================
```

## 3.3 Subgroup Analysis by Sex

The Question: "Does homeownership matter more for boys or girls?"

```
# āā Subgroup analysis by sex āāprint("=" * 80)print("š SUBGROUP ANALYSIS: BY SEX")print("=" * 80)for sex_code, sex_name in [(1, "Male"), (2, "Female")]:    subgroup = analysis_sample[analysis_sample['child_sex'] == sex_code]    model = smf.ols('child_educ_yrs ~ parent_homeowner', data=subgroup).fit()    coef = model.params['parent_homeowner']    pval = model.pvalues['parent_homeowner']    sig = "***" if pval < 0.001 else "**" if pval < 0.01 else "*" if pval < 0.05 else "n.s."    print(f"\n{sex_name}:")    print(f"  N: {len(subgroup):,}")    print(f"  Homeownership Effect: {coef:.3f} years {sig}")    print(f"  P-value: {pval:.4f}")print("\n" + "=" * 80)
```

```
================================================================================š SUBGROUP ANALYSIS: BY SEX================================================================================Male:  N: 1,508  Homeownership Effect: 0.993 years ***  P-value: 0.0000Female:  N: 2,013  Homeownership Effect: 0.971 years ***  P-value: 0.0000================================================================================
```

## 3.4 Cohort Analysis by Birth Decade

The Question: "Did the homeownership effect change over time?"

Why This Matters:
- 
Educational opportunities changed dramatically across decades
- 
Economic conditions varied
- 
Homeownership may have mattered more in some eras

Note: Remember that this sample is restricted to children aged ā„ 25 in 2023 (born ā¤ 1998). The youngest cohorts will have fewer observations.

```
# āā Cohort analysis by birth decade āāprint("=" * 80)print("š COHORT ANALYSIS: BY BIRTH DECADE")print("=" * 80)decades = sorted(analysis_sample['birth_decade'].dropna().unique().astype(int))results = []for decade in decades:    subgroup = analysis_sample[analysis_sample['birth_decade'] == decade]    if len(subgroup) < 50:  # Skip decades with too few observations        print(f"\nā ļø  {decade}s: Too few observations ({len(subgroup)}) ā skipped")        continue    model = smf.ols('child_educ_yrs ~ parent_homeowner', data=subgroup).fit()    coef = model.params['parent_homeowner']    pval = model.pvalues['parent_homeowner']    n = len(subgroup)    results.append({        'Decade': decade,        'N': n,        'Coefficient': coef,        'P-value': pval    })    sig = "***" if pval < 0.001 else "**" if pval < 0.01 else "*" if pval < 0.05 else "n.s."    print(f"\n{decade}s:")    print(f"  N: {n:,}")    print(f"  Homeownership Effect: {coef:.3f} years {sig}")# Plot trends over timeif results:    results_df = pd.DataFrame(results)    fig, ax = plt.subplots(figsize=(10, 6))    ax.plot(results_df['Decade'], results_df['Coefficient'],            marker='o', linewidth=2, markersize=10, color='steelblue')    ax.axhline(0, color='red', linestyle='--', alpha=0.5)    ax.set_xlabel('Birth Decade', fontsize=12)    ax.set_ylabel('Homeownership Effect on Education (Years)', fontsize=12)    ax.set_title('Homeownership Effect by Birth Cohort', fontsize=14, fontweight='bold')    ax.grid(True, alpha=0.3)    plt.tight_layout()    plt.savefig(f'{OUTPUT_DIR}/cohort_analysis.png', dpi=300, bbox_inches='tight')    print("\nā Plot saved")    plt.show()print("\n" + "=" * 80)
```

```
================================================================================š COHORT ANALYSIS: BY BIRTH DECADE================================================================================ā ļø  1930s: Too few observations (11) ā skipped1940s:  N: 320  Homeownership Effect: 2.164 years ***1950s:  N: 1,631  Homeownership Effect: 1.124 years ***1960s:  N: 1,559  Homeownership Effect: 0.655 years ***ā Plot saved
```

```
================================================================================
```

# Part 4: Missing Data Analysis

## The Goal

Understand who's missing from our analysis and whether missingness is systematic.

Why This Matters:

If data is "missing at random," our results are fine. But if missingness is systematic (e.g., all homeowners are missing education data), we could have biased results.

Important: For this section we load the pre-filter version of the data (psid_intergenerational_clean_prefilt.csv). This file contains all parent-child pairs aged ā„ 25 in 2023 before dropping rows with missing outcomes. That way we can actually see who's missing what.

## 4.1 Missing Data Patterns

What we're checking:
- 
Which variables have missing data?
- 
How much is missing?
- 
Is missingness related to homeownership?

```
# āā Load pre-filter data for missing data analysis āādf_prefilt = pd.read_csv(PREFILT_FILE)print(f"Pre-filter dataset: {len(df_prefilt):,} rows")print(f"(Main analytic sample: {len(df):,} rows)")print(f"Difference: {len(df_prefilt) - len(df):,} rows dropped for missing data")# āā Missing data by variable āāprint("\n" + "=" * 80)print("š MISSING DATA ANALYSIS")print("=" * 80)key_vars = ['parent_homeowner', 'parent_educ_yrs', 'parent_income_1968',            'child_educ_yrs', 'child_homeowner_2023', 'race', 'child_sex']print("\nš Missing Data by Variable:\n")for var in key_vars:    if var in df_prefilt.columns:        n_missing = df_prefilt[var].isna().sum()        pct_missing = (n_missing / len(df_prefilt)) * 100        print(f"{var:25s}: {n_missing:7,} missing ({pct_missing:5.1f}%)")    else:        print(f"{var:25s}: ā ļø  column not in pre-filter file")print("\n" + "=" * 80)
```

```
Pre-filter dataset: 11,664 rows(Main analytic sample: 3,560 rows)Difference: 8,104 rows dropped for missing data================================================================================š MISSING DATA ANALYSIS================================================================================š Missing Data by Variable:parent_homeowner         :       0 missing (  0.0%)parent_educ_yrs          :     104 missing (  0.9%)parent_income_1968       :       2 missing (  0.0%)child_educ_yrs           :   8,104 missing ( 69.5%)child_homeowner_2023     :   7,977 missing ( 68.4%)race                     :      23 missing (  0.2%)child_sex                :       0 missing (  0.0%)================================================================================
```

## 4.2 Check if Missingness Relates to Homeownership

Critical question: "Is education data more likely to be missing for homeowners vs. renters?"

If yes ā potential bias If no ā probably okay

```
# āā Does missingness relate to homeownership? āāprint("=" * 80)print("š DOES MISSINGNESS RELATE TO HOMEOWNERSHIP?")print("=" * 80)# Only check among rows where parent_homeowner is knowndf_hw_known = df_prefilt[df_prefilt['parent_homeowner'].notna()].copy()# Create missingness indicator for child educationdf_hw_known['educ_missing'] = df_hw_known['child_educ_yrs'].isna()# Compare missingness ratesmissing_by_owner = df_hw_known.groupby('parent_homeowner')['educ_missing'].mean() * 100print("\nš Education Missing Rate by Homeownership:\n")print(f"  Renters (0): {missing_by_owner.get(0, 0):.1f}% missing")print(f"  Owners  (1): {missing_by_owner.get(1, 0):.1f}% missing")difference = abs(missing_by_owner.get(1, 0) - missing_by_owner.get(0, 0))if difference < 5:    print("\nā GOOD: Missing rates are similar between groups")    print("   Missingness appears to be independent of homeownership")else:    print("\nā ļø  WARNING: Missing rates differ substantially")    print(f"   Difference: {difference:.1f} percentage points")    print("   This could introduce bias into our estimates")print("\n" + "=" * 80)
```

```
================================================================================š DOES MISSINGNESS RELATE TO HOMEOWNERSHIP?================================================================================š Education Missing Rate by Homeownership:  Renters (0): 75.3% missing  Owners  (1): 64.5% missingā ļø  WARNING: Missing rates differ substantially   Difference: 10.8 percentage points   This could introduce bias into our estimates================================================================================
```

# Part 5: Robustness Checks

## The Goal

Test whether our main finding holds under different assumptions.

Tests we'll run:
1. 
Different age cutoffs (21, 23, 25, 30)
2. 
Full sample including Hispanic & Other (same as Notebook 02 Section 5)

Main analysis age cutoff: ā„ 25 (set in Notebook 01.2)

## 5.1 Alternative Age Cutoffs

The Question: "Does our finding change if we use different age cutoffs for 'observable' education?"

Main analysis: Age ā„ 25 (from Notebook 01.2) Robustness: Try age ā„ 21, 23, 30

We use the pre-filter file here so we can apply different age restrictions ourselves.

```
# āā Robustness: alternative age cutoffs āā# Use pre-filter data so we can apply our own age restrictions# Derive child_age in pre-filter datadf_prefilt['child_age'] = 2023 - df_prefilt['birth_year']# Restrict to White + Black (same as main analysis)df_pf_wb = df_prefilt[df_prefilt['race'].isin(['White', 'Black'])].copy()print("=" * 80)print("š ROBUSTNESS: ALTERNATIVE AGE CUTOFFS")print("=" * 80)age_cutoffs = [21, 23, 25, 30]results = []for age_cutoff in age_cutoffs:    robust_sample = df_pf_wb[        (df_pf_wb['child_age'] >= age_cutoff) &        (df_pf_wb['parent_homeowner'].notna()) &        (df_pf_wb['child_educ_yrs'].notna())    ]    if len(robust_sample) < 100:        print(f"\nā ļø  Age ā„ {age_cutoff}: Too few observations ({len(robust_sample)})")        continue    model = smf.ols('child_educ_yrs ~ parent_homeowner', data=robust_sample).fit()    coef = model.params['parent_homeowner']    pval = model.pvalues['parent_homeowner']    n = len(robust_sample)    results.append({        'Age Cutoff': age_cutoff,        'N': n,        'Coefficient': round(coef, 3),        'P-value': round(pval, 4)    })    sig = "***" if pval < 0.001 else "**" if pval < 0.01 else "*" if pval < 0.05 else "n.s."    marker = "ā­" if age_cutoff == 25 else "  "    print(f"\n{marker} Age ā„ {age_cutoff}:")    print(f"   N: {n:,}")    print(f"   Coefficient: {coef:.3f} years {sig}")# Summaryif results:    results_df = pd.DataFrame(results)    print("\n" + "=" * 80)    print("š SUMMARY:")    print(results_df.to_string(index=False))    print("\nš” If coefficients are similar across cutoffs ā Finding is ROBUST")    print("   ā­ = Main analysis (age ā„ 25)")print("\n" + "=" * 80)
```

```
================================================================================š ROBUSTNESS: ALTERNATIVE AGE CUTOFFS================================================================================   Age ā„ 21:   N: 3,521   Coefficient: 0.986 years ***   Age ā„ 23:   N: 3,521   Coefficient: 0.986 years ***ā­ Age ā„ 25:   N: 3,521   Coefficient: 0.986 years ***   Age ā„ 30:   N: 3,521   Coefficient: 0.986 years ***================================================================================š SUMMARY: Age Cutoff    N  Coefficient  P-value         21 3521        0.986      0.0         23 3521        0.986      0.0         25 3521        0.986      0.0         30 3521        0.986      0.0š” If coefficients are similar across cutoffs ā Finding is ROBUST   ā­ = Main analysis (age ā„ 25)================================================================================
```

## 5.2 Full Sample Including Hispanic & Other

Same robustness check as Notebook 02 Section 5. The main analysis uses only White and Black families because the PSID's 1968 sample was designed around these two groups. Here we check whether including all races changes the result.

```
# āā Full-sample robustness (all races) āāprint("=" * 80)print("š ROBUSTNESS: FULL SAMPLE (ALL RACES)")print("=" * 80)df_full = df.dropna(subset=['child_educ_yrs', 'parent_homeowner']).copy()print(f"\nFull sample (all races): {len(df_full):,}")print(f"Main sample (White+Black): {len(analysis_sample):,}")print(f"\nRace breakdown in full sample:")print(df_full['race'].value_counts())# Run bivariate model on full samplemodel_full = smf.ols('child_educ_yrs ~ parent_homeowner', data=df_full).fit()coef_full = model_full.params['parent_homeowner']pval_full = model_full.pvalues['parent_homeowner']sig = "***" if pval_full < 0.001 else "**" if pval_full < 0.01 else "*" if pval_full < 0.05 else "n.s."# Compare to White+Black onlymodel_wb = smf.ols('child_educ_yrs ~ parent_homeowner', data=analysis_sample).fit()coef_wb = model_wb.params['parent_homeowner']print(f"\nš COMPARISON:")print(f"  White+Black only:  {coef_wb:.3f} years")print(f"  All races:         {coef_full:.3f} years {sig}")print(f"  Difference:        {abs(coef_full - coef_wb):.3f} years")if abs(coef_full - coef_wb) < 0.2:    print("\nā Results are similar ā finding is robust to sample composition")else:    print("\nā ļø  Results differ ā sample composition matters for this estimate")print("\n" + "=" * 80)
```

```
================================================================================š ROBUSTNESS: FULL SAMPLE (ALL RACES)================================================================================Full sample (all races): 3,560Main sample (White+Black): 3,521Race breakdown in full sample:raceWhite       2195Black       1326Hispanic      26Other         11Name: count, dtype: int64š COMPARISON:  White+Black only:  0.986 years  All races:         1.010 years ***  Difference:        0.024 yearsā Results are similar ā finding is robust to sample composition================================================================================
```

# Summary: What We Learned

Run this cell after all analyses above have completed. It prints a consolidated summary.

```
print("=" * 80)print("š NOTEBOOK 03 ā EXPLORATION & ROBUSTNESS SUMMARY")print("=" * 80)print("""This notebook tested whether the main finding from Notebook 02 āthat parental homeownership in 1968 is associated with higher childeducation ā holds up under closer examination.Analyses performed:  1. Family structure (children per parent, by homeownership)  2. Subgroup analysis by race (White vs. Black)  3. Subgroup analysis by sex (Male vs. Female)  4. Cohort analysis by birth decade  5. Missing data patterns  6. Alternative age cutoffs (21, 23, 25, 30)  7. Full sample including Hispanic & OtherReview the output above for each section's results.Fill in the summary below after running all cells.""")print("=" * 80)print("\nFill in after reviewing results:")print("  Family structure:    ___")print("  Race subgroups:      ___")print("  Sex subgroups:       ___")print("  Cohort trends:       ___")print("  Missing data:        ___")print("  Age cutoff robust:   ___")print("  Full sample robust:  ___")print("\n" + "=" * 80)
```

```
================================================================================š NOTEBOOK 03 ā EXPLORATION & ROBUSTNESS SUMMARY================================================================================This notebook tested whether the main finding from Notebook 02 āthat parental homeownership in 1968 is associated with higher childeducation ā holds up under closer examination.Analyses performed:  1. Family structure (children per parent, by homeownership)  2. Subgroup analysis by race (White vs. Black)  3. Subgroup analysis by sex (Male vs. Female)  4. Cohort analysis by birth decade  5. Missing data patterns  6. Alternative age cutoffs (21, 23, 25, 30)  7. Full sample including Hispanic & OtherReview the output above for each section's results.Fill in the summary below after running all cells.================================================================================Fill in after reviewing results:  Family structure:    ___  Race subgroups:      ___  Sex subgroups:       ___  Cohort trends:       ___  Missing data:        ___  Age cutoff robust:   ___  Full sample robust:  ___================================================================================
```

## Files Generated
- 
family_size_distribution.png ā Family size histogram
- 
cohort_analysis.png ā Homeownership effect by birth decade

# End of Notebook 03

Status: ā Exploration & Robustness Complete Next: Proceed to Three Generation Family Explorer
