# Universal PSID Prep Tool


# Universal PSID Prep Tool

A general-purpose notebook for cleaning and merging any PSID Data Center extract

## What This Notebook Does

If you've just downloaded data from the PSID Data Center, you're probably staring at a pile of cryptic variable names like V103, ER32000, and ER30001 — and wondering what to do next.

This notebook takes you from that raw download to a clean, labeled, analysis-ready CSV file. Specifically, it will:
1. 
Load your PSID extract (the CSV file from the Data Center)
2. 
Convert your FIMS file (the Excel file that maps IDs to variable names) into a usable format
3. 
Build person IDs so you can track individuals across waves
4. 
Merge parent–child links so you can connect generations
5. 
Apply human-readable labels from the PSID's own label definitions
6. 
Filter by sample status to keep only valid PSID sample members
7. 
Export a clean CSV ready for your analysis

You don't need to know your variables in advance. You don't need to be a Python expert. Just follow along, cell by cell.

## Before You Start: What You Need

You should have three files from the PSID Data Center:

Place all three files in the same folder as this notebook, or update the file paths in the Configuration cell below.

> 
Don't have a labels.txt file? That's okay — the notebook will still work. You'll just get numeric codes (like 1, 5, 8) instead of labels (like Owns, Rents, Neither). You can always add labels later.


## Step 0: Configuration

This is the only cell you must edit. Everything else runs as-is.

Update the three file paths below to match your actual file names. If your files are in the same folder as this notebook, you only need the file names — no full paths required.

```
# ============================================================# CONFIGURATION — Edit these three lines, then run everything# ============================================================# Your PSID Data Center extract (CSV)EXTRACT_FILE = "J358642.csv"            # <-- Replace with your file name# Your FIMS mapping file (usually .xlsx, sometimes .csv)FIMS_FILE = "J358642.xlsx"              # <-- Replace with your file name# Your labels file (set to None if you don't have one)LABELS_FILE = "labels.txt"              # <-- Replace with your file name, or set to None# Output file name (what you want your clean data saved as)OUTPUT_FILE = "psid_clean.csv"          # <-- Change if you like# Also save a pre-filter version? (before sample/age filtering)SAVE_PRE_FILTER = True                  # Set to False if you only want the final filePRE_FILTER_FILE = "psid_pre_filter.csv"
```

## Step 1: Import Libraries

We only need two libraries, both of which come pre-installed with Anaconda or any standard Python data-science setup:
- 
pandas — the workhorse for reading, cleaning, and reshaping data
- 
os — just for checking that your files actually exist before we try to open them

```
import pandas as pdimport osimport warningswarnings.filterwarnings('ignore')  # Suppress openpyxl styling warningsprint("Libraries loaded.")
```

## Step 2: Verify Your Files Exist

Before we do anything else, let's make sure Python can actually find your files. This saves you from the frustration of running ten cells only to discover a typo in a file name.

If any file is missing, you'll get a clear message telling you exactly what's wrong.

```
# Check that required files existall_good = Truefor label, path in [("Extract", EXTRACT_FILE), ("FIMS", FIMS_FILE)]:    if os.path.exists(path):        size_mb = os.path.getsize(path) / (1024 * 1024)        print(f"  ✓ {label} file found: {path} ({size_mb:.1f} MB)")    else:        print(f"  ✗ {label} file NOT FOUND: {path}")        print(f"    → Check the file name in the Configuration cell above.")        all_good = Falseif LABELS_FILE and os.path.exists(LABELS_FILE):    print(f"  ✓ Labels file found: {LABELS_FILE}")elif LABELS_FILE:    print(f"  ⚠ Labels file not found: {LABELS_FILE}")    print(f"    → Continuing without labels. You'll get numeric codes instead of text.")    LABELS_FILE = Noneelse:    print(f"  ⚠ No labels file specified. Continuing without labels.")if all_good:    print("\nAll required files found. Ready to proceed.")else:    print("\n⛔ Fix the missing files above before continuing.")
```

## Step 3: Convert the FIMS File

Here's something the PSID doesn't make obvious: when you download an extract, the CSV data file has generic column names like V1, V2, V3. Those are just column positions — they're not the actual PSID variable names.

The FIMS file (Family Identification Mapping System) is the Rosetta Stone. It tells you that, say, column 1 is actually ER30000, column 2 is ER30001, and so on.

The FIMS file usually arrives as an Excel file (.xlsx). We need to:
1. 
Read it
2. 
Extract the mapping of column positions → variable names
3. 
Use that mapping to rename the columns in our data

If your FIMS file is already a CSV, this cell handles that too.

```
# ── Read the FIMS file ──────────────────────────────────────if FIMS_FILE.endswith('.xlsx') or FIMS_FILE.endswith('.xls'):    print(f"Reading FIMS as Excel file: {FIMS_FILE}")    fims_raw = pd.read_excel(FIMS_FILE)else:    print(f"Reading FIMS as CSV file: {FIMS_FILE}")    fims_raw = pd.read_csv(FIMS_FILE)print(f"FIMS file has {len(fims_raw)} rows and {len(fims_raw.columns)} columns.")print(f"\nColumn names in the FIMS file:")print(list(fims_raw.columns))print(f"\nFirst 5 rows:")fims_raw.head()
```

### Understanding the FIMS File

The FIMS file typically has columns like name (or NAME) and col_num (or similar). The key information is:
- 
Variable name — the actual PSID variable ID (like V103, ER30001)
- 
Column number — which column position in the extract CSV this variable occupies

The cell below auto-detects which columns contain this mapping. If the auto-detection fails, you'll get instructions for how to fix it manually.

```
# ── Build the column-name mapping from FIMS ─────────────────# Auto-detect the variable name column and column number columnfims_cols = [c.lower().strip() for c in fims_raw.columns]# Try common FIMS column name patternsname_col = Nonefor candidate in ['name', 'variable', 'var_name', 'varname', 'variable_name']:    matches = [i for i, c in enumerate(fims_cols) if candidate in c]    if matches:        name_col = fims_raw.columns[matches[0]]        break# If no name column found, use the first column that has string values# looking like PSID variable names (starting with V or ER)if name_col is None:    for col in fims_raw.columns:        sample = fims_raw[col].dropna().astype(str).head(10)        if any(s.startswith(('V', 'ER', 'v', 'er')) for s in sample):            name_col = col            breakif name_col is None:    print("⛔ Could not auto-detect the variable name column in FIMS.")    print("   Please set name_col manually in this cell.")    print(f"   Available columns: {list(fims_raw.columns)}")else:    print(f"Variable name column detected: '{name_col}'")# Extract the variable names in order# The FIMS file lists variables in the same order as the extract columnsvar_names = fims_raw[name_col].astype(str).str.strip().tolist()print(f"\nFound {len(var_names)} variables in the FIMS file.")print(f"First 10: {var_names[:10]}")print(f"Last 5:   {var_names[-5:]}")
```

## Step 4: Load the Extract and Apply Variable Names

Now we load the actual data. Remember — the CSV from the Data Center has meaningless column headers. We'll replace them with the real PSID variable names from the FIMS mapping.

After this cell, your DataFrame columns will be named things like ER30000, V103, ER30001 — still cryptic, but at least they're the real variable IDs that you can look up in the PSID codebook.

```
# ── Load the extract CSV ────────────────────────────────────print(f"Loading extract: {EXTRACT_FILE}")df = pd.read_csv(EXTRACT_FILE)print(f"Raw extract: {df.shape[0]:,} rows × {df.shape[1]:,} columns")# ── Rename columns using FIMS mapping ───────────────────────if len(var_names) == len(df.columns):    df.columns = var_names    print(f"\n✓ Columns renamed successfully using FIMS mapping.")elif len(var_names) > len(df.columns):    # Sometimes FIMS has more entries than extract columns    df.columns = var_names[:len(df.columns)]    print(f"\n⚠ FIMS has {len(var_names)} names but extract has {len(df.columns)} columns.")    print(f"  Using first {len(df.columns)} names from FIMS.")else:    # More columns than names — unusual, but handle it    padded = var_names + [f"UNKNOWN_{i}" for i in range(len(df.columns) - len(var_names))]    df.columns = padded    print(f"\n⚠ Extract has {len(df.columns)} columns but FIMS only has {len(var_names)} names.")    print(f"  Extra columns labeled as UNKNOWN_0, UNKNOWN_1, etc.")print(f"\nYour variables:")print(list(df.columns))print(f"\nFirst 3 rows:")df.head(3)
```

## Step 5: Build Person IDs

This is one of the trickiest parts of working with PSID data, and one of the most important.

The PSID tracks people using two numbers:
- 
ER30000 — the family interview number (identifies the family)
- 
ER30001 — the person number within that family

Neither one alone is unique. Family 1234 might have person 1, person 2, person 3. And person number 1 exists in thousands of families. But the combination — 1234_001 — uniquely identifies one human being across the entire study.

We'll create a person_id column by concatenating these two numbers. This is the key that lets you track the same person across decades of data.

```
# ── Build person_id from ER30000 + ER30001 ──────────────────# Check that the required variables existid_vars = ['ER30000', 'ER30001']missing_id_vars = [v for v in id_vars if v not in df.columns]if missing_id_vars:    print(f"⛔ Missing ID variables: {missing_id_vars}")    print(f"   Your extract must include ER30000 and ER30001.")    print(f"   Go back to the PSID Data Center and add them to your extract.")else:    # Zero-pad the person number to 3 digits for consistent formatting    # Family 1234, person 1 → "1234_001"    df['person_id'] = (        df['ER30000'].astype(str) + '_' +        df['ER30001'].astype(str).str.zfill(3)    )    n_unique = df['person_id'].nunique()    print(f"✓ person_id created.")    print(f"  {df.shape[0]:,} rows → {n_unique:,} unique person IDs")    if n_unique < df.shape[0]:        n_dupes = df.shape[0] - n_unique        print(f"  ⚠ {n_dupes:,} duplicate person_ids detected.")        print(f"    This is normal if your extract spans multiple waves.")    print(f"\nSample person_ids: {df['person_id'].head(5).tolist()}")
```

## Step 6: Filter by Sample Status

Not everyone in a PSID extract is an actual PSID sample member. The variable ER32006 tells you each person's sample status:

For most analyses, you want codes 1 through 4 — these are the people the PSID was designed to be representative of. Codes 5+ include non-sample members and special sub-samples that require different weighting.

If your extract doesn't include ER32006, we'll skip this filter and keep everyone.

```
# ── Filter by sample status (ER32006) ───────────────────────SAMPLE_CODES = [1, 2, 3, 4]  # Standard PSID sample membersif 'ER32006' in df.columns:    before = len(df)    # Show the distribution before filtering    print("Sample status distribution (ER32006):")    print(df['ER32006'].value_counts().sort_index().to_string())    # Save pre-filter version if requested    if SAVE_PRE_FILTER:        df.to_csv(PRE_FILTER_FILE, index=False)        print(f"\n✓ Pre-filter data saved: {PRE_FILTER_FILE} ({before:,} rows)")    # Apply the filter    df = df[df['ER32006'].isin(SAMPLE_CODES)].copy()    after = len(df)    dropped = before - after    print(f"\nFiltered to sample codes {SAMPLE_CODES}:")    print(f"  Before: {before:,} rows")    print(f"  After:  {after:,} rows")    print(f"  Dropped: {dropped:,} non-sample rows")else:    print("⚠ ER32006 (sample status) not found in your extract.")    print("  Skipping sample filter — all rows retained.")    print("  Consider adding ER32006 to your extract for future use.")    if SAVE_PRE_FILTER:        df.to_csv(PRE_FILTER_FILE, index=False)        print(f"\n✓ Pre-filter data saved: {PRE_FILTER_FILE} ({len(df):,} rows)")
```

## Step 7: Merge Parent–Child Links

One of the most powerful features of the PSID is its generational structure. Because the study follows families, you can link parents to children — and sometimes grandparents to grandchildren.

The variable ER32000 contains the person number of the individual's 1968 family head. Combined with ER30000, this lets you trace who belongs to which original family lineage.

For direct parent–child linking, you'll also want ER30002 (relation to head). If your extract includes it, we can identify who is a head, who is a spouse, and who is a child within each family unit.

This step creates a family_lineage_id that groups people by their original 1968 family, enabling intergenerational analysis.

```
# ── Build family lineage links ──────────────────────────────if 'ER32000' in df.columns:    # ER32000 = 1968 interview number of the family this person descends from    # This links everyone back to an original 1968 PSID family.    df['family_lineage_id'] = df['ER32000'].astype(str)    n_lineages = df['family_lineage_id'].nunique()    print(f"✓ family_lineage_id created from ER32000.")    print(f"  {n_lineages:,} unique family lineages in your data.")else:    print("⚠ ER32000 not found. Cannot build family lineage links.")    print("  Consider adding ER32000 to your extract.")# ── Identify family roles (if ER30002 is available) ─────────if 'ER30002' in df.columns:    role_map = {        1: 'Head',        2: 'Spouse/Partner',        3: 'Child',        # Codes 4+ vary by wave; these are the most common    }    df['family_role'] = df['ER30002'].map(role_map).fillna('Other')    print(f"\nFamily role distribution (ER30002):")    print(df['family_role'].value_counts().to_string())else:    print("\n⚠ ER30002 (relation to head) not found.")    print("  Cannot assign family roles.")
```

## Step 8: Apply Human-Readable Labels

At this point your data is structurally correct, but the values are still numeric codes. The variable V103 might contain 1, 5, and 8 — but unless you memorize the codebook, you won't know those mean Owns, Rents, and Neither.

The labels.txt file from the Data Center contains these mappings. This step reads that file and creates labeled versions of your coded variables.

We don't replace the original numeric values — we add new columns alongside them with a _label suffix. That way you keep the numbers for computation and the text for readability.

If you didn't download a labels file, no problem — just skip this cell. You can always look up codes in the PSID Cross-Year Index.

```
# ── Parse and apply labels ──────────────────────────────────labels_applied = 0if LABELS_FILE and os.path.exists(LABELS_FILE):    print(f"Reading labels from: {LABELS_FILE}")    # Parse the labels.txt file    # Format is typically:    #   VARIABLE_NAME    #     code  label    #     code  label    #   (blank line)    #   NEXT_VARIABLE    label_dict = {}  # {variable_name: {code: label_text}}    current_var = None    with open(LABELS_FILE, 'r', encoding='utf-8', errors='replace') as f:        for line in f:            line = line.rstrip()            if not line.strip():                current_var = None                continue            # Check if this is a variable name (no leading whitespace,            # starts with V or ER, or is all caps)            if not line.startswith((' ', '\t')) and line.strip():                candidate = line.strip().split()[0]                if candidate in df.columns:                    current_var = candidate                    label_dict[current_var] = {}                else:                    current_var = None            elif current_var and line.strip():                # This is a code-label pair                parts = line.strip().split(None, 1)                if len(parts) == 2:                    try:                        code = int(parts[0])                        label_text = parts[1].strip()                        label_dict[current_var][code] = label_text                    except ValueError:                        pass  # Not a numeric code line    # Apply labels to matching columns    for var_name, code_map in label_dict.items():        if var_name in df.columns and code_map:            df[f"{var_name}_label"] = df[var_name].map(code_map)            labels_applied += 1    print(f"\n✓ Labels applied to {labels_applied} variables.")    if labels_applied > 0:        label_cols = [c for c in df.columns if c.endswith('_label')]        print(f"  New label columns: {label_cols}")        # Show a sample        if label_cols:            sample_col = label_cols[0]            orig_col = sample_col.replace('_label', '')            print(f"\n  Sample — {orig_col} vs {sample_col}:")            print(df[[orig_col, sample_col]].drop_duplicates().head(10).to_string(index=False))    else:        print("  No matching variables found between labels file and your extract.")        print("  This might mean the labels file format wasn't recognized.")        print("  You can still use the numeric codes — check the PSID codebook for meanings.")else:    print("No labels file — skipping. All values remain as numeric codes.")    print("You can look up variable codes at: https://simba.isr.umich.edu/VS/s.aspx")
```

## Step 9: Inspect Your Clean Data

Before we export, let's take a look at what we've built. This is your chance to sanity-check the data:
- 
Are the variable names correct?
- 
Do the row counts make sense?
- 
Are person_ids unique (or appropriately duplicated for multi-wave data)?
- 
Do the labels look right?

If anything looks off, you can go back and adjust earlier cells.

```
# ── Data summary ────────────────────────────────────────────print("="*60)print("CLEAN DATA SUMMARY")print("="*60)print(f"Rows:        {df.shape[0]:,}")print(f"Columns:     {df.shape[1]:,}")if 'person_id' in df.columns:    print(f"Unique IDs:  {df['person_id'].nunique():,}")if 'family_lineage_id' in df.columns:    print(f"Lineages:    {df['family_lineage_id'].nunique():,}")print(f"\nAll columns:")for i, col in enumerate(df.columns):    dtype = df[col].dtype    n_missing = df[col].isna().sum()    missing_pct = (n_missing / len(df)) * 100    print(f"  {i+1:3d}. {col:<25s}  dtype={str(dtype):<10s}  missing={n_missing:,} ({missing_pct:.1f}%)")print(f"\nFirst 5 rows:")df.head()
```

## Step 10: Export Clean CSV

That's it. Your PSID extract has been:
- 
✓ Loaded and column-named using the FIMS mapping
- 
✓ Assigned unique person IDs
- 
✓ Filtered to valid PSID sample members
- 
✓ Linked by family lineage
- 
✓ Labeled with human-readable value descriptions

The cell below saves your clean data as a CSV. Open it in Excel, R, Stata, or read it back into Python with pd.read_csv() — it's ready for whatever analysis you have in mind.

```
# ── Save the clean data ─────────────────────────────────────df.to_csv(OUTPUT_FILE, index=False)output_size = os.path.getsize(OUTPUT_FILE) / (1024 * 1024)print(f"✓ Clean data saved: {OUTPUT_FILE}")print(f"  {df.shape[0]:,} rows × {df.shape[1]:,} columns")print(f"  File size: {output_size:.1f} MB")if SAVE_PRE_FILTER and os.path.exists(PRE_FILTER_FILE):    pre_size = os.path.getsize(PRE_FILTER_FILE) / (1024 * 1024)    print(f"\n✓ Pre-filter data also saved: {PRE_FILTER_FILE}")    print(f"  File size: {pre_size:.1f} MB")print(f"\n" + "="*60)print("DONE. Your data is ready for analysis.")print("="*60)
```

## What Next?

You now have a clean, labeled CSV with person IDs and family lineage links. Here are some things you might want to do with it:
- 
Explore your variables — look at distributions, cross-tabulations, missing data patterns
- 
Merge across waves — if your extract includes variables from multiple years, you can reshape the data from wide to long format for panel analysis
- 
Link generations — use the family_lineage_id to compare parents and children on the same outcomes
- 
Add more variables — go back to the Data Center, add variables to your extract, re-download, and run this notebook again

The PSID is one of the longest-running and richest longitudinal studies in the world. You've just taken the hardest step — getting the data clean and usable. The analysis is the fun part.

This notebook is part of the PSID Book Project. For the full analysis pipeline — from data preparation through regression and visualization — see the companion chapters and notebooks.
