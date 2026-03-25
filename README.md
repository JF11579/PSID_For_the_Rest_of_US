PSID For the Rest of Us 
Does homeownership change a family’s trajectory? The data says yes. 
This repository contains the complete set of Jupyter notebooks behind PSID For the Rest of Us, a book that walks through the Panel Study of Income
Dynamics – one of the most powerful and least accessible datasets in the social sciences. 
The project links 1968 homeownership to children’s educational outcomes measured decades later. Kids whose parents owned their home in 1968
completed about one extra year of schooling. That finding survived every robustness check we threw at it. 
But the finding is not the point. The point is the pipeline. These notebooks show you how to go from a raw PSID extract to a publishable analysis, step by
step, with every decision explained. 
 The NotebooksNotebook What It DoesNotebook_01_2_Prepare_Dataset Takes a raw PSID extract and a FIMS genealogy file and produces a clean, labeled, analysis-ready CSV. Builds
person IDs, filters to sample members, recodes variables, and links parents to children.Notebook_02_Main_Analysis Runs OLS regressions for education and linear probability models for homeownership. Produces plain
English results tables and figures. This is where the main findings come from.Notebook_Regression_Report Generates the full regression report – every coefficient, every model, summary statistics, transition
matrices, and robustness checks in one place.03_Exploration_Robustness_CORRECTED Tests whether the findings hold up across race, sex, birth cohort, and sample composition. Includes
subgroup analyses and full-sample comparisons.Three_Generation_Family_Explorer_v4 An interactive tool that visualizes real three-generation family trees – grandparent, parent, grandchild –
showing how homeownership and education flow across generations. Run in Google Colab for the
interactive widgets.Universal_PSID_Prep_Tool A general-purpose version of the data preparation pipeline. Point it at any PSID extract and FIMS file, and it
will rename columns, build IDs, filter, label, and export a clean CSV – for any research question, not just this
one. 
 How to Use These Notebooks 
All notebooks are designed to run in Google Colab. Upload a notebook, connect your Google Drive with the data files, and run. 
What you need 
A PSID data extract from the PSID Data Center. Registration is free. The main restriction is that you do not attempt to identify individuals.
The FIMS genealogy file for your extract (downloaded alongside the data).
A Google Drive folder to store the data files. 
Getting started 
If you want to replicate this project’s analysis, start with Notebook 01.2 (Prepare Dataset), then run Notebook 02 (Main Analysis). The other notebooks
build on the output of these two. 
If you want to run your own analysis on different PSID variables, start with the Universal PSID Prep Tool. It generalizes the pipeline so you can skip the
variable-specific setup. 
The book walks through every line of code in detail. If something in a notebook is unclear, the corresponding chapter will explain it. 
 The Book 
The full text is at joe-foley.gitbook.io/psid-book 
Chapters cover: why the PSID matters, how to find variables in the Cross-Year Index, how the data preparation pipeline works, the main regression
analysis, robustness checks, three-generation family trees, and the full regression report. The conclusion explains how to take the tools in this repo and
apply them to your own question. 
 Key Findings 
Children whose parents owned their home in 1968 completed about one extra year of education (0.99 years, p < 0.001).
After controlling for parent education, income, race, and sex, the effect is +0.34 years – still significant.
Children of homeowners were 21.5 percentage points more likely to own a home themselves by 2023. After controls, +12.5 percentage points (p
< 0.001).
Black children were 18.9 percentage points less likely to own a home than White children, holding all else constant – pointing to structural
barriers beyond income.
The homeownership-education effect is fading across birth cohorts: 2.16 years for the 1940s cohort, 0.66 years for the 1960s cohort. All still
significant, but shrinking.
Results are robust to state fixed effects, sample composition, and subgroup analysis. Data 
The PSID data itself is not included in this repository. The PSID requires users to register and agree to terms of use that prohibit redistribution. You can
get the data for free at simba.isr.umich.edu. 
The specific extract used in this project contains 774 variables. The codebook reference is J358642. 
 License 
Text and narrative content © 2026 Joe Foley. All rights reserved. 
Notebook code is released under the MIT License – use it, adapt it, build on it. 
 Contact 
Questions, corrections, or ideas for extending this work? Reach me at Jfoley648@gmail.com or open an issue on this repo
