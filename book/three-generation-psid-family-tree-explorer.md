# 🌳 Three-Generation PSID Family Tree Explorer


# 🌳 Three-Generation PSID Family Tree Explorer

Longitudinal Study: 1968-Present

Explore how 1968 homeownership affected educational outcomes measured decades later:
- 
Generation 1 (Blue/Purple): Household heads in 1968 - Homeownership status
- 
Generation 2 (Colored): Their children - Education measured as adults
- 
Generation 3 (Colored): Grandchildren - Education measured as adults (1990s-2020s)

Color = Education Level | Shape = Homeownership

This shows the long-term intergenerational effects of homeownership!

### Data Requirements

This notebook builds its three-generation dataset from two files produced by the earlier notebooks in this project:
1. 
psid_intergenerational_clean.csv — the analytic dataset from Notebook 01.2 (Prepare Dataset). Each row is a parent–child pair linking a 1968 household head (Generation 1) to one of their children (Generation 2), with recoded variables for homeownership, education, income, race, and more.
2. 
fim15712_gidpro_BO_2_BAL.csv — the PSID FIMS genealogy file, which maps parent–child relationships using 1968 family interview IDs. We chain this file to extend the two-generation links from Notebook 01 into three generations: G1 → G2 → G3.

Both files live in the project data/outputs folders on Google Drive. If you've already run Notebook 01.2, these files are ready to go.

### How the Bridge Works

Notebook 01.2 gives us G1–G2 pairs. To get G3, we look at which G2 individuals are themselves listed as parents in the FIMS file. Their children become G3. We then merge G3's outcome data (education, homeownership) from the PSID extract to build the full three-generation tree.



## A Note on Sample Size

The three-generation sample is intentionally small. To appear in this dataset, a family must clear several hurdles: the G1 household head must be in the 1968 PSID sample with homeownership data; their G2 child must also appear in FIMS with education and homeownership outcomes; and that G2 child must themselves be a parent in FIMS whose G3 children are old enough (25+) to have completed their education and entered the housing market by 2023. That last requirement is the biggest filter. Many G3 grandchildren are simply too young — born after 1998, they haven't yet reached the age where adult outcomes like homeownership can be meaningfully measured. As the PSID continues to follow these families, this three-generation sample will grow substantially. For now, treat the family trees in this notebook as illustrative — real families showing how homeownership, education, and opportunity flow (or don't) across generations. The statistical analysis in Notebook 02 is where the larger two-generation sample does the heavy inferential lifting.

```
# Install required packages!pip install networkx matplotlib ipywidgets -q
```

```
# Import librariesimport pandas as pdimport numpy as npimport matplotlib.pyplot as pltimport networkx as nxfrom ipywidgets import interact, widgets, HBox, VBox, Button, Outputfrom IPython.display import display, clear_outputimport warningswarnings.filterwarnings('ignore')print("✅ Libraries imported successfully!")
```

```
# Mount Google Drivefrom google.colab import drivedrive.mount('/content/drive')print("✅ Google Drive mounted!")Mounted at /content/drive✅ Google Drive mounted!
```

## 1. Load Data and Build Three-Generation Links

We load the two source files and construct the three-generation dataset.

Step 1: Load psid_intergenerational_clean.csv from Notebook 01.2. This gives us G1 → G2 pairs with all recoded variables.

Step 2: Load the FIMS genealogy file and build person IDs. The FIMS file maps G1ID68 + G1PN (parent) to G2ID68 + G2PN (child) using the standard PSID person_id construction: (ID68 × 1000) + PN.

Step 3: Chain the linkage. For each G2 person in our Notebook 01 data, check if they appear as a parent in FIMS. If so, their FIMS children become G3.

```
import os# ── Paths ─────────────────────────────────────────────────────────────────────# These must match the paths used in Notebook 01.2BASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'DATA_DIR   = os.path.join(BASE, 'data')OUTPUT_DIR = os.path.join(BASE, 'outputs')# File produced by Notebook 01.2 — the two-generation analytic datasetNB01_OUTPUT = os.path.join(OUTPUT_DIR, 'psid_intergenerational_clean.csv')# FIMS genealogy file — maps parent–child links using 1968 family IDsFIMS_FILE  = os.path.join(DATA_DIR, 'fim15712_gidpro_BO_2_BAL.csv')# The original PSID extract — needed for G3 outcome variablesPSID_FILE  = os.path.join(DATA_DIR, 'J358642.csv')# Verify files existfor label, path in [('Notebook 01 output', NB01_OUTPUT), ('FIMS file', FIMS_FILE), ('PSID extract', PSID_FILE)]:    if os.path.exists(path):        print(f'✅ {label}: {path}')    else:        print(f'❌ {label} NOT FOUND: {path}')        ✅ Notebook 01 output: /content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2/outputs/psid_intergenerational_clean.csv✅ FIMS file: /content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2/data/fim15712_gidpro_BO_2_BAL.csv✅ PSID extract: /content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2/data/J358642.csv
```

### Build the Three-Generation Dataset

The key insight: in Notebook 01.2's output, parent_id is the G1 (1968 household head) and child_id is G2 (their child). To find G3, we look for G2 individuals who appear as parents in the FIMS genealogy. Their children are G3.

We then merge G3's own outcome variables (education, homeownership) from the PSID extract, applying the same recoding rules used in Notebook 01.2.

```
# ── Step 1: Load the two-generation data from Notebook 01.2 ──────────────────print('📂 Loading Notebook 01.2 output (G1–G2 pairs)...')df_nb01 = pd.read_csv(NB01_OUTPUT)print(f'   Loaded {len(df_nb01):,} rows')print(f'   Columns: {list(df_nb01.columns)}')# ── Step 2: Load FIMS and build linkage ──────────────────────────────────────print('\n📂 Loading FIMS genealogy file...')fims_raw = pd.read_csv(FIMS_FILE)fims = fims_raw.copy()fims['fims_parent_id'] = (fims['G1ID68'] * 1000) + fims['G1PN']fims['fims_child_id']  = (fims['G2ID68'] * 1000) + fims['G2PN']fims = fims[['fims_parent_id', 'fims_child_id']].drop_duplicates()print(f'   FIMS linkage pairs: {len(fims):,}')# ── Step 3: Chain to get G3 ──────────────────────────────────────────────────# G2 people from Notebook 01 are in the 'child_id' column.# Find which of them are themselves parents in FIMS.# Their FIMS children become G3.print('\n🔗 Chaining G2 → G3 via FIMS...')g2_ids = set(df_nb01['child_id'].unique())fims_g2_as_parent = fims[fims['fims_parent_id'].isin(g2_ids)].copy()fims_g2_as_parent = fims_g2_as_parent.rename(columns={    'fims_parent_id': 'g2_id',   # This is the G2 person (child in NB01)    'fims_child_id': 'g3_id'     # This is the G3 person (grandchild of G1)})print(f'   G2 parents found in FIMS: {fims_g2_as_parent["g2_id"].nunique():,}')print(f'   G3 children found: {fims_g2_as_parent["g3_id"].nunique():,}')# ── Step 4: Get G3 outcome data from PSID extract ───────────────────────────print('\n📂 Loading PSID extract for G3 outcomes...')psid_raw = pd.read_csv(PSID_FILE, usecols=['ER30001', 'ER30002', 'ER30004', 'ER32000', 'ER32006', 'ER35152', 'ER82032'])psid_raw['person_id'] = (psid_raw['ER30001'] * 1000) + psid_raw['ER30002']# Filter to sample memberspsid_g3 = psid_raw[psid_raw['ER32006'].isin([1, 2, 3, 4])].copy()# Recode G3 variables (same rules as Notebook 01.2)psid_g3['g3_educ_yrs'] = np.where(psid_g3['ER35152'].isin([0, 99]), np.nan, psid_g3['ER35152'])psid_g3['g3_homeowner'] = np.where(psid_g3['ER82032'] == 1, 1,                            np.where(psid_g3['ER82032'].isin([5, 8]), 0, np.nan))psid_g3['g3_sex'] = psid_g3['ER32000']psid_g3['birth_year_g3'] = np.where(psid_g3['ER30004'] > 0, 1968 - psid_g3['ER30004'], np.nan)# Age restriction: G3 must be >= 25 in 2023psid_g3 = psid_g3[psid_g3['birth_year_g3'] <= 1998].copy()psid_g3 = psid_g3[['person_id', 'g3_educ_yrs', 'g3_homeowner', 'g3_sex', 'birth_year_g3']]psid_g3 = psid_g3.rename(columns={'person_id': 'g3_id'})# ── Step 5: Merge everything into a 3-generation dataset ─────────────────────print('\n🔧 Building three-generation dataset...')# Start: G2→G3 links merged with G3 outcomesdf_3gen = fims_g2_as_parent.merge(psid_g3, on='g3_id', how='inner')# Merge G1 and G2 data from Notebook 01 output# In NB01: parent_id = G1, child_id = G2# race is already a string label ('White', 'Black', etc.) from NB01g1g2_data = df_nb01[['parent_id', 'child_id', 'parent_homeowner', 'parent_educ_yrs',                      'race', 'child_educ_yrs', 'child_homeowner_2023']].copy()g1g2_data = g1g2_data.rename(columns={    'parent_id': 'grandparent_id',    'child_id': 'g2_id',    'parent_homeowner': 'g1_homeowner',        # G1 homeownership (1968)    'parent_educ_yrs': 'g1_educ_yrs',          # G1 education (years)    'race': 'g1_race',                          # G1 race (string label from NB01)    'child_educ_yrs': 'g2_educ_yrs',           # G2 education (years)    'child_homeowner_2023': 'g2_homeowner',     # G2 homeownership (2023)})# Merge: bring G1+G2 data onto the G2→G3 framedf_3gen = df_3gen.merge(g1g2_data, on='g2_id', how='inner')# Rename g2_id to parent_id for the visualization codedf_3gen = df_3gen.rename(columns={'g2_id': 'parent_id'})print(f'\n📈 Three-Generation Structure:')print(f'   Generation 1 (Grandparents): {df_3gen["grandparent_id"].nunique():,} families')print(f'   Generation 2 (Parents): {df_3gen["parent_id"].nunique():,} individuals')print(f'   Generation 3 (Grandchildren): {len(df_3gen):,} individuals')print(f'\n✅ Three-generation dataset built successfully from Notebook 01 output + FIMS!')
```

```
📂 Loading Notebook 01.2 output (G1–G2 pairs)...   Loaded 3,560 rows   Columns: ['child_id', 'parent_id', 'birth_year', 'child_sex', 'child_educ_yrs', 'child_homeowner_2023', 'parent_homeowner', 'parent_educ_yrs', 'parent_income_1968', 'parent_income_1968_log', 'race', 'state_1968', 'head_sex_1968', 'V103', 'V313', 'V81', 'V181', 'V93', 'V119', 'ER35152', 'ER82032', 'ER32000']📂 Loading FIMS genealogy file...   FIMS linkage pairs: 58,585🔗 Chaining G2 → G3 via FIMS...   G2 parents found in FIMS: 1,768   G3 children found: 4,397📂 Loading PSID extract for G3 outcomes...🔧 Building three-generation dataset...📈 Three-Generation Structure:   Generation 1 (Grandparents): 39 families   Generation 2 (Parents): 30 individuals   Generation 3 (Grandchildren): 85 individuals✅ Three-generation dataset built successfully from Notebook 01 output + FIMS!
```

## 2. Prepare Data for Visualization

Now we recode the numeric variables into human-readable labels for the tree visualizations. PSID stores race, sex, and education as numeric codes — we translate them here so the trees are easy to read.

```
# ── Recode variables for display ─────────────────────────────────────────────print('🔧 Preparing labels for visualization...')df = df_3gen.copy()# Only require grandparent columns to be completegrandparent_cols = ['grandparent_id', 'g1_homeowner']df_clean = df.dropna(subset=grandparent_cols)print(f'After filtering for complete grandparent data: {len(df_clean):,} rows')# ── Homeownership labels (all three generations) ────────────────────────────homeowner_map = {1.0: 'Owner', 0.0: 'Renter'}sex_map = {1.0: 'Male', 2.0: 'Female'}df_clean['g1_homeowner_label'] = df_clean['g1_homeowner'].map(homeowner_map)df_clean['g2_homeowner_label'] = df_clean['g2_homeowner'].map(homeowner_map)df_clean['g3_homeowner_label'] = df_clean['g3_homeowner'].map(homeowner_map)# ── Race label ───────────────────────────────────────────────────────────────# g1_race is already a string ('White', 'Black', etc.) from Notebook 01.2# No numeric-to-string conversion neededdf_clean['g1_race_label'] = df_clean['g1_race']# ── Education: convert years to category codes for color mapping ─────────────def years_to_ed_code(years):    if pd.isna(years): return np.nan    if years <= 5:   return 1   # 0-5 grades    elif years <= 8: return 2   # 6-8 grades    elif years <= 11: return 3  # 9-11 grades    elif years == 12: return 4  # HS graduate    elif years <= 14: return 5  # Some college    elif years <= 16: return 6  # College+    else: return 7              # Advanced degreedf_clean['g2_educ_code'] = df_clean['g2_educ_yrs'].apply(years_to_ed_code)# ── G2 education labels ─────────────────────────────────────────────────────parent_ed_map = {    1: '0-5 grades', 2: '6-8 grades', 3: '9-11 grades',    4: 'HS Graduate', 5: 'Some College', 6: 'College+',    7: 'Advanced Degree'}df_clean['g2_ed_label'] = df_clean['g2_educ_code'].map(parent_ed_map)# ── Sex labels ───────────────────────────────────────────────────────────────df_clean['g3_sex_label'] = df_clean['g3_sex'].map(sex_map)# ── G3 education category ───────────────────────────────────────────────────def categorize_ed(years):    if pd.isna(years): return 'Unknown'    elif years < 12: return '<HS'    elif years == 12: return 'HS'    elif years < 16: return 'Some College'    elif years >= 16: return 'College+'    return 'Unknown'df_clean['g3_ed_category'] = df_clean['g3_educ_yrs'].apply(categorize_ed)# ── Family IDs ───────────────────────────────────────────────────────────────unique_grandparents = df_clean['grandparent_id'].unique()family_id_map = {gid: f'Family_{i:05d}' for i, gid in enumerate(unique_grandparents, 1)}df_clean['family_id'] = df_clean['grandparent_id'].map(family_id_map)# ── Summary ──────────────────────────────────────────────────────────────────g1_counts = df_clean['grandparent_id'].nunique()g2_counts = df_clean['parent_id'].nunique()g3_counts = len(df_clean)print(f'\n📈 Three-Generation Structure:')print(f'   Generation 1 (Grandparents): {g1_counts:,} families')print(f'   Generation 2 (Parents): {g2_counts:,} families')print(f'   Generation 3 (Grandchildren): {g3_counts:,} individuals')if 'g1_race_label' in df_clean.columns:    print(f'\n📊 Race distribution (G1):')    race_dist = df_clean.groupby('g1_race_label')['grandparent_id'].nunique()    for race, count in race_dist.items():        print(f'   {race}: {count:,} families')print(f'\n📊 Homeownership labels check:')print(f'   G1: {df_clean["g1_homeowner_label"].value_counts().to_dict()}')print(f'   G2: {df_clean["g2_homeowner_label"].value_counts(dropna=False).to_dict()}')print(f'   G3: {df_clean["g3_homeowner_label"].value_counts(dropna=False).to_dict()}')print(f'\n✅ Data preparation complete!')
```

## 3. Build and Visualize Family Trees

The tree builder takes all rows for a given family (identified by grandparent_id) and reconstructs the three-generation structure:
- 
G1 node: The 1968 household head. Labeled with race and homeownership status.
- 
G2 nodes: Their children (one per unique parent_id). Labeled with education category.
- 
G3 nodes: Grandchildren. Labeled with sex and years of education.

The visualization uses color to show education level (red = less than HS, green = college, blue = graduate+) and shape to show homeownership (circle = owner, square = renter, diamond = unknown).

```
# Build 3-generation treedef build_three_gen_tree(family_rows):    """Build a 3-generation family tree showing educational outcomes.    Research Question: Does Gen1 homeownership affect Gen2 & Gen3 education?    Gen 1 = Grandparent (1968 homeowner status)    Gen 2 = Parent (has g2_educ_yrs measured as adult)    Gen 3 = Grandchild (has g3_educ_yrs measured as adult)    """    first_row = family_rows.iloc[0]    tree = {        'family_id': first_row['family_id'],        'g1': {            'id': first_row['grandparent_id'],            'race': first_row.get('g1_race_label', 'Unknown'),            'homeowner_1968': first_row.get('g1_homeowner_label', 'Unknown'),        },        'g2': {},  # Will be dict of parent_id -> parent data        'g3': []   # List of all grandchildren    }    # Group by parent_id to get Generation 2 structure    for parent_id in family_rows['parent_id'].unique():        parent_rows = family_rows[family_rows['parent_id'] == parent_id]        first_parent_row = parent_rows.iloc[0]        tree['g2'][parent_id] = {            'id': parent_id,            'education_category': first_parent_row.get('g2_ed_label', 'Unknown'),            'education_code': first_parent_row.get('g2_educ_code'),            'education_years': first_parent_row.get('g2_educ_yrs'),            'homeowner': first_parent_row.get('g2_homeowner_label', 'Unknown'),            'children': []        }        # Gen 3 - Add each grandchild with their education        for idx, row in parent_rows.iterrows():            child_data = {                'parent_id': parent_id,                'sex': row.get('g3_sex_label', 'Unknown'),                'education_years': row.get('g3_educ_yrs'),                'education_category': row.get('g3_ed_category', 'Unknown'),                'homeowner': row.get('g3_homeowner_label', 'Unknown'),            }            tree['g2'][parent_id]['children'].append(child_data)            tree['g3'].append(child_data)    return treeprint("✅ Three-generation tree builder defined")
```

```
================================================================================🌳 INTERGENERATIONAL EDUCATION EXPLORER (Longitudinal)================================================================================Research Question: Does 1968 homeownership affect educationaloutcomes for children and grandchildren measured decades later?Note: Gen2 & Gen3 education measured when they were ADULTS.This shows COMPLETED education, not childhood education.
```

![]()



![]()


















