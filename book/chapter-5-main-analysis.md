# Chapter 5: Main Analysis


# 

# 📊 





PSID Main Analysis: Homeownership, Education, and Intergenerational Wealth

## What This Notebook Does

This notebook answers two central research questions:

Education: Do children whose parents owned their home in 1968 achieve higher educational attainment compared to children whose parents rented?

Homeownership transmission: Are children of 1968 homeowners more likely to own a home themselves in 2023?

The short answer to both: Yes. And the effects persist even after controlling for parent education, parent income, race, and sex -- and even after adding state fixed effects.

## The Journey

Data loading & sample construction
- 
Load the prepared dataset from Notebook 01.2 (psid_intergenerational_clean.csv)
- 
Create a White/Black subsample for the main analysis (Hispanic and Other subsamples are too small for reliable inference in the 1968 PSID)

Descriptive statistics
- 
Summary statistics (Table 1)
- 
Means by parent homeownership status (Table 1b)
- 
Homeownership rate by race (crosstab)
- 
Homeownership transition matrix: did parents' ownership predict children's ownership?

Education regressions (Models 1-3)
- 
Model 1: Bivariate -- homeownership only
- 
Model 2: With controls (parent education, parent income, race, sex)
- 
Model 3: With controls + state fixed effects

Homeownership regressions (Models 4-6)
- 
Model 4: Bivariate -- homeownership only
- 
Model 5: With controls
- 
Model 6: With controls + state fixed effects

Figures
- 
Figure 1: Child homeownership rate by parent status
- 
Figure 2: Child homeownership by parent status and race
- 
Figure 3: Coefficient plots (education and homeownership side by side)
- 
Figure 4: Mean child education by parent homeownership and race

Robustness
- 
Re-run key models on full sample (all races) to confirm results hold

## Research Question Breakdown

Independent variable: Parent homeownership status in 1968 (parent_homeowner: 1 = owned, 0 = rented/neither)

Dependent variables:
- 
child_educ_yrs: Child's years of completed education (measured as adult)
- 
child_homeowner_2023: Whether the child owns a home in 2023 (1 = yes, 0 = no)

Control variables:
- 
parent_educ_yrs: Parent education (approximate years, derived from V313)
- 
parent_income_1968_log: Log of parent income in 1968
- 
race: Child's race (White or Black -- reference group is White)
- 
child_sex: Child's sex (1 = Male reference, 2 = Female)
- 
state_1968: State of residence in 1968 (used as fixed effects in Models 3 and 6)

## Statistical Approach

We use Ordinary Least Squares (OLS) for all models. For the homeownership outcome (Models 4-6), this is called a Linear Probability Model (LPM) -- same math as OLS, but with a 0/1 outcome. Coefficients represent changes in probability (percentage points).

Reference group for all models: White male whose parents rented in 1968. The intercept represents the predicted outcome for this reference person.

Key concepts (plain language)

Regression coefficient: The estimated difference in the outcome between two otherwise identical children whose parents differ only by homeownership status.

Controlling for variables: Comparing children with similar characteristics (same race, sex, parent education, parent income) to isolate the effect of homeownership.

State fixed effects: Including a separate adjustment for each state. This accounts for the fact that states differ in school quality, housing costs, and other factors that might affect both homeownership and education.

Significance stars:
- 
*** = p < 0.001 (less than 1 in 1,000 chance this is random)
- 
** = p < 0.01 (less than 1 in 100 chance)
- 
* = p < 0.05 (less than 1 in 20 chance)

## Part 1: Setup & Data Loading

What we do: mount Google Drive, import libraries, load the prepared dataset from Notebook 01.2, and verify its structure.

### 1.1 Mount Google Drive and Import Libraries

```
from google.colab import drivedrive.mount('/content/drive')import pandas as pdimport numpy as npimport statsmodels.api as smimport statsmodels.formula.api as smfimport matplotlib.pyplot as pltimport matplotlib.ticker as mtickimport seaborn as snsfrom IPython.display import display, HTMLimport os# ---- Style ----sns.set_style('whitegrid')plt.rcParams.update({    'figure.figsize': (10, 6),    'font.size': 12,    'axes.titlesize': 14,    'axes.labelsize': 12})
```

We use statsmodels.formula.api (imported as smf) for regression -- it lets us write models in R-style formula syntax like child_educ_yrs ~ parent_homeowner + race. We use seaborn and matplotlib for publication-quality figures.

### 1.2 Set Paths and Load Data

```
# ---- Paths ----BASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'DATA_DIR = f'{BASE}/data'OUTPUT_DIR = f'{BASE}/outputs'INPUT_FILE = f'{OUTPUT_DIR}/psid_intergenerational_clean.csv'
```

Important: The input file is psid_intergenerational_clean.csv -- this is the output from Notebook 01.2. It contains one row per parent-child pair with all variables already recoded: parent_homeowner (binary), parent_educ_yrs (continuous), parent_income_1968 and its log transform, race (string labels: 'White', 'Black', 'Hispanic', 'Other'), child_educ_yrs, child_homeowner_2023, and more.

```
df = pd.read_csv(INPUT_FILE)print(f'Dataset shape: {df.shape}')print(f'\nColumns: {list(df.columns)}')df.head()
```

### 1.3 Variable Prep and Initial Checks

```
# ---- Variable prep ----# Sex label for readabilitydf['sex_label'] = df['child_sex'].map({1: 'Male', 2: 'Female'})# Confirm race is stringprint('=== Race distribution ===')print(df['race'].value_counts())print(f'\n=== Sex distribution ===')print(df['sex_label'].value_counts())print(f'\n=== Parent homeowner ===')print(df['parent_homeowner'].value_counts())print(f'\n=== Missing values ===')print(df.isnull().sum())
```

Note that race is already a string variable ('White', 'Black', etc.) -- Notebook 01.2 converted the numeric V181 codes into labels. And parent_homeowner is already binary (1 = owned, 0 = rented/neither) -- there is no need to recode V103 here; that was done in Notebook 01.2.

### 1.4 Create White/Black Subsample

```
# ---- Create White/Black-only subsample for main analysis ----# Hispanic (<20) and Other (<20) subsamples are too small for reliable inferencedf_wb = df[df['race'].isin(['White', 'Black'])].copy()print(f'Full sample: {len(df):,} observations')print(f'White + Black only: {len(df_wb):,} observations')print(f'Dropped (Hispanic/Other): {len(df) - len(df_wb):,} observations')print(f'\nWhite/Black subsample race counts:')print(df_wb['race'].value_counts())
```

The PSID's original 1968 sample was designed around White and Black families. The Hispanic and Other subsamples have fewer than 40 observations each in our data -- too few for the regression coefficients to be reliable. We run the main analysis on White and Black families only, then check robustness with the full sample in Section 5.

## Part 2: Sample Description (Table 1)

Before running regressions, we describe the data. This is Table 1 in any empirical paper -- it tells the reader who's in the sample and what the key variables look like.

### 2.1 Summary Statistics

```
# ---- Table 1: Summary Statistics (White + Black sample) ----summary_vars = ['parent_homeowner', 'parent_educ_yrs', 'parent_income_1968',                'parent_income_1968_log', 'child_educ_yrs', 'child_homeowner_2023']table1 = df_wb[summary_vars].describe().T[['count', 'mean', 'std', 'min', 'max']]table1.columns = ['N', 'Mean', 'Std Dev', 'Min', 'Max']table1['N'] = table1['N'].astype(int)# Rename for readabilitytable1.index = ['Parent Homeowner (1=Yes)', 'Parent Education (years)',                'Parent Income (1968 $)', 'Log Parent Income',                'Child Education (years)', 'Child Homeowner 2023 (1=Yes)']print('=== TABLE 1: Summary Statistics (White & Black Families) ===')display(table1.round(3))table1.round(3).to_csv(f'{OUTPUT_DIR}/table1_summary_stats.csv')
```

### 2.2 Means by Parent Homeownership Status

```
# ---- Table 1b: Summary Stats by Parent Homeownership ----table1b = df_wb.groupby('parent_homeowner')[summary_vars].mean()table1b.index = ['Parent: Renter', 'Parent: Owner']table1b.columns = ['Homeowner (mean)', 'Parent Educ (yrs)', 'Parent Income ($)',                    'Log Income', 'Child Educ (yrs)', 'Child Homeowner Rate']print('=== TABLE 1b: Means by Parent Homeownership Status ===')display(table1b.round(3))
```

This table is the heart of the descriptive story. Look at the last two columns: do children of owners have more education and higher homeownership rates than children of renters? The raw means tell you "yes" -- but they don't account for the fact that homeowners also tend to be wealthier and more educated. That's what the regressions in Parts 3 and 4 are for.

### 2.3 Homeownership Rate by Race

```
ct_race = pd.crosstab(df_wb['race'], df_wb['parent_homeowner'], normalize='index')ct_race.columns = ['Renter Rate', 'Homeowner Rate']print('=== Parent Homeownership Rate by Race ===')display(ct_race.round(3))
```

### 2.4 Homeownership Transition Matrix

```
# ---- Homeownership Transition Matrix ----ct_hw = pd.crosstab(df_wb['parent_homeowner'], df_wb['child_homeowner_2023'], margins=True)ct_hw.index = ['Parent: Renter', 'Parent: Owner', 'Total']ct_hw.columns = ['Child: Renter', 'Child: Owner', 'Total']print('=== Homeownership Transition Matrix (Counts) ===')display(ct_hw)ct_hw_pct = pd.crosstab(df_wb['parent_homeowner'], df_wb['child_homeowner_2023'], normalize='index')ct_hw_pct.index = ['Parent: Renter', 'Parent: Owner']ct_hw_pct.columns = ['Child: Renter', 'Child: Owner']print('\n=== Homeownership Transition Matrix (Row %) ===')display(ct_hw_pct.round(3))
```

The transition matrix shows the probability of a child owning a home in 2023 given their parent's 1968 status. Read the rows: what fraction of renters' children became owners? What fraction of owners' children became owners? The difference is the raw intergenerational transmission of homeownership.

## Part 3: Child Education Regressions (OLS)

Reference group for all models: White male whose parents rented in 1968. The intercept represents the predicted education for this reference person.

### 3.1 Prepare Education Analysis Sample

```
educ_vars = ['child_educ_yrs', 'parent_homeowner', 'parent_educ_yrs',             'parent_income_1968_log', 'race', 'child_sex']df_educ = df_wb.dropna(subset=educ_vars).copy()print(f'Education analysis sample: {len(df_educ):,} obs')print(f'  (dropped {len(df_wb) - len(df_educ):,} with missing values)')
```

We drop rows with missing values on any variable used in the regressions. This ensures all three education models are estimated on the same sample, making coefficients directly comparable.

### 3.2 Model 1: Bivariate (Homeownership Only)

```
model1 = smf.ols('child_educ_yrs ~ parent_homeowner', data=df_educ).fit()print('=== MODEL 1: Bivariate ===')print(model1.summary2().tables[1].to_string())print(f'\nR2 = {model1.rsquared:.4f}   N = {int(model1.nobs)}')
```

How to read this: The intercept is the average education for children of renters. The parent_homeowner coefficient is how many additional years of education children of owners complete. This is the raw, unadjusted gap -- it doesn't account for the fact that homeowners differ from renters in many ways.

### 3.3 Model 2: With Controls

```
model2 = smf.ols(    'child_educ_yrs ~ parent_homeowner + parent_educ_yrs + parent_income_1968_log '    '+ C(race, Treatment(reference="White")) + C(child_sex, Treatment(reference=1))',    data=df_educ).fit()print('=== MODEL 2: With Controls ===')print(model2.summary2().tables[1].to_string())print(f'\nR2 = {model2.rsquared:.4f}   Adj R2 = {model2.rsquared_adj:.4f}   N = {int(model2.nobs)}')
```

What the formula means:

C(race, Treatment(reference="White")) tells statsmodels to treat race as a categorical variable with White as the reference group. The output will show a coefficient for Black (vs. White).

C(child_sex, Treatment(reference=1)) uses Male (code 1) as the reference. The output shows a coefficient for Female (vs. Male).

parent_educ_yrs and parent_income_1968_log enter as continuous controls.

How to read this: The parent_homeowner coefficient now answers: comparing two children who are the same race, same sex, whose parents have the same education and income -- does the child of the homeowner still have more education? If this coefficient is still positive and significant, homeownership has an effect beyond simply reflecting that homeowners are richer and more educated.

### 3.4 Model 3: With Controls + State Fixed Effects

```
model3 = smf.ols(    'child_educ_yrs ~ parent_homeowner + parent_educ_yrs + parent_income_1968_log '    '+ C(race, Treatment(reference="White")) + C(child_sex, Treatment(reference=1)) '    '+ C(state_1968)',    data=df_educ).fit()print('=== MODEL 3: With Controls + State FE ===')non_state = [v for v in model3.params.index if 'state' not in v]print(model3.summary2().tables[1].loc[non_state].to_string())print(f'\n(State fixed effects suppressed -- {int(df_educ["state_1968"].nunique())} states)')print(f'R2 = {model3.rsquared:.4f}   Adj R2 = {model3.rsquared_adj:.4f}   N = {int(model3.nobs)}')
```

Why state fixed effects? States differ in school quality, housing costs, and labor markets. By including C(state_1968), we effectively compare families within the same state. This removes state-level confounds. The state coefficients themselves are not interesting -- we suppress them in the output and just report the key variables.

### 3.5 Plain English Education Results

```
rename_educ = {    'Intercept': 'Intercept (White Male, Parents Rented)',    'parent_homeowner': 'Parent Owned Home (vs Rented)',    'parent_educ_yrs': 'Parent Education (per year)',    'parent_income_1968_log': 'Parent Income (log, 1968)',    'C(race, Treatment(reference="White"))[T.Black]': 'Black (vs White)',    'C(child_sex, Treatment(reference=1))[T.2]': 'Female (vs Male)',}def make_plain_table(model, rename_map, model_label):    """Create a plain-English regression summary table."""    params = model.params    se = model.bse    pval = model.pvalues    ci = model.conf_int()    rows = []    for var, label in rename_map.items():        if var in params.index:            stars = '***' if pval[var] < 0.001 else '**' if pval[var] < 0.01 else '*' if pval[var] < 0.05 else ''            rows.append({                'Variable': label,                'Coefficient': f'{params[var]:+.3f}{stars}',                'Std Error': f'{se[var]:.3f}',                'p-value': f'{pval[var]:.4f}',                '95% CI': f'[{ci.loc[var, 0]:.3f}, {ci.loc[var, 1]:.3f}]'            })    tbl = pd.DataFrame(rows)    stats = pd.DataFrame([{        'Variable': 'R-squared',        'Coefficient': f'{model.rsquared:.4f}',        'Std Error': '', 'p-value': '',        '95% CI': f'N = {int(model.nobs):,}'    }])    tbl = pd.concat([tbl, stats], ignore_index=True)    tbl = tbl.set_index('Variable')    return tbl
```

This helper function translates the cryptic statsmodels variable names (like C(race, Treatment(reference="White"))[T.Black]) into plain English labels (like "Black (vs White)"). It also adds significance stars and confidence intervals. We'll reuse it for the homeownership models too.

### 3.6 Save Education Regression Table

```
plain_m2 = make_plain_table(model2, rename_educ, 'M2')plain_m2.to_csv(f'{OUTPUT_DIR}/table2_education_regression_plain.csv')
```

## Part 4: Homeownership Transmission (Linear Probability Model)

LPM = Linear Probability Model. Same math as OLS, but with a 0/1 outcome. Coefficients represent the change in probability (in percentage points) of a child owning a home in 2023.

Reference group: White male whose parents rented in 1968.

### 4.1 Prepare Homeownership Analysis Sample

```
homeown_vars = ['child_homeowner_2023', 'parent_homeowner', 'parent_educ_yrs',                'parent_income_1968_log', 'race', 'child_sex']df_homeown = df_wb.dropna(subset=homeown_vars).copy()print(f'Homeownership analysis sample: {len(df_homeown):,} obs')print(f'  (dropped {len(df_wb) - len(df_homeown):,} with missing values)')print(f'\nOverall child homeownership rate: {df_homeown["child_homeowner_2023"].mean():.1%}')
```

### 4.2 Model 4: Bivariate LPM

```
model4 = smf.ols('child_homeowner_2023 ~ parent_homeowner', data=df_homeown).fit()print('=== MODEL 4: Bivariate LPM ===')print(model4.summary2().tables[1].to_string())print(f'\nR2 = {model4.rsquared:.4f}   N = {int(model4.nobs)}')
```

How to read this: The intercept is the homeownership rate for children of renters. The parent_homeowner coefficient is the additional probability (in percentage points) for children of owners. For example, a coefficient of 0.15 means owners' children are 15 percentage points more likely to own a home.

### 4.3 Model 5: LPM With Controls

```
model5 = smf.ols(    'child_homeowner_2023 ~ parent_homeowner + parent_educ_yrs + parent_income_1968_log '    '+ C(race, Treatment(reference="White")) + C(child_sex, Treatment(reference=1))',    data=df_homeown).fit()print('=== MODEL 5: LPM With Controls ===')print(model5.summary2().tables[1].to_string())print(f'\nR2 = {model5.rsquared:.4f}   Adj R2 = {model5.rsquared_adj:.4f}   N = {int(model5.nobs)}')
```

### 4.4 Model 6: LPM With Controls + State Fixed Effects

```
model6 = smf.ols(    'child_homeowner_2023 ~ parent_homeowner + parent_educ_yrs + parent_income_1968_log '    '+ C(race, Treatment(reference="White")) + C(child_sex, Treatment(reference=1)) '    '+ C(state_1968)',    data=df_homeown).fit()print('=== MODEL 6: LPM With Controls + State FE ===')non_state_hw = [v for v in model6.params.index if 'state' not in v]print(model6.summary2().tables[1].loc[non_state_hw].to_string())print(f'\n(State fixed effects suppressed -- {int(df_homeown["state_1968"].nunique())} states)')print(f'R2 = {model6.rsquared:.4f}   Adj R2 = {model6.rsquared_adj:.4f}   N = {int(model6.nobs)}')
```

### 4.5 Plain English Homeownership Results

```
rename_homeown = {    'Intercept': 'Intercept (White Male, Parents Rented)',    'parent_homeowner': 'Parent Owned Home (vs Rented)',    'parent_educ_yrs': 'Parent Education (per year)',    'parent_income_1968_log': 'Parent Income (log, 1968)',    'C(race, Treatment(reference="White"))[T.Black]': 'Black (vs White)',    'C(child_sex, Treatment(reference=1))[T.2]': 'Female (vs Male)',}
```

### 4.6 Save Homeownership Regression Table

```
plain_m5 = make_plain_table(model5, rename_homeown, 'M5')plain_m5.to_csv(f'{OUTPUT_DIR}/table3_homeownership_regression_plain.csv')
```

## Part 5: Figures

### 5.1 Figure 1: Child Homeownership Rate by Parent Status

```
fig1, ax1 = plt.subplots(figsize=(8, 5))hw_rates = df_homeown.groupby('parent_homeowner')['child_homeowner_2023'].mean()bars = ax1.bar(['Parent: Renter', 'Parent: Owner'], hw_rates.values,               color=['#c44e52', '#4c72b0'], width=0.5, edgecolor='white')for bar, val in zip(bars, hw_rates.values):    ax1.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.01,             f'{val:.1%}', ha='center', va='bottom', fontsize=13, fontweight='bold')ax1.set_ylabel('Child Homeownership Rate (2023)')ax1.set_title('Child Homeownership by Parent Homeownership Status (1968)\nWhite & Black Families')ax1.yaxis.set_major_formatter(mtick.PercentFormatter(1.0))ax1.set_ylim(0, 1)ax1.spines['top'].set_visible(False)ax1.spines['right'].set_visible(False)plt.tight_layout()fig1.savefig(f'{OUTPUT_DIR}/fig1_homeownership_by_parent_status.png', dpi=300, bbox_inches='tight')plt.show()
```

### 5.2 Figure 2: Child Homeownership by Parent Status AND Race

```
fig2, ax2 = plt.subplots(figsize=(10, 6))hw_by_race = df_homeown.groupby(['race', 'parent_homeowner'])['child_homeowner_2023'].mean().unstack()hw_by_race.columns = ['Parent: Renter', 'Parent: Owner']race_order = [r for r in ['White', 'Black'] if r in hw_by_race.index]hw_by_race = hw_by_race.loc[race_order]hw_by_race.plot(kind='bar', ax=ax2, color=['#c44e52', '#4c72b0'],                width=0.6, edgecolor='white')for container in ax2.containers:    labels = [f'{v.get_height():.1%}' for v in container]    ax2.bar_label(container, labels=labels, label_type='edge', fontsize=11, padding=3)ax2.set_ylabel('Child Homeownership Rate (2023)')ax2.set_title('Child Homeownership by Parent Status and Race')ax2.yaxis.set_major_formatter(mtick.PercentFormatter(1.0))ax2.set_ylim(0, 1.05)ax2.set_xticklabels(ax2.get_xticklabels(), rotation=0)ax2.legend(title='Parent Status (1968)', loc='upper right')ax2.spines['top'].set_visible(False)ax2.spines['right'].set_visible(False)plt.tight_layout()fig2.savefig(f'{OUTPUT_DIR}/fig2_homeownership_by_race.png', dpi=300, bbox_inches='tight')plt.show()
```

### 5.3 Figure 3: Coefficient Plots (Education and Homeownership)

```
fig3, axes = plt.subplots(1, 2, figsize=(14, 5))plot_vars = ['parent_homeowner', 'parent_educ_yrs', 'parent_income_1968_log',             'C(race, Treatment(reference="White"))[T.Black]',             'C(child_sex, Treatment(reference=1))[T.2]']plot_labels = ['Parent Homeowner\n(vs Rented)', 'Parent Education\n(per year)',               'Log Parent Income', 'Black\n(vs White)', 'Female\n(vs Male)']# --- Panel A: Education ---for i, (model, color, label) in enumerate([    (model2, '#4c72b0', 'M2: Controls'),    (model3, '#55a868', 'M3: + State FE')]):    available = [v for v in plot_vars if v in model.params.index]    avail_labels = [plot_labels[plot_vars.index(v)] for v in available]    coefs = model.params[available]    cis = model.conf_int().loc[available]    y_pos = np.arange(len(available)) + i * 0.15    axes[0].errorbar(coefs, y_pos, xerr=[coefs - cis[0], cis[1] - coefs],                     fmt='o', color=color, label=label, capsize=3, markersize=6)axes[0].axvline(0, color='gray', linestyle='--', alpha=0.5)axes[0].set_yticks(np.arange(len(available)) + 0.075)axes[0].set_yticklabels(avail_labels)axes[0].set_xlabel('Effect on Education (years)')axes[0].set_title('Panel A: Child Education')axes[0].legend(loc='lower right', fontsize=9)axes[0].invert_yaxis()# --- Panel B: Homeownership ---for i, (model, color, label) in enumerate([    (model5, '#4c72b0', 'M5: Controls'),    (model6, '#55a868', 'M6: + State FE')]):    available = [v for v in plot_vars if v in model.params.index]    avail_labels = [plot_labels[plot_vars.index(v)] for v in available]    coefs = model.params[available]    cis = model.conf_int().loc[available]    y_pos = np.arange(len(available)) + i * 0.15    axes[1].errorbar(coefs, y_pos, xerr=[coefs - cis[0], cis[1] - coefs],                     fmt='o', color=color, label=label, capsize=3, markersize=6)axes[1].axvline(0, color='gray', linestyle='--', alpha=0.5)axes[1].set_yticks(np.arange(len(available)) + 0.075)axes[1].set_yticklabels(avail_labels)axes[1].set_xlabel('Effect on Homeownership (percentage points)')axes[1].set_title('Panel B: Child Homeownership')axes[1].legend(loc='lower right', fontsize=9)axes[1].invert_yaxis()plt.suptitle('Figure 3: What Predicts Child Outcomes?', y=1.02, fontsize=14)plt.tight_layout()fig3.savefig(f'{OUTPUT_DIR}/fig3_coefficient_plots.png', dpi=300, bbox_inches='tight')plt.show()
```

This is the most important figure in the notebook. Each dot is a regression coefficient; the horizontal lines are 95% confidence intervals. If a confidence interval crosses zero (the dashed line), the effect is not statistically significant. Panel A shows education effects; Panel B shows homeownership effects. Models with controls (blue) and with state fixed effects (green) are overlaid so you can see how robust the estimates are.

### 5.4 Figure 4: Mean Child Education by Parent Homeownership and Race

```
fig4, ax4 = plt.subplots(figsize=(10, 6))educ_by_race = df_educ.groupby(['race', 'parent_homeowner'])['child_educ_yrs'].mean().unstack()educ_by_race.columns = ['Parent: Renter', 'Parent: Owner']educ_by_race = educ_by_race.loc[[r for r in ['White', 'Black'] if r in educ_by_race.index]]educ_by_race.plot(kind='bar', ax=ax4, color=['#c44e52', '#4c72b0'],                  width=0.6, edgecolor='white')for container in ax4.containers:    ax4.bar_label(container, fmt='%.1f', label_type='edge', fontsize=11, padding=3)ax4.set_ylabel('Mean Years of Education (Child)')ax4.set_title('Child Educational Attainment by Parent Homeownership and Race')ax4.set_xticklabels(ax4.get_xticklabels(), rotation=0)ax4.legend(title='Parent Status (1968)', loc='upper right')ax4.spines['top'].set_visible(False)ax4.spines['right'].set_visible(False)plt.tight_layout()fig4.savefig(f'{OUTPUT_DIR}/fig4_education_by_race.png', dpi=300, bbox_inches='tight')plt.show()
```

## Part 6: Robustness -- Full Sample Including Hispanic & Other

The main analysis above uses only White and Black families because the PSID's 1968 sample was designed around these two groups. Hispanic and Other subsamples have fewer than 40 observations each, making their coefficients unreliable.

This section re-runs the key models on the full sample to confirm that results are substantively similar. These can be cited in a footnote.

```
df_full = df.dropna(subset=educ_vars).copy()print(f'Full sample (all races): {len(df_full):,} obs')print(f'Race distribution:')print(df_full['race'].value_counts())# Education model with controlsmodel2_full = smf.ols(    'child_educ_yrs ~ parent_homeowner + parent_educ_yrs + parent_income_1968_log '    '+ C(race, Treatment(reference="White")) + C(child_sex, Treatment(reference=1))',    data=df_full).fit()# Homeownership model with controlsdf_full_hw = df.dropna(subset=homeown_vars).copy()model5_full = smf.ols(    'child_homeowner_2023 ~ parent_homeowner + parent_educ_yrs + parent_income_1968_log '    '+ C(race, Treatment(reference="White")) + C(child_sex, Treatment(reference=1))',    data=df_full_hw).fit()print('\n=== ROBUSTNESS: Education Model (Full Sample) ===')print(f'Parent homeowner coeff: {model2_full.params["parent_homeowner"]:+.3f}')print(f'  (Main analysis: {model2.params["parent_homeowner"]:+.3f})')print(f'\n=== ROBUSTNESS: Homeownership Model (Full Sample) ===')print(f'Parent homeowner coeff: {model5_full.params["parent_homeowner"]:+.3f}')print(f'  (Main analysis: {model5.params["parent_homeowner"]:+.3f})')print('\n-> If coefficients are similar, this confirms that dropping Hispanic/Other')print('  does not materially change the findings.')
```

## Part 7: Output Summary

```
print('=' * 60)print('NOTEBOOK 02 -- OUTPUT SUMMARY')print('=' * 60)print(f'\nSample: White & Black families only (N = {len(df_wb):,})')print(f'\nTables saved to {OUTPUT_DIR}/:')print(f'  table1_summary_stats.csv')print(f'  table2_education_regression_plain.csv')print(f'  table3_homeownership_regression_plain.csv')print(f'\nFigures saved to {OUTPUT_DIR}/:')print(f'  fig1_homeownership_by_parent_status.png')print(f'  fig2_homeownership_by_race.png')print(f'  fig3_coefficient_plots.png')print(f'  fig4_education_by_race.png')print('\n--- KEY FINDINGS ---')print(f'Education sample: {int(model1.nobs):,} | Homeownership sample: {int(model4.nobs):,}')print(f'\nEDUCATION (child years of schooling):')print(f'  Raw gap (owners vs renters):        {model1.params["parent_homeowner"]:+.2f} years')print(f'  After controls:                     {model2.params["parent_homeowner"]:+.2f} years')print(f'  After controls + state FE:          {model3.params["parent_homeowner"]:+.2f} years')print(f'\nHOMEOWNERSHIP (prob. child owns in 2023):')print(f'  Raw gap (owners vs renters):        {model4.params["parent_homeowner"]:+.2f} pp')print(f'  After controls:                     {model5.params["parent_homeowner"]:+.2f} pp')print(f'  After controls + state FE:          {model6.params["parent_homeowner"]:+.2f} pp')
```

Output files:
- 
table1_summary_stats.csv -- descriptive statistics
- 
table2_education_regression_plain.csv -- education regression results (plain English)
- 
table3_homeownership_regression_plain.csv -- homeownership regression results (plain English)
- 
fig1_homeownership_by_parent_status.png -- bar chart of child homeownership rates
- 
fig2_homeownership_by_race.png -- bar chart by race
- 
fig3_coefficient_plots.png -- coefficient plots for both outcomes
- 
fig4_education_by_race.png -- bar chart of education by race and parent status

"Next up: we stress-test these findings -- what happens when we push the data harder?"


