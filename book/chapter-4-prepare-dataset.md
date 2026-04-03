# Chapter 4: Prepare_Dataset


📊 PSID Data Preparation: From Raw Data to Analysis-Ready Dataset

This notebook takes us from raw PSID files to a clean, analysis-ready dataset that links parents (Generation 1, or "G1") to their children (Generation 2, or "G2"). The goal is to link each child back to their parent's 1968 homeownership status so we can ask: "Do children whose parents owned their home do better in school—and do they end up owning homes themselves?"

Key tasks:
• Use FIMS (Family Identification Mapping System) to build family trees
• Use the PSID Data Center extract for variables (homeownership, demographics, education, income, race)
• Combine via unique person IDs
• Recode raw codes into analysis-ready variables
• Apply sample and age restrictions

What you'll learn:
• How PSID person IDs work
• How to link generations using FIMS output
• How to merge parent traits (homeownership, education, income) with child traits (education, homeownership)
• How to recode PSID variables and handle missing data
• How to clean and validate your data

Prerequisites
1. 
FIMS Output File (CSV)

• Generated from PSID's online FIMS tool.
• Select "Inter-generational Prospective (GID PRO)" to link G1 → G2.
• This provides parent-child relationships. Save as CSV.
1. 
PSID Data Center Extract (CSV)

Variables required:

![]()
1. 
Google Drive Access (if using Colab)

• Files should be in your Google Drive, accessible from Colab.
• If not using Colab, ensure files are in a folder accessible by your Python environment.

Key Concepts

PSID Person IDs

PSID uses a two-part ID system:

• ER30001 : 1968 family number
• ER30002 : person number within that family

Combine into a single person_id:

person_id = (ER30001 * 1000) + ER30002

Examples:

• Family 2, Person 1 → 2001
• Family 345, Person 7 → 345007

Homeownership Variable (V103)

1968 homeownership codes:

• 1 = Owns or is buying home
• 5 = Rents
• 8 = Neither owns nor rents
• 0 or 9 = Missing / NA / DK

We recode to binary:

• 1 = Parent owned home in 1968 (V103 == 1)
• 0 = Parent did not own (V103 is 5 or 8)
• NaN = Missing data (codes 0, 9) — excluded from analysis

Note: The original PSID codebook uses 1 for "Owns or is buying," 5 for "Rents," and 8 for "Neither owns nor rents." Code 8 captures a small number of families who, for example, live with relatives rent-free. We group codes 5 and 8 together as "did not own."

Birth Year

ER30004 gives each person's age in 1968. We derive birth year as 1968 - ER30004. A value of 0 means age is unknown; we treat these as missing.

Part 1: Setup & File Loading

What we do: import Python libraries, mount Google Drive (if using Colab), and load our two data files.

Code: Import libraries and set paths

```
import pandas as pdimport numpy as npimport os
```

```
# ── Paths ──────────────────────────────────────────────────────────────────#BASE       = '/content/drive/MyDrive/PSID/PSID_2026/PSID_book_v2'BASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'DATA_DIR   = os.path.join(BASE, 'data')OUTPUT_DIR = os.path.join(BASE, 'outputs')PSID_FILE  = os.path.join(DATA_DIR, 'J358642.csv')#FIMS_FILE  = os.path.join(DATA_DIR, 'fim15712_gidpro_BO_2_BAL.xlsx')FIMS_FILE  = os.path.join(DATA_DIR, 'fim15712_gidpro_BO_2_BAL.csv')LABEL_FILE = os.path.join(DATA_DIR, 'J358642_labels.txt')# ── Output filename ────────────────────────────────────────────────────────OUT_FILE   = os.path.join(OUTPUT_DIR, 'psid_intergenerational_clean.csv')# Confirm paths existfor f in [PSID_FILE, FIMS_FILE]:    status = '✅ Found' if os.path.exists(f) else '❌ NOT FOUND'    print(f'{status}: {f}')
```

Update the BASE path to wherever you stored your files. If you are not using Colab, just make sure the paths point to the right folder.

Mount Google Drive (Colab only):

```
from google.colab import drivedrive.mount('/content/drive')
```

Load PSID extract:

```
# All variables needed for v2 pipeline# 'ER30003',   # age in 1968 (was ER30004 — not in this extractVARS_NEEDED = [    # Identification    'ER30001', 'ER30002',    # Demographics    'ER30004',   # age in 1968    'ER32006',   # sample status    'ER32000',   # sex    # Parent (G1) variables — 1968 Family File    'V103',      # parent homeownership    'V313',      # parent education    'V81',       # parent income 1968    'V181',      # race 1968    'V93',       # state 1968    'V119',      # sex of 1968 head    # Child (G2) outcome variables    'ER35152',   # child education years    'ER82032',   # child homeownership 2023     ]print('Loading PSID extract...')psid_raw = pd.read_csv(PSID_FILE, usecols=VARS_NEEDED, low_memory=False)print(f'Loaded: {psid_raw.shape[0]:,} rows × {psid_raw.shape[1]} columns')psid_raw.head()
```



We use usecols to load only the columns we need. The full extract (J358642) has 774 variables; we only need 14.

Load FIMS genealogy file:

```
print('Loading FIMS file...')fims_raw = pd.read_csv(FIMS_FILE)print(f'FIMS loaded: {fims_raw.shape[0]:,} rows x {fims_raw.shape[1]} columns')fims_raw.head()
```

Part 2: Create Unique Person IDs and Birth Year

PSID uses two columns for identification. We need a single person_id for merges, plus a derived birth_year so we can apply age restrictions later.

Create person_id and birth_year in PSID data:

```
psid = psid_raw.copy()# person_id: standard PSID constructionpsid['person_id'] = (psid['ER30001'] * 1000) + psid['ER30002']# birth_year derived from age in 1968# ER30004 = 0 means age unknown - treat as missingpsid['birth_year'] = np.where(psid['ER30004'] > 0,1968 - psid['ER30004'],np.nan)print(f'person_id created. Unique IDs: {psid["person_id"].nunique():,}')print(f'birth_year range: {psid["birth_year"].min():.0f} - 'f'{psid["birth_year"].max():.0f}')
```

Filter to sample members (ER32006):

The PSID tracks both original sample members and others who enter the study through marriage or other means. We keep only original sample members (codes 1–4).

```
# ER32006: 1=SRC original, 2=Census SEO, 3=Latino 1990, 4=Immigrant 1997VALID_SAMPLE = [1, 2, 3, 4]n_before = len(psid)psid = psid[psid['ER32006'].isin(VALID_SAMPLE)].copy()n_after = len(psid)print(f'Before sample filter: {n_before:,}')print(f'After sample filter: {n_after:,}')print(f'Dropped: {n_before - n_after:,}')
```

Create parent_id and child_id in FIMS:

```
fims = fims_raw.copy()# Construct parent_id and child_id using standard PSID constructionfims['parent_id'] = (fims['G1ID68'] * 1000) + fims['G1PN']fims['child_id'] = (fims['G2ID68'] * 1000) + fims['G2PN']# Keep only the linkage columnsfims = fims[['parent_id', 'child_id']].drop_duplicates()print(f'FIMS linkage pairs: {len(fims):,}')print(f'Unique parents:{fims["parent_id"].nunique():,}')print(f'Unique children:{fims["child_id"].nunique():,}')
```

Part 3: Merge Parent and Child Data

We now split the PSID extract into parent-level variables and child-level variables, then merge both onto the FIMS linkage map. This gives us one row per parent-child pair with all variables attached.

Separate parent and child variables:

```
# Parent vars: the 1968 family-level variablesPARENT_VARS = ['person_id', 'V103', 'V313', 'V81', 'V181', 'V93', 'V119']CHILD_VARS = ['person_id', 'birth_year', 'ER32000', 'ER35152', 'ER82032']psid_parents = psid[PARENT_VARS].copy().rename(columns={'person_id': 'parent_id'})psid_children = psid[CHILD_VARS].copy().rename(columns={'person_id': 'child_id'})print(f'Parent records: {len(psid_parents):,}')print(f'Child records: {len(psid_children):,}')
```

The parent variables (V103, V313, V81, V181, V93, V119) are all from the 1968 Family File. They
describe the head of household in 1968. The child variables (birth_year, ER32000, ER35152,
ER82032) describe the Generation 2 individual.

Merge onto FIMS links:

```
# Merge: FIMS -> parent vars -> child varsdf = fims.merge(psid_parents, on='parent_id', how='inner')print(f'After merging parent vars: {len(df):,}')df = df.merge(psid_children, on='child_id', how='inner')print(f'After merging child vars: {len(df):,}')
```

Part 4: Recode Variables

Raw PSID codes are not analysis-ready. We recode each variable into a form suitable for regression and descriptive statistics.

V103: Parent Homeownership → binary

```
# V103: 1=Owns, 5=Rents, 8=Neither; 0/9 = missingdf['parent_homeowner'] = np.where(df['V103'] == 1, 1,np.where(df['V103'].isin([5, 8]), 0, np.nan))print('parent_homeowner:')print(df['parent_homeowner'].value_counts(dropna=False))
```

This is the central variable in our analysis. A parent is coded as a homeowner (1) if V103 equals 1 ("Owns or is buying"). Parents who rent (code 5) or neither own nor rent (code 8) are coded as 0. Codes 0 and 9 are treated as missing.

V313: Parent Education → approximate years

```
ed_map = {0: 3,# <5th grade, difficulty reading1: 3,# <5th grade, no difficulty2: 7,# 6-8th grade3: 10, # 9-11th grade4: 12, # 12th grade / HS complete5: 13, # 12th + non-academic training6: 14, # College, no degree7: 16, # BA/BS8: 18, # Advanced/professional degree9: np.nan # NA/DK}df['parent_educ_yrs'] = df['V313'].map(ed_map)
```

V313 is a categorical variable (0–9). We convert it to approximate years of education so it can be
used as a continuous control variable in regression.

V81: Parent Income 1968

```
# Treat 0 and negative as missing; keep as continuousdf['parent_income_1968'] = np.where(df['V81'] > 0, df['V81'], np.nan)df['parent_income_1968_log'] = np.log(df['parent_income_1968'])
```

We create both a raw income variable and a log-transformed version. Income distributions are heavily right-skewed, so log income often performs better in regression.

V181: Race 1968

```
# 1=White, 2=Black, 3=Hispanic, 7=Other; 0/9 = missingrace_map = {1: 'White', 2: 'Black', 3: 'Hispanic', 7: 'Other'}df['race'] = df['V181'].map(race_map) # unmapped codes -> NaN
```

Other variables:

```
# V93: State 1968 - keep as numeric PSID state codedf['state_1968'] = np.where(df['V93'] > 0, df['V93'], np.nan)# V119: Sex of 1968 Headdf['head_sex_1968'] = df['V119']# 1=Male, 2=Female# ER32000: Child sexdf['child_sex'] = df['ER32000']# 1=Male, 2=Female
```

ER35152: Child Education Years

```
# 1-17 = actual years; 0 and 99 = missingdf['child_educ_yrs'] = np.where(df['ER35152'].isin([0, 99]),np.nan,df['ER35152'])
```

ER82032: Child Homeownership 2023 → binary

```
# 1=Owns, 5=Rents, 8=Neither; 0/9 = missingdf['child_homeowner_2023'] = np.where(df['ER82032'] == 1, 1,np.where(df['ER82032'].isin([5, 8]), 0, np.nan))
```

This follows the same logic as the parent homeownership recode: code 1 means owns, codes 5 and 8 mean does not own, and codes 0 and 9 are treated as missing.

Part 5: Age and Sample Restrictions

We restrict to children aged 25 or older in 2023, so their education is likely complete. We also drop rows with missing values on our key analysis variables.

```
# Restrict to children aged >= 25 in 2023# birth_year <= 1998 -> age >= 25 in 2023n_before = len(df)df = df[df['birth_year'] <= 1998].copy()n_after = len(df)print(f'Before age restriction (>=25 in 2023): {n_before:,}')print(f'After age restriction:{n_after:,}')print(f'Dropped: {n_before - n_after:,}')# Restrict to complete cases on key analysis variablesn_before = len(df)df_analysis = df[df['child_educ_yrs'].notna() &df['child_homeowner_2023'].notna() &df['parent_homeowner'].notna()].copy()n_after = len(df_analysis)print(f'Before outcome missingness drop: {n_before:,}')print(f'After outcome missingness drop: {n_after:,}')print(f'Dropped: {n_before - n_after:,}')print(f'\n--- FINAL ANALYTIC SAMPLE: {n_after:,} observations ---')
```



Part 6: Select and Label Final Columns

```
FINAL_COLS = [# IDs'child_id', 'parent_id',# Child demographics'birth_year', 'child_sex',# Child outcomes'child_educ_yrs', 'child_homeowner_2023',# Parent characteristics (G1)'parent_homeowner', 'parent_educ_yrs','parent_income_1968', 'parent_income_1968_log','race', 'state_1968', 'head_sex_1968',# Raw source vars retained for checking'V103', 'V313', 'V81', 'V181', 'V93', 'V119','ER35152', 'ER82032', 'ER32000']df_final = df_analysis[FINAL_COLS].copy()print(f'Final dataset: {df_final.shape[0]:,} rows x 'f'{df_final.shape[1]} columns')
```

We keep both the recoded analysis variables and the raw source variables. The raw variables let us double-check our recodes later if anything looks off.



Part 7: Descriptive Check

```
print('=== DESCRIPTIVE SUMMARY - KEY VARIABLES ===')key_vars = ['parent_homeowner', 'parent_educ_yrs', 'parent_income_1968','child_educ_yrs', 'child_homeowner_2023', 'birth_year']print(df_final[key_vars].describe().round(2))print('--- Parent homeownership rate ---')print(f"{df_final['parent_homeowner'].mean():.1%}")print('--- Child homeownership rate (2023) ---')print(f"{df_final['child_homeowner_2023'].mean():.1%}")print('--- Race distribution ---')print(df_final['race'].value_counts(dropna=False))
```

Always inspect your data before saving. Check that homeownership rates are plausible (roughly
60–65% of the 1968 sample were homeowners), that education years fall in a reasonable range, and that the race distribution matches what you know about the PSID sample.



Part 8: Export Prepared Dataset

```
os.makedirs(OUTPUT_DIR, exist_ok=True)df_final.to_csv(OUT_FILE, index=False)print(f'Saved: {OUT_FILE}')print(f'Rows: {len(df_final):,}')print(f'Columns: {df_final.shape[1]}')# Also save the pre-missingness-drop version for robustness checksout_full = OUT_FILE.replace('.csv', '_prefilt.csv')df[FINAL_COLS].to_csv(out_full, index=False)print(f'Saved (pre-filter): {out_full}')
```

We save two files: the main analysis file (complete cases only) and a pre-filter version that retains all rows before the missingness restriction. The pre-filter file is useful for robustness checks in later notebooks.

Output files:

• psid_intergenerational_clean.csv — main analysis dataset (complete cases)

 • psid_intergenerational_clean_prefilt.csv — pre-filter version (all rows, for robustness)



"Next up: the analysis itself—where three asterisks change everything."





