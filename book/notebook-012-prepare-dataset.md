# Notebook 01.2 — Prepare Dataset


# Notebook 01.2 — Prepare Dataset (v2)

Project: PSID Book v2
Purpose: Load and merge PSID extract + FIMS genealogy file; recode variables; apply sample restrictions; save clean intergenerational dataset.
Primary extract: J358642.csv (774 vars)
FIMS file: fim15712_gidpro_BO_2_BAL.xlsx
Last updated: 2026-03-09

### Variable Reference

## 0. Mount Google Drive

```
from google.colab import drivedrive.mount('/content/drive')
```

Output:

```
Drive already mounted at /content/drive; to attempt to forcibly remount, call drive.mount("/content/drive", force_remount=True).
```

## 1. Setup — Paths and Imports

```
import pandas as pdimport numpy as npimport os# ── Paths ──────────────────────────────────────────────────────────────────BASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'DATA_DIR   = os.path.join(BASE, 'data')OUTPUT_DIR = os.path.join(BASE, 'outputs')PSID_FILE  = os.path.join(DATA_DIR, 'J358642.csv')FIMS_FILE  = os.path.join(DATA_DIR, 'fim15712_gidpro_BO_2_BAL.csv')LABEL_FILE = os.path.join(DATA_DIR, 'J358642_labels.txt')# ── Output filename ────────────────────────────────────────────────────────OUT_FILE   = os.path.join(OUTPUT_DIR, 'psid_intergenerational_clean.csv')# Confirm paths existfor f in [PSID_FILE, FIMS_FILE]:    status = '✅ Found' if os.path.exists(f) else '❌ NOT FOUND'    print(f'{status}: {f}')
```

Output:

```
✅ Found: /content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2/data/J358642.csv✅ Found: /content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2/data/fim15712_gidpro_BO_2_BAL.csv
```

## 2. Load PSID Extract (J358642)

```
# All variables needed for v2 pipelineVARS_NEEDED = [    # Identification    'ER30001', 'ER30002',    # Demographics    'ER30004',   # age in 1968    'ER32006',   # sample status    'ER32000',   # sex    # Parent (G1) variables — 1968 Family File    'V103',      # parent homeownership    'V313',      # parent education    'V81',       # parent income 1968    'V181',      # race 1968    'V93',       # state 1968    'V119',      # sex of 1968 head    # Child (G2) outcome variables    'ER35152',   # child education years    'ER82032',   # child homeownership 2023    'ER30003',   # age in 1968]print('Loading PSID extract...')psid_raw = pd.read_csv(PSID_FILE, usecols=VARS_NEEDED, low_memory=False)print(f'Loaded: {psid_raw.shape[0]:,} rows × {psid_raw.shape[1]} columns')psid_raw.head()
```

Output:

```
Loading PSID extract...Loaded: 85,536 rows × 14 columns
```

```
# Diagnostic: check what's actually in the columnsprint("Shape:", psid.shape)print("\nFirst 20 columns:", psid.columns[:20].tolist())print("\nAll ER3xxxx columns:")er3_cols = [c for c in psid.columns if str(c).startswith('ER3')]print(er3_cols)# Check for ER32006 specifically with fuzzy matchmatches = [c for c in psid.columns if '32006' in str(c)]print("\nAnything containing '32006':", matches)
```

Output:

```
Shape: (54432, 16)First 20 columns: ['ER30001', 'ER30002', 'ER32000', 'ER32006', 'V81', 'V93', 'V103','V119', 'V181', 'V313', 'ER30003', 'ER30004', 'ER82032', 'ER35152', 'person_id', 'birth_year']All ER3xxxx columns:['ER30001', 'ER30002', 'ER32000', 'ER32006', 'ER30003', 'ER30004', 'ER35152']Anything containing '32006': ['ER32006']
```

## 3. Create person_id and birth_year

```
psid = psid_raw.copy()# person_id: standard PSID constructionpsid['person_id'] = (psid['ER30001'] * 1000) + psid['ER30002']# birth_year derived from age in 1968# ER30004 = 0 means age unknown — treat as missingpsid['birth_year'] = np.where(psid['ER30004'] > 0,                               1968 - psid['ER30004'],                               np.nan)print(f'person_id created. Unique IDs: {psid["person_id"].nunique():,}')print(f'birth_year range: {psid["birth_year"].min():.0f} – {psid["birth_year"].max():.0f}')print(f'birth_year missing: {psid["birth_year"].isna().sum():,}')
```

Output:

```
person_id created. Unique IDs: 85,536birth_year range: 969 – 1967birth_year missing: 67,306
```

## 4. Filter to Sample Members (ER32006)

```
# Keep only original PSID sample members# ER32006: 1=SRC original, 2=Census SEO, 3=Latino 1990, 4=Immigrant 1997VALID_SAMPLE = [1, 2, 3, 4]n_before = len(psid)psid = psid[psid['ER32006'].isin(VALID_SAMPLE)].copy()n_after = len(psid)print(f'Before sample filter: {n_before:,}')print(f'After sample filter:  {n_after:,}')print(f'Dropped: {n_before - n_after:,}')print(f'\nSample breakdown:\n{psid["ER32006"].value_counts().sort_index()}')
```

Output:

```
Before sample filter: 85,536After sample filter:  54,432Dropped: 31,104Sample breakdown:ER320061    301392    215053     26844      104Name: count, dtype: int64
```

## 5. Load and Merge FIMS Genealogy File

```
print('Loading FIMS file...')fims_raw = pd.read_csv(FIMS_FILE)print(f'FIMS loaded: {fims_raw.shape[0]:,} rows × {fims_raw.shape[1]} columns')print(f'FIMS columns: {list(fims_raw.columns)}')fims_raw.head()
```

Output:

```
Loading FIMS file...FIMS loaded: 58,585 rows × 8 columnsFIMS columns: ['G1ID68', 'G1PN', 'G1TYPE', 'G1POS', 'G2ID68', 'G2PN', 'G2TYPE', 'G2POS']
```

```
fims = fims_raw.copy()# Construct parent_id and child_id using standard PSID constructionfims['parent_id'] = (fims['G1ID68'] * 1000) + fims['G1PN']fims['child_id']  = (fims['G2ID68'] * 1000) + fims['G2PN']# Keep only the linkage columnsfims = fims[['parent_id', 'child_id']].drop_duplicates()print(f'FIMS linkage pairs: {len(fims):,}')print(f'Unique parents:     {fims["parent_id"].nunique():,}')print(f'Unique children:    {fims["child_id"].nunique():,}')
```

Output:

```
FIMS linkage pairs: 58,585Unique parents:     21,250Unique children:    47,414
```

```
# Separate parent-level and child-level data from PSIDPARENT_VARS = ['person_id', 'V103', 'V313', 'V81', 'V181', 'V93', 'V119']CHILD_VARS  = ['person_id', 'birth_year', 'ER32000', 'ER35152', 'ER82032']psid_parents  = psid[PARENT_VARS].copy().rename(columns={'person_id': 'parent_id'})psid_children = psid[CHILD_VARS].copy().rename(columns={'person_id': 'child_id'})print(f'Parent records: {len(psid_parents):,}')print(f'Child records:  {len(psid_children):,}')
```

Output:

```
Parent records: 54,432Child records:  54,432
```

```
# Merge: FIMS → parent vars → child varsdf = fims.merge(psid_parents, on='parent_id', how='inner')print(f'After merging parent vars: {len(df):,}')df = df.merge(psid_children, on='child_id', how='inner')print(f'After merging child vars:  {len(df):,}')df.head()
```

Output:

```
After merging parent vars: 57,982After merging child vars:  45,138
```

## 6. Recode Variables

```
# ── V103: Parent Homeownership → binary ───────────────────────────────────# 1=Owns, 5=Rents, 8=Neither; 0/9 = missingdf['parent_homeowner'] = np.where(df['V103'] == 1, 1,                          np.where(df['V103'].isin([5, 8]), 0, np.nan))print('parent_homeowner:')print(df['parent_homeowner'].value_counts(dropna=False))
```

Output:

```
parent_homeowner1.0    168310.0    16824NaN    11483Name: count, dtype: int64
```

```
# ── V313: Parent Education → approximate years ────────────────────────────ed_map = {    0: 3,   # <5th grade, difficulty reading    1: 3,   # <5th grade, no difficulty    2: 7,   # 6–8th grade    3: 10,  # 9–11th grade    4: 12,  # 12th grade / HS complete    5: 13,  # 12th + non-academic training    6: 14,  # College, no degree    7: 16,  # BA/BS    8: 18,  # Advanced/professional degree    9: np.nan  # NA/DK — drop}df['parent_educ_yrs'] = df['V313'].map(ed_map)print('parent_educ_yrs:')print(df['parent_educ_yrs'].value_counts(dropna=False).sort_index())
```

Output:

```
parent_educ_yrs3.0      42497.0      725110.0     797512.0     533913.0     266414.0     313416.0     185218.0      991NaN     11683Name: count, dtype: int64
```

```
# ── V81: Parent Income 1968 ───────────────────────────────────────────────# Treat 0 and negative as missing; keep as continuousdf['parent_income_1968'] = np.where(df['V81'] > 0, df['V81'], np.nan)df['parent_income_1968_log'] = np.log(df['parent_income_1968'])print('parent_income_1968 — descriptive stats:')print(df['parent_income_1968'].describe())
```

Output:

```
count    33648.000000mean      7600.013552std       5545.377111min        135.00000025%       3900.00000050%       6313.00000075%      10000.000000max      65400.000000Name: parent_income_1968, dtype: float64
```

```
# ── V181: Race 1968 → labeled categories ─────────────────────────────────# 1=White, 2=Black, 3=Hispanic, 7=Other; 0/9 = missingrace_map = {1: 'White', 2: 'Black', 3: 'Hispanic', 7: 'Other'}df['race'] = df['V181'].map(race_map)  # unmapped codes → NaNprint('race:')print(df['race'].value_counts(dropna=False))
```

Output:

```
raceWhite       17945Black       14892NaN         11546Hispanic      549Other         206Name: count, dtype: int64
```

```
# ── V93: State 1968 → keep as numeric PSID state code ────────────────────df['state_1968'] = np.where(df['V93'] > 0, df['V93'], np.nan)# ── V119: Sex of 1968 Head ────────────────────────────────────────────────df['head_sex_1968'] = df['V119']  # 1=Male, 2=Female# ── ER32000: Child sex ────────────────────────────────────────────────────df['child_sex'] = df['ER32000']  # 1=Male, 2=Femaleprint('state_1968 non-missing:', df['state_1968'].notna().sum())print('child_sex:')print(df['child_sex'].value_counts(dropna=False))
```

Output:

```
state_1968 non-missing: 33655child_sex:child_sex1    229112    22227Name: count, dtype: int64
```

```
# ── ER35152: Child Education Years ───────────────────────────────────────# 1–17 = actual years; 0 and 99 = missingdf['child_educ_yrs'] = np.where(df['ER35152'].isin([0, 99]),                                 np.nan,                                 df['ER35152'])print('child_educ_yrs — descriptive stats:')print(df['child_educ_yrs'].describe())
```

Output:

```
count    12484.000000mean        13.546940std          2.243231min          4.00000025%         12.00000050%         13.00000075%         16.000000max         17.000000Name: child_educ_yrs, dtype: float64
```

```
# ── ER82032: Child Homeownership 2023 → binary ───────────────────────────# 1=Owns, 5=Rents, 8=Neither; 0/9 = missingdf['child_homeowner_2023'] = np.where(df['ER82032'] == 1, 1,                              np.where(df['ER82032'].isin([5, 8]), 0, np.nan))print('child_homeowner_2023:')print(df['child_homeowner_2023'].value_counts(dropna=False))
```

Output:

```
child_homeowner_2023NaN    262891.0    105670.0     8282Name: count, dtype: int64
```

## 7. Age / Education / Sample Restrictions

```
# Restrict to children aged ≥25 in 2023# birth_year ≤ 1998 → age ≥ 25 in 2023n_before = len(df)df = df[df['birth_year'] <= 1998].copy()n_after = len(df)print(f'Before age restriction (≥25 in 2023): {n_before:,}')print(f'After age restriction:                {n_after:,}')print(f'Dropped: {n_before - n_after:,}')
```

Output:

```
Before age restriction (≥25 in 2023): 45,138After age restriction:                11,664Dropped: 33,474
```

```
# Restrict to children with non-missing education AND homeownership outcomen_before = len(df)df_analysis = df[    df['child_educ_yrs'].notna() &    df['child_homeowner_2023'].notna() &    df['parent_homeowner'].notna()].copy()n_after = len(df_analysis)print(f'Before outcome missingness drop: {n_before:,}')print(f'After outcome missingness drop:  {n_after:,}')print(f'Dropped: {n_before - n_after:,}')print(f'\n--- FINAL ANALYTIC SAMPLE: {n_after:,} observations ---')
```

Output:

```
Before outcome missingness drop: 11,664After outcome missingness drop:  3,560Dropped: 8,104--- FINAL ANALYTIC SAMPLE: 3,560 observations ---
```

## 8. Select and Label Final Columns

```
# Check what columns are actually in the filepsid_check = pd.read_csv(PSID_FILE, nrows=5)actual_cols = set(psid_check.columns.tolist())needed = ['ER30001', 'ER30002', 'ER30003', 'ER32006', 'ER32000',          'V103', 'V313', 'V81', 'V181', 'V93', 'V119',          'ER35152', 'ER82032']print('MISSING from file:')for v in needed:    if v not in actual_cols:        print(f'  ❌ {v}')print('\nPRESENT in file:')for v in needed:    if v in actual_cols:        print(f'  ✅ {v}')
```

Output:

```
MISSING from file:PRESENT in file:  ✅ ER30001    ✅ ER30002    ✅ ER30003  ✅ ER32006    ✅ ER32000    ✅ V103  ✅ V313       ✅ V81        ✅ V181  ✅ V93        ✅ V119       ✅ ER35152  ✅ ER82032
```

```
FINAL_COLS = [    # IDs    'child_id', 'parent_id',    # Child demographics    'birth_year', 'child_sex',    # Child outcomes    'child_educ_yrs', 'child_homeowner_2023',    # Parent characteristics (G1)    'parent_homeowner', 'parent_educ_yrs',    'parent_income_1968', 'parent_income_1968_log',    'race', 'state_1968', 'head_sex_1968',    # Raw source vars retained for checking    'V103', 'V313', 'V81', 'V181', 'V93', 'V119',    'ER35152', 'ER82032', 'ER32000']df_final = df_analysis[FINAL_COLS].copy()print(f'Final dataset: {df_final.shape[0]:,} rows × {df_final.shape[1]} columns')df_final.head()
```

Output:

```
Final dataset: 3,560 rows × 22 columns
```

## 9. Descriptive Check

```
print('=== DESCRIPTIVE SUMMARY — KEY VARIABLES ===')key_vars = [    'parent_homeowner', 'parent_educ_yrs', 'parent_income_1968',    'child_educ_yrs', 'child_homeowner_2023', 'birth_year']print(df_final[key_vars].describe().round(2))print('\n--- Parent homeownership rate ---')print(f"{df_final['parent_homeowner'].mean():.1%}")print('\n--- Child homeownership rate (2023) ---')print(f"{df_final['child_homeowner_2023'].mean():.1%}")print('\n--- Race distribution ---')print(df_final['race'].value_counts(dropna=False))
```

Output:

```
=== DESCRIPTIVE SUMMARY — KEY VARIABLES ===       parent_homeowner  parent_educ_yrs  parent_income_1968  child_educ_yrscount           3560.00          3528.00             3560.00         3560.00mean               0.62            10.65             9257.13           13.78std                0.48             3.96             6572.44            2.17min                0.00             3.00              275.00            4.0025%                0.00             7.00             4802.25           12.0050%                1.00            12.00             8075.00           14.0075%                1.00            13.00            12060.00           16.00max                1.00            18.00            65400.00           17.00       child_homeowner_2023  birth_yearcount               3560.00     3560.00mean                   0.73     1957.94std                    0.45        6.02min                    0.00     1934.0025%                    0.00     1953.0050%                    1.00     1958.0075%                    1.00     1963.00max                    1.00     1967.00--- Parent homeownership rate ---62.5%--- Child homeownership rate (2023) ---72.5%--- Race distribution ---raceWhite       2195Black       1326Hispanic      26Other         11NaN            2Name: count, dtype: int64
```

## 10. Save Output

```
os.makedirs(OUTPUT_DIR, exist_ok=True)df_final.to_csv(OUT_FILE, index=False)print(f'✅ Saved: {OUT_FILE}')print(f'   Rows: {len(df_final):,}')print(f'   Columns: {df_final.shape[1]}')# Also save the pre-missingness-drop version for robustness checksout_full = OUT_FILE.replace('.csv', '_prefilt.csv')df[FINAL_COLS].to_csv(out_full, index=False)print(f'✅ Saved (pre-filter): {out_full}')
```

Output:

```
✅ Saved: .../outputs/psid_intergenerational_clean.csv   Rows: 3,560   Columns: 22✅ Saved (pre-filter): .../outputs/psid_intergenerational_clean_prefilt.csv
```

## End of Notebook 01.2

Next: Notebook 02 — Main Analysis
- 
Baseline regression: child_educ_yrs ~ parent_homeowner + controls
- 
Homeownership outcome: child_homeowner_2023 ~ parent_homeowner + controls
- 
Tables and figures for book
