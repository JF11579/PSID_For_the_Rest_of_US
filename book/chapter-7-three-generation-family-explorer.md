# Chapter 7: Three Generation Family Explorer


# Chapter 7: Three Generation Family Explorer

## The Payoff

Up to this point, the families in this study have been rows in a spreadsheet. We have counted them, regressed them, sliced them by race and sex and birth decade — but we have never actually looked at one. This chapter changes that.

The Three Generation Family Explorer is a notebook that builds interactive family trees from the same PSID data we have been analyzing. Each tree shows three generations of a single family: the 1968 household head at the top (Generation 1), their adult children in the middle (Generation 2), and their grandchildren at the bottom (Generation 3). Color represents education level. Shape represents homeownership. You can filter by race and homeownership status, click a button to see a random family, or search for a specific one by ID.

It is, frankly, the most fun notebook in the project. After spending chapters watching coefficients tick up and down, there is something grounding about seeing a real family where the grandparent owned a home in 1968, all three of their children finished college, and five of their six grandchildren did too. Or a family where the grandparent rented, and the educational attainment thins out as you move down the tree. The regression told us the effect was about one year of education. The family trees show you what that year looks like when it compounds — or doesn't — across generations.

## Setup

The notebook starts with the standard Colab setup: install packages, import libraries, mount Google Drive.

```
# Install required packages!pip install networkx matplotlib ipywidgets -q
```

```
# Import librariesimport pandas as pdimport numpy as npimport matplotlib.pyplot as pltimport networkx as nxfrom ipywidgets import interact, widgets, HBox, VBox, Button, Outputfrom IPython.display import display, clear_outputimport warningswarnings.filterwarnings('ignore')print("✅ Libraries imported successfully!")
```

NetworkX is the new addition here. We have used pandas and matplotlib throughout this project, but NetworkX is a graph library — it lets us define nodes (people) and edges (parent-child links), arrange them in a layout, and draw them. It is the engine behind the family tree visualizations.

```
# Mount Google Drivefrom google.colab import drivedrive.mount('/content/drive')print("✅ Google Drive mounted!")
```

## Part 1: Load Data and Build Three-Generation Links

This is where the notebook does something the earlier notebooks did not: it extends the two-generation structure from Notebook 01.2 into three generations. The logic works in five steps.

First, we set up the paths. Three files are needed: the clean dataset from Notebook 01.2, the FIMS genealogy file, and the original PSID extract (for G3 outcome variables).

```
import os# ── Paths ─────────────────────────────────────────────────────────────────────BASE = '/content/drive/MyDrive/Colab Notebooks/Projects/PSID/PSID_2026/PSID_book_v2'DATA_DIR   = os.path.join(BASE, 'data')OUTPUT_DIR = os.path.join(BASE, 'outputs')# File produced by Notebook 01.2 — the two-generation analytic datasetNB01_OUTPUT = os.path.join(OUTPUT_DIR, 'psid_intergenerational_clean.csv')# FIMS genealogy file — maps parent–child links using 1968 family IDsFIMS_FILE  = os.path.join(DATA_DIR, 'fim15712_gidpro_BO_2_BAL.csv')# The original PSID extract — needed for G3 outcome variablesPSID_FILE  = os.path.join(DATA_DIR, 'J358642.csv')# Verify files existfor label, path in [('Notebook 01 output', NB01_OUTPUT),                     ('FIMS file', FIMS_FILE),                     ('PSID extract', PSID_FILE)]:    if os.path.exists(path):        print(f'✅ {label}: {path}')    else:        print(f'❌ {label} NOT FOUND: {path}')
```

The key insight is how the generational chaining works. In Notebook 01.2's output, parent_id is G1 (the 1968 household head) and child_id is G2 (their child). To find G3, we ask: which of those G2 children went on to become parents themselves? The FIMS file knows. It maps every parent-child relationship in the PSID. So we take each G2 person, look them up as a potential parent in FIMS, and if they have children listed, those children become G3.

Once we identify G3 individuals, we go back to the raw PSID extract and pull their outcome variables — education (ER35152), homeownership (ER82032), sex (ER32000), and age derived from ER30004. We apply the same recoding rules used in Notebook 01.2: education values of 0 and 99 become missing, homeownership is 1 for owners and 0 for renters or neither, and we restrict to age 25 or older in 2023 so we are measuring completed education.

Here is the full data-building cell:

```
# ── Step 1: Load the two-generation data from Notebook 01.2 ──────────────────print('📂 Loading Notebook 01.2 output (G1–G2 pairs)...')df_nb01 = pd.read_csv(NB01_OUTPUT)print(f'   Loaded {len(df_nb01):,} rows')print(f'   Columns: {list(df_nb01.columns)}')# ── Step 2: Load FIMS and build linkage ──────────────────────────────────────print('\n📂 Loading FIMS genealogy file...')fims_raw = pd.read_csv(FIMS_FILE)fims = fims_raw.copy()fims['fims_parent_id'] = (fims['G1ID68'] * 1000) + fims['G1PN']fims['fims_child_id']  = (fims['G2ID68'] * 1000) + fims['G2PN']fims = fims[['fims_parent_id', 'fims_child_id']].drop_duplicates()print(f'   FIMS linkage pairs: {len(fims):,}')# ── Step 3: Chain to get G3 ──────────────────────────────────────────────────print('\n🔗 Chaining G2 → G3 via FIMS...')g2_ids = set(df_nb01['child_id'].unique())fims_g2_as_parent = fims[fims['fims_parent_id'].isin(g2_ids)].copy()fims_g2_as_parent = fims_g2_as_parent.rename(columns={    'fims_parent_id': 'g2_id',    'fims_child_id': 'g3_id'})print(f'   G2 parents found in FIMS: {fims_g2_as_parent["g2_id"].nunique():,}')print(f'   G3 children found: {fims_g2_as_parent["g3_id"].nunique():,}')# ── Step 4: Get G3 outcome data from PSID extract ───────────────────────────print('\n📂 Loading PSID extract for G3 outcomes...')psid_raw = pd.read_csv(PSID_FILE,    usecols=['ER30001', 'ER30002', 'ER30004', 'ER32000',             'ER32006', 'ER35152', 'ER82032'])psid_raw['person_id'] = (psid_raw['ER30001'] * 1000) + psid_raw['ER30002']psid_g3 = psid_raw[psid_raw['ER32006'].isin([1, 2, 3, 4])].copy()# Recode G3 variables (same rules as Notebook 01.2)psid_g3['g3_educ_yrs'] = np.where(    psid_g3['ER35152'].isin([0, 99]), np.nan, psid_g3['ER35152'])psid_g3['g3_homeowner'] = np.where(    psid_g3['ER82032'] == 1, 1,    np.where(psid_g3['ER82032'].isin([5, 8]), 0, np.nan))psid_g3['g3_sex'] = psid_g3['ER32000']psid_g3['birth_year_g3'] = np.where(    psid_g3['ER30004'] > 0, 1968 - psid_g3['ER30004'], np.nan)# Age restriction: G3 must be >= 25 in 2023psid_g3 = psid_g3[psid_g3['birth_year_g3'] <= 1998].copy()psid_g3 = psid_g3[['person_id', 'g3_educ_yrs', 'g3_homeowner',                     'g3_sex', 'birth_year_g3']]psid_g3 = psid_g3.rename(columns={'person_id': 'g3_id'})# ── Step 5: Merge everything into a 3-generation dataset ─────────────────────print('\n🔧 Building three-generation dataset...')df_3gen = fims_g2_as_parent.merge(psid_g3, on='g3_id', how='inner')g1g2_data = df_nb01[['parent_id', 'child_id', 'parent_homeowner',    'parent_educ_yrs', 'race', 'child_educ_yrs',    'child_homeowner_2023']].copy()g1g2_data = g1g2_data.rename(columns={    'parent_id': 'grandparent_id',    'child_id': 'g2_id',    'parent_homeowner': 'g1_homeowner',    'parent_educ_yrs': 'g1_educ_yrs',    'race': 'g1_race',    'child_educ_yrs': 'g2_educ_yrs',    'child_homeowner_2023': 'g2_homeowner',})df_3gen = df_3gen.merge(g1g2_data, on='g2_id', how='inner')df_3gen = df_3gen.rename(columns={'g2_id': 'parent_id'})print(f'\n📈 Three-Generation Structure:')print(f'   Generation 1 (Grandparents): {df_3gen["grandparent_id"].nunique():,} families')print(f'   Generation 2 (Parents): {df_3gen["parent_id"].nunique():,} individuals')print(f'   Generation 3 (Grandchildren): {len(df_3gen):,} individuals')print(f'\n✅ Three-generation dataset built successfully!')
```

Output:

```
📂 Loading Notebook 01.2 output (G1–G2 pairs)...   Loaded 3,560 rows📂 Loading FIMS genealogy file...   FIMS linkage pairs: 58,585🔗 Chaining G2 → G3 via FIMS...   G2 parents found in FIMS: 1,768   G3 children found: 4,397📂 Loading PSID extract for G3 outcomes...🔧 Building three-generation dataset...📈 Three-Generation Structure:   Generation 1 (Grandparents): 39 families   Generation 2 (Parents): 30 individuals   Generation 3 (Grandchildren): 85 individuals✅ Three-generation dataset built successfully!
```

Look at that funnel. We start with 3,560 G1–G2 pairs from Notebook 01. The FIMS file contains 58,585 parent-child links across the full PSID. Of our G2 individuals, 1,768 appear as parents in FIMS, producing 4,397 potential G3 grandchildren. But after restricting to PSID sample members aged 25 or older with valid data, we land on 85 grandchildren in 39 families.

## A Note on Sample Size

That sample is small — and that is by design, not by error.

To appear in this dataset, a family has to clear several hurdles. The G1 household head must be in the original 1968 PSID sample with valid homeownership data. Their G2 child must appear in the FIMS genealogy with education and homeownership outcomes. And that G2 child must themselves be a parent whose G3 children are old enough — born in 1998 or earlier — to have completed their education and entered the housing market by 2023.

That last requirement is the biggest filter. Many G3 grandchildren are simply too young. Born after 1998, they have not yet reached the age where we can meaningfully measure adult outcomes like homeownership. As the PSID continues to follow these families in future waves, this three-generation sample will grow substantially.

The race distribution reflects this filtering: 30 of the 39 families are Black, 9 are White.  Black families in the PSID tend to have earlier family formation, meaning their G3 grandchildren are more likely to have crossed the age-25 threshold by 2023.

For now, treat the family trees in this chapter as illustrative. These are real families showing how homeownership, education, and opportunity flow — or do not flow — across generations. The statistical heavy lifting was done in Notebook 02 with the much larger two-generation sample of 3,560 pairs.

## Part 2: Prepare Data for Visualization

Before we can draw the trees, we need to translate the numeric codes into human-readable labels. The PSID stores everything as numbers — homeownership as 1 and 0, sex as 1 and 2, education as years. The visualization needs words and categories.

```
# ── Recode variables for display ─────────────────────────────────────────────print('🔧 Preparing labels for visualization...')df = df_3gen.copy()grandparent_cols = ['grandparent_id', 'g1_homeowner']df_clean = df.dropna(subset=grandparent_cols)print(f'After filtering for complete grandparent data: {len(df_clean):,} rows')# ── Homeownership labels (all three generations) ────────────────────────────homeowner_map = {1.0: 'Owner', 0.0: 'Renter'}sex_map = {1.0: 'Male', 2.0: 'Female'}df_clean['g1_homeowner_label'] = df_clean['g1_homeowner'].map(homeowner_map)df_clean['g2_homeowner_label'] = df_clean['g2_homeowner'].map(homeowner_map)df_clean['g3_homeowner_label'] = df_clean['g3_homeowner'].map(homeowner_map)# g1_race is already a string ('White', 'Black', etc.) from Notebook 01.2df_clean['g1_race_label'] = df_clean['g1_race']# ── Education: convert years to category codes for color mapping ─────────────def years_to_ed_code(years):    if pd.isna(years): return np.nan    if years <= 5:   return 1   # 0-5 grades    elif years <= 8: return 2   # 6-8 grades    elif years <= 11: return 3  # 9-11 grades    elif years == 12: return 4  # HS graduate    elif years <= 14: return 5  # Some college    elif years <= 16: return 6  # College+    else: return 7              # Advanced degreedf_clean['g2_educ_code'] = df_clean['g2_educ_yrs'].apply(years_to_ed_code)parent_ed_map = {    1: '0-5 grades', 2: '6-8 grades', 3: '9-11 grades',    4: 'HS Graduate', 5: 'Some College', 6: 'College+',    7: 'Advanced Degree'}df_clean['g2_ed_label'] = df_clean['g2_educ_code'].map(parent_ed_map)df_clean['g3_sex_label'] = df_clean['g3_sex'].map(sex_map)def categorize_ed(years):    if pd.isna(years): return 'Unknown'    elif years < 12: return '<HS'    elif years == 12: return 'HS'    elif years < 16: return 'Some College'    elif years >= 16: return 'College+'    return 'Unknown'df_clean['g3_ed_category'] = df_clean['g3_educ_yrs'].apply(categorize_ed)# ── Family IDs ───────────────────────────────────────────────────────────────unique_grandparents = df_clean['grandparent_id'].unique()family_id_map = {gid: f'Family_{i:05d}'                 for i, gid in enumerate(unique_grandparents, 1)}df_clean['family_id'] = df_clean['grandparent_id'].map(family_id_map)
```

A few things to notice here.

Homeownership labels are created for all three generations — G1, G2, and G3. This matters. In the original version of this notebook, only G1 had homeownership labels wired through. G2 and G3 nodes always showed as diamonds (unknown ownership) in the tree, even when the data was there. The corrected version maps Owner/Renter for all three generations. Nearly half of G3 individuals (41 of 85) still show as unknown because they have not yet appeared as household heads in a PSID wave — but that is real missingness, not a bug.

Race comes through as a string label directly from Notebook 01.2. No numeric-to-string conversion needed.

Education gets two treatments. For G2 (the parents), we convert years into ordinal category codes (1 through 7) that drive the color mapping. For G3 (the grandchildren), we keep the actual years and add a category label alongside. The trees will show both — "16y (College)" is more informative than just "College" because it tells you the exact measurement.

Each family gets a unique ID based on its G1 grandparent: Family_00001, Family_00002, and so on. These are the IDs you use to search for a specific family in the interactive interface.

Output:

```
📈 Three-Generation Structure:   Generation 1 (Grandparents): 39 families   Generation 2 (Parents): 30 families   Generation 3 (Grandchildren): 85 individuals📊 Race distribution (G1):   Black: 30 families   White: 9 families📊 Homeownership labels check:   G1: {'Renter': 43, 'Owner': 42}   G2: {'Owner': 45, 'Renter': 40}   G3: {nan: 41, 'Owner': 27, 'Renter': 17}✅ Data preparation complete!
```

## Part 3: Build and Visualize Family Trees

Now the fun part. For each family, we build a nested dictionary with three levels: G1 at the top (race, homeownership), G2 in the middle (education, homeownership), and G3 at the bottom (sex, education years, homeownership). Then we render it as a directed graph using NetworkX.

### The Tree Builder

```
def build_three_gen_tree(family_rows):    """Build a 3-generation family tree showing educational outcomes.    Gen 1 = Grandparent (1968 homeowner status)    Gen 2 = Parent (has g2_educ_yrs measured as adult)    Gen 3 = Grandchild (has g3_educ_yrs measured as adult)    """    first_row = family_rows.iloc[0]    tree = {        'family_id': first_row['family_id'],        'g1': {            'id': first_row['grandparent_id'],            'race': first_row.get('g1_race_label', 'Unknown'),            'homeowner_1968': first_row.get('g1_homeowner_label', 'Unknown'),        },        'g2': {},        'g3': []    }    for parent_id in family_rows['parent_id'].unique():        parent_rows = family_rows[family_rows['parent_id'] == parent_id]        first_parent_row = parent_rows.iloc[0]        tree['g2'][parent_id] = {            'id': parent_id,            'education_category': first_parent_row.get('g2_ed_label', 'Unknown'),            'education_code': first_parent_row.get('g2_educ_code'),            'education_years': first_parent_row.get('g2_educ_yrs'),            'homeowner': first_parent_row.get('g2_homeowner_label', 'Unknown'),            'children': []        }        for idx, row in parent_rows.iterrows():            child_data = {                'parent_id': parent_id,                'sex': row.get('g3_sex_label', 'Unknown'),                'education_years': row.get('g3_educ_yrs'),                'education_category': row.get('g3_ed_category', 'Unknown'),                'homeowner': row.get('g3_homeowner_label', 'Unknown'),            }            tree['g2'][parent_id]['children'].append(child_data)            tree['g3'].append(child_data)    return tree
```

The function takes all rows for one family (identified by grandparent_id) and organizes them into a tree structure. It loops through each unique G2 parent, collects their G3 children, and nests everything under the G1 grandparent. The result is a dictionary that the visualization function can walk through top-to-bottom.

### The Visualization

The visualization function is the longest piece of code in the notebook. It takes a tree dictionary, creates a NetworkX directed graph, assigns colors and shapes to every node, computes a layered layout, and draws the result.

The encoding scheme:

Color = Education Level:
- 
Red (#D32F2F) — Less than high school (< 12 years)
- 
Orange (#F57C00) — High school graduate (12 years)
- 
Yellow (#FBC02D) — Some college (13–15 years)
- 
Green (#388E3C) — College graduate (16 years)
- 
Blue (#1976D2) — Graduate school (17+ years)
- 
Gray (#90A4AE) — Education data missing
- 
Purple (#7E57C2) — G1 node (always purple, education not the focus)

Shape = Homeownership:
- 
Circle (o) — Owner
- 
Square (s) — Renter
- 
Diamond (d) — Unknown / missing

```
def visualize_three_gen_tree(tree, figsize=(18, 12)):    """Visualize educational outcomes across 3 generations.    COLOR = Education level, SHAPE = Homeownership    """    G = nx.DiGraph()    def get_education_color_from_years(years):        if pd.isna(years):  return '#90A4AE'        if years < 12:      return '#D32F2F'        elif years == 12:   return '#F57C00'        elif years < 16:    return '#FBC02D'        elif years == 16:   return '#388E3C'        else:               return '#1976D2'    def get_education_color_from_code(code):        if pd.isna(code):   return '#90A4AE'        code = int(code)        if code <= 3:       return '#D32F2F'        elif code == 4:     return '#F57C00'        elif code == 5:     return '#FBC02D'        elif code == 6:     return '#388E3C'        elif code >= 7:     return '#1976D2'        return '#90A4AE'    def get_ed_label_from_code(code):        if pd.isna(code):   return '?'        code = int(code)        if code <= 3:       return '<HS'        elif code == 4:     return 'HS'        elif code == 5:     return 'SomeColl'        elif code == 6:     return 'College'        elif code >= 7:     return 'Grad+'        return '?'    def get_ed_label_from_years(years):        if pd.isna(years):  return '?'        y = int(years)        if y < 12:          return f'{y}y (<HS)'        elif y == 12:       return '12y (HS)'        elif y < 16:        return f'{y}y (SomeColl)'        elif y == 16:       return '16y (College)'        else:               return f'{y}y (Grad+)'    def get_ownership_shape(owner_status):        if owner_status == 'Owner':   return 'o'        elif owner_status == 'Renter': return 's'        else:                          return 'd'    nodes_by_shape = {'o': [], 's': [], 'd': []}    node_colors = {}    node_labels = {}    # ── Gen 1 ────────────────────────────────────────────────────────────────    grandparent_node = 'g1'    homeowner_status = tree['g1']['homeowner_1968']    g1_shape = get_ownership_shape(homeowner_status)    g1_label = f"G1\n{tree['g1']['race']}\n{homeowner_status}"    G.add_node(grandparent_node, label=g1_label, generation=1)    nodes_by_shape[g1_shape].append(grandparent_node)    node_colors[grandparent_node] = '#7E57C2'    node_labels[grandparent_node] = g1_label    # ── Gen 2 ────────────────────────────────────────────────────────────────    parent_nodes = []    for i, (parent_id, parent_data) in enumerate(tree['g2'].items()):        parent_node = f"g2_{i}"        parent_nodes.append(parent_node)        parent_ed_code = parent_data.get('education_code')        parent_owner = parent_data.get('homeowner', 'Unknown')        parent_color = get_education_color_from_code(parent_ed_code)        parent_shape = get_ownership_shape(parent_owner)        parent_label = f"G2-{i+1}\n{get_ed_label_from_code(parent_ed_code)}"        G.add_node(parent_node, label=parent_label, generation=2)        G.add_edge(grandparent_node, parent_node)        nodes_by_shape[parent_shape].append(parent_node)        node_colors[parent_node] = parent_color        node_labels[parent_node] = parent_label        # ── Gen 3 ────────────────────────────────────────────────────────────        for j, child in enumerate(parent_data['children']):            child_node = f"g3_{i}_{j}"            ed_years = child['education_years']            child_color = get_education_color_from_years(ed_years)            child_shape = get_ownership_shape(child.get('homeowner', 'Unknown'))            child_label = f"G3\n{child['sex']}\n{get_ed_label_from_years(ed_years)}"            G.add_node(child_node, label=child_label, generation=3)            G.add_edge(parent_node, child_node)            nodes_by_shape[child_shape].append(child_node)            node_colors[child_node] = child_color            node_labels[child_node] = child_label    # ── Layout ───────────────────────────────────────────────────────────────    pos = {grandparent_node: (0, 2)}    num_parents = len(parent_nodes)    parent_x = [0] if num_parents == 1 \        else list(np.linspace(-2, 2, num_parents))    for i, pnode in enumerate(parent_nodes):        pos[pnode] = (parent_x[i], 1)    for i, (pid, pdata) in enumerate(tree['g2'].items()):        nc = len(pdata['children'])        px = parent_x[i]        if nc == 1:            cx = [px]        else:            spread = min(1.3, nc * 0.4)            cx = list(np.linspace(px - spread/2, px + spread/2, nc))        for j in range(nc):            pos[f"g3_{i}_{j}"] = (cx[j], 0)    # ── Draw ─────────────────────────────────────────────────────────────────    fig, ax = plt.subplots(figsize=figsize)    nx.draw_networkx_edges(G, pos, ax=ax, edge_color='#666666',                          width=1.5, alpha=0.4, arrows=False)    for shape, node_list in nodes_by_shape.items():        if node_list:            colors = [node_colors[n] for n in node_list]            nx.draw_networkx_nodes(G, pos, nodelist=node_list,                node_color=colors, node_size=5250,                node_shape=shape, ax=ax,                edgecolors='black', linewidths=2)    nx.draw_networkx_labels(G, pos, node_labels, font_size=9,        font_color='white', font_weight='bold',        verticalalignment='center', ax=ax)    title = f"Family: {tree['family_id']}\n"    title += f"G1 (1968): {tree['g1']['race']} {homeowner_status}"    ax.set_title(title, fontsize=14, fontweight='bold', pad=20)    from matplotlib.patches import Patch    from matplotlib.lines import Line2D    ed_legend = [        Patch(facecolor='#D32F2F', label='<12y = <HS'),        Patch(facecolor='#F57C00', label='12y = HS'),        Patch(facecolor='#FBC02D', label='13-15y = Some College'),        Patch(facecolor='#388E3C', label='16y = College'),        Patch(facecolor='#1976D2', label='17+y = Graduate+'),    ]    shape_legend = [        Line2D([0], [0], marker='o', color='w', markerfacecolor='gray',               markersize=10, label='Owner', markeredgecolor='black'),        Line2D([0], [0], marker='s', color='w', markerfacecolor='gray',               markersize=10, label='Renter', markeredgecolor='black'),        Line2D([0], [0], marker='d', color='w', markerfacecolor='gray',               markersize=10, label='Unknown', markeredgecolor='black'),    ]    first_legend = ax.legend(handles=ed_legend, loc='upper left',                            title='Years of Education', fontsize=8)    ax.add_artist(first_legend)    ax.legend(handles=shape_legend, loc='upper right',             title='Homeownership', fontsize=8)    ax.axis('off')    plt.tight_layout()    plt.show()
```

That is a lot of code for what is essentially "draw colored shapes and connect them with lines." But most of it is bookkeeping — mapping education values to colors, homeownership to shapes, computing positions so the tree looks right. The NetworkX library does the hard part of graph layout and rendering.

![]()

### What a Family Tree Looks Like

 



Start at the top. The purple G1 node shows the 1968 household head — their race and whether they owned or rented. The shape tells you ownership: a circle means owner, a square means renter.

Move down to G2. Each node represents one of the G1 parent's children, now an adult. The color tells you their education level — green for college, blue for graduate school, orange for high school. The shape tells you whether they own a home in 2023.

At the bottom, G3. Each node is a grandchild, labeled with sex and exact years of education. Again, color is education and shape is homeownership. Diamonds mean we do not yet have homeownership data for that person — common for younger G3 individuals who have not appeared as household heads in a PSID wave.

The patterns are not always clean. Some owner families have grandchildren who did not finish high school. Some renter families produced college graduates across all three generations. That is the point — the regression gives you the average, but the trees show you the variance.

## Part 4: Interactive Controls

The final section of the notebook sets up an interactive interface. Two dropdown filters let you narrow by G1 race and G1 homeownership status. Two buttons let you either show a random family matching the filters or search for a specific family by ID.

```
race_widget = widgets.Dropdown(    options=['All', 'White', 'Black', 'Hispanic', 'Other'],    value='All',    description='Gen1 Race:',    style={'description_width': '150px'},    layout=widgets.Layout(width='350px'))homeowner_widget = widgets.Dropdown(    options=['All', 'Owner', 'Renter'],    value='All',    description='Gen1 Homeowner (1968):',    style={'description_width': '200px'},    layout=widgets.Layout(width='400px'))search_widget = widgets.Text(    placeholder='Enter Family ID (e.g., Family_00001)',    description='Family ID:',    style={'description_width': '100px'},    layout=widgets.Layout(width='350px'))
```

The callback functions are straightforward. show_random_family applies whatever filters are currently selected, picks a random family from the matching set, builds its tree, and draws it. search_by_id does the same for a specific Family ID. Both update the output area in place, so you can click through family after family without the page scrolling away.

```
def show_random_family(b=None):    with output:        clear_output(wait=True)        filtered = df_clean.copy()        if race_widget.value != 'All':            filtered = filtered[                filtered['g1_race_label'] == race_widget.value]        if homeowner_widget.value != 'All':            filtered = filtered[                filtered['g1_homeowner_label'] == homeowner_widget.value]        if len(filtered) == 0:            print("❌ No families found matching your criteria!")            return        family_ids = filtered['family_id'].unique()        random_family_id = np.random.choice(family_ids)        family_rows = df_clean[df_clean['family_id'] == random_family_id]        tree = build_three_gen_tree(family_rows)        visualize_three_gen_tree(tree)
```

Family IDs are created from grandparent_id as Family_00001, Family_00002, and so on. If you find a family you want to revisit, note the ID from the title of the tree and type it into the search box.

## Tips for Exploring

Set the race filter to "Black" and the homeownership filter to "Owner," then click through a few random families. Then switch to "Renter" and do the same. You will start to see the patterns that the regression quantified — but now with faces (or at least, with nodes).

Try filtering to White owners and White renters. The sample is small (9 White families total), but the contrast can still be striking.

Pay attention to the shapes. Now that homeownership is tracked across all three generations, you can see the transmission chain: did the grandparent's 1968 ownership predict whether their children and grandchildren became owners themselves? G1 homeownership comes from V103 in the 1968 family file. G2 and G3 homeownership come from ER82032 in the 2023 wave. The diamonds on G3 nodes (about half the sample) are real missingness — those grandchildren have not yet appeared as household heads in a PSID wave.

![]()

## What Comes Next

The family trees give you a qualitative feel for the data — one family at a time, three generations deep. 



Next up: the full regression report — every number, every model, every finding in one place.
