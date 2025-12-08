# Machine Learning-Assisted Discovery of Novel Cycloaddition Reactions

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![DOI](https://img.shields.io/badge/DOI-10.1002%2Fanie.XXXXXXXX-blue)](https://doi.org/10.1002/anie.XXXXXXXX)

> **Automated pipeline for discovering new chemical transformations by combining rule-based generation, quantum chemistry, and unsupervised machine learning.**

This repository contains code and data for the manuscript:  
**"Machine Learning-Assisted Discovery of Novel Cycloaddition Reactions"**  
*Nikita I. Kolomoets†, Daniil A. Boiko†, Leonid V. Romashov, Kirill S. Kozlov, Evgeniy G. Gordeev, Alexey S. Galushko, Valentine P. Ananikov\**  
Published in *Angewandte Chemie International Edition* (2025)

---

## 🎯 Overview

The search for new chemical transformations is a fundamental challenge in modern chemistry with broad implications for drug discovery, materials science, and sustainable synthesis. This work presents an **automated discovery pipeline** that:

1. **Generates 31,000+ cycloaddition reaction candidates** from the QM9 molecular database using rule-based templates
2. **Filters thermodynamically favorable reactions** (ΔrG < −10 kcal/mol) using pre-computed molecular energies
3. **Clusters reactions** into 13 structurally distinct families using transformer-based embeddings
4. **Prioritizes candidates** through expert evaluation and DFT-based substituent optimization
5. **Validates experimentally**, resulting in **2 novel cycloaddition reactions**

🌐 **Explore the dataset interactively:** [http://app.ananikovlab.ai:8510](http://app.ananikovlab.ai:8510)

---

## 📁 Repository Structure

\`\`\`
.
├── mining_pubchem/              # Reaction generation from QM9 database
│   ├── generate_templates.py   # Create cycloaddition reaction templates
│   ├── processing_dataset.py   # Parse QM9 dataset and extract molecules
│   ├── reaction_find.sh         # Parallel reaction mining script
│   └── reacts_concatenation.py # Combine generated reactions
│
├── clustering_reactions/        # Unsupervised learning pipeline
│   ├── smi2vec.py              # SMILES → embeddings (rxnfp transformer)
│   ├── dimensionality_reduction.py  # PCA/t-SNE/UMAP for visualization
│   └── cluster_reactions.py    # k-means/hierarchical clustering
│
├── creation_reaction_cards/     # Expert evaluation tools
│   ├── filter_reactions_by_energy.py  # Energy-based sampling
│   └── parse_manual_labels.py  # Process expert annotations
│
├── laboratory_database_of_reagents/  # Experimental reagent filtering
│   └── get_substituents.py     # Extract alkyne substituents from lab DB
│
├── generate_computation/        # DFT workflow automation
│   ├── generate_mopac_smiles.py  # Create product structures
│   ├── mopac_generate.py       # PM7 pre-optimization (MOPAC)
│   ├── gaussian_generate.py    # Generate Gaussian input files
│   ├── withdrawal_energy.py    # Parse DFT energies from Gaussian logs
│   └── prop_rxns.py           # Compile reaction energies into CSV
│
├── final_filter/               # Post-DFT prioritization
│   └── fine_filter_reactions_by_energy.py  # Select top candidates
│
└── working_files/              # Data directory (not included in repo)
    ├── dsgdb9nsd.xyz.tar.bz2  # QM9 dataset (download separately)
    ├── reactions.pkl          # Generated reactions
    ├── embeds_reacts.pkl      # Reaction embeddings
    └── ...
\`\`\`

---

## 🚀 Quick Start

### Prerequisites

- Python 3.8+
- [RDKit](https://www.rdkit.org/) (chemistry toolkit)
- [rxnfp](https://github.com/rxn4chemistry/rxnfp) (reaction fingerprints)
- MOPAC2016 (for PM7 calculations)
- Gaussian 16 (for DFT calculations)

### Installation

\`\`\`bash
# Clone repository
git clone https://github.com/Ananikov-Lab/Digitcal-co-Expert.git
cd Digitcal-co-Expert

# Install Python dependencies
pip install -r requirements.txt

# Download QM9 dataset (134k molecules, ~3 GB)
wget https://figshare.com/ndownloader/files/3195389 -O working_files/dsgdb9nsd.xyz.tar.bz2
\`\`\`

---

## 📊 Reproducing the Results

### Step 1: Reaction Generation (31k candidates)

Generate cycloaddition templates and mine the QM9 database:

\`\`\`bash
cd mining_pubchem

# 1. Create reaction templates
python generate_templates.py --output ../working_files/templates.pkl

# 2. Process QM9 dataset and annotate with CAS numbers
python processing_dataset.py \\
    --ds-path ../working_files/dsgdb9nsd.xyz.tar.bz2 \\
    --db-path ../working_files/cas \\
    --db-name 'CAS' \\
    --n-jobs 16 \\
    --output-dir ../working_files/dataset_split/ \\
    --output ../working_files/proc_dataset.pkl

# 3. Mine reactions in parallel (16 processes)
sh reaction_find.sh \\
    16 \\
    ../working_files/proc_dataset.pkl \\
    ../working_files/dataset_split/ \\
    ../working_files/templates.pkl \\
    ../working_files/reactions_dir/ \\
    'CAS'

# 4. Concatenate results
python reacts_concatenation.py \\
    --input ../working_files/reactions_dir/ \\
    --output ../working_files/reactions.pkl \\
    --output-rs ../working_files/embeds/smiles_reags.txt \\
    --output-ps ../working_files/embeds/smiles_prods.txt
\`\`\`

**Output:** \`reactions.pkl\` (31,000 generated reactions with QM9-derived energies)

---

### Step 2: Unsupervised Clustering (13 clusters)

Convert SMILES to embeddings and cluster reactions:

\`\`\`bash
cd clustering_reactions

# 1. Generate reaction embeddings using pre-trained transformer
python smi2vec.py \\
    --p-vocab ../working_files/vocab.pkl \\
    --p-trfm ../working_files/trfm.pkl \\
    --smi ../working_files/embeds/smiles_reags.txt \\
    --output ../working_files/embeds/reags.npy &

python smi2vec.py \\
    --p-vocab ../working_files/vocab.pkl \\
    --p-trfm ../working_files/trfm.pkl \\
    --smi ../working_files/embeds/smiles_prods.txt \\
    --output ../working_files/embeds/prods.npy

# 2. Dimensionality reduction for visualization
python dimensionality_reduction.py \\
    --emb-r ../working_files/embeds/reags.npy \\
    --emb-p ../working_files/embeds/prods.npy \\
    --smi-r ../working_files/embeds/smiles_reags.txt \\
    --smi-p ../working_files/embeds/smiles_prods.txt \\
    --reacts ../working_files/reactions.pkl \\
    --method 't-SNE' \\
    --output ../working_files/embeds_reacts.pkl \\
    --n_components 2 \\
    --perplexity 100

# 3. Cluster reactions (k-means, 13 clusters)
python cluster_reactions.py \\
    --input ../working_files/embeds_reacts.pkl \\
    --method 'KMeans' \\
    --metric 'euclidean' \\
    --plot ../working_files/clusters.png \\
    --model ../working_files/cluster_model.pkl \\
    --n_clusters 13
\`\`\`

**Output:** \`cluster_model.pkl\` (clustering assignments for 29k reactions)

---

### Step 3: Expert Evaluation (205 candidates)

Sample up to 18 reactions per cluster for manual assessment:

\`\`\`bash
cd creation_reaction_cards

# 1. Filter by energy and CAS availability
python filter_reactions_by_energy.py \\
    --reactions ../working_files/reactions.pkl \\
    --db-name 'CAS' \\
    --model ../working_files/cluster_model.pkl \\
    --number 18 \\
    --output ../working_files/candidates_205.pkl \\
    --output-numbers ../working_files/reaction_ids.pkl

# 2. Parse expert annotations (after manual evaluation)
python parse_manual_labels.py \\
    --archive ../working_files/expert_evaluations.zip \\
    --output ../working_files/expert_labels/ \\
    --numbers ../working_files/reaction_ids.pkl \\
    --csv ../working_files/expert_data.csv
\`\`\`

**Output:** \`expert_data.csv\` (205 reactions with feasibility scores and synthetic utility ratings)

---

### Step 4: Substituent Optimization (9 priority candidates)

Generate product structures with laboratory-available alkynes and optimize with DFT:

\`\`\`bash
cd laboratory_database_of_reagents

# 1. Extract alkyne substituents from lab inventory
python get_substituents.py \\
    --input-db ../working_files/Lab_Reagents.sdf \\
    --output-db ../working_files/alkynes_lab.txt

cd ../generate_computation

# 2. Generate product SMILES with alkyne substituents
python generate_mopac_smiles.py \\
    ../working_files/priority_reactions.pkl \\
    ../working_files/alkynes_lab.txt \\
    ../working_files/products_to_optimize.txt

# 3. Create MOPAC input files (PM7 pre-optimization)
python mopac_generate.py \\
    ../working_files/products_to_optimize.txt \\
    ../working_files/mopac_inputs/

# (Run MOPAC calculations manually or via cluster scheduler)

# 4. Generate Gaussian input files (B3LYP/6-31G(2df,p))
python gaussian_generate.py \\
    ../working_files/mopac_outputs/ \\
    ../working_files/gaussian_check/ \\
    16000 \\
    'B3LYP/6-31G(2df,p)' \\
    16 \\
    ../working_files/gaussian_inputs/

# (Run Gaussian calculations)

# 5. Extract energies from Gaussian log files
python withdrawal_energy.py \\
    ../working_files/gaussian_outputs/ \\
    ../working_files/products_to_optimize.txt \\
    ../working_files/product_energies.txt

# 6. Compile reaction energies into table
python prop_rxns.py \\
    ../working_files/reagent_energies.txt \\
    ../working_files/alkyne_energies.txt \\
    ../working_files/product_energies.txt \\
    ../working_files/dft_reactions.csv
\`\`\`

**Output:** \`dft_reactions.csv\` (optimized reaction energies for experimental screening)

---

### Step 5: Final Prioritization

Select top candidates for synthesis:

\`\`\`bash
cd final_filter

python fine_filter_reactions_by_energy.py \\
    --input-csv ../working_files/dft_reactions.csv \\
    --number 10 \\
    --output ../working_files/final_candidates.csv
\`\`\`

**Output:** 10 reactions for high-throughput GC-MS screening → **2 experimentally validated novel cycloadditions**

---

## 📦 Datasets and Interactive Interface

### Download Datasets

All computational data are available at [http://app.ananikovlab.ai:8510](http://app.ananikovlab.ai:8510):

- **Full reaction dataset** (CSV, 31k reactions): Reaction SMILES, ΔrG from QM9, CAS numbers, functional group annotations
- **Reaction embeddings** (PKL, 768D vectors): Transformer-based features for clustering
- **Expert evaluation template** (PDF): Printable form for manual assessment

### Web Interface Features

- Browse 31,000 reactions with sortable/filterable table
- Interactive 2D projection (t-SNE) of reaction space
- Filter by energy, heteroatoms, CAS availability
- Export subsets in CSV format

---

## 🔧 Adapting to Other Reaction Types

To apply this pipeline to different reaction classes:

1. **Modify template generation** (\`mining_pubchem/generate_templates.py\`):
   - Define new reaction SMARTS patterns
   - Example: For Diels-Alder, use \`[C:1]=[C:2]-[C:3]=[C:4].[C:5]=[C:6]>>[C:1]1-[C:2]=[C:3]-[C:4]-[C:5]-[C:6]-1\`

2. **Update molecular database** (replace QM9 with your dataset):
   - Format: XYZ files with energy annotations
   - Ensure energies are in Hartree or kcal/mol

3. **Adjust clustering parameters** (\`clustering_reactions/cluster_reactions.py\`):
   - Tune \`n_clusters\` based on chemical diversity

4. **Customize expert evaluation criteria** (\`creation_reaction_cards/\`):
   - Modify assessment form to match your domain

---

## 📄 Citation

If you use this code or data, please cite:

\`\`\`bibtex
@article{kolomoets2025cycloaddition,
  title={Machine Learning-Assisted Discovery of Novel Cycloaddition Reactions},
  author={Kolomoets, Nikita I. and Boiko, Daniil A. and Romashov, Leonid V. and Kozlov, Kirill S. and Gordeev, Evgeniy G. and Galushko, Alexey S. and Ananikov, Valentine P.},
  journal={Angewandte Chemie International Edition},
  year={2025},
  publisher={Wiley-VCH},
  doi={10.1002/anie.XXXXXXXX}
}
\`\`\`

**Web interface:** Kolomoets, N. I. et al. (2025). *Cycloaddition Reaction Database*. [http://app.ananikovlab.ai:8510](http://app.ananikovlab.ai:8510)

---

## 📬 Contact

- **Corresponding author:** Valentine P. Ananikov ([val@ioc.ac.ru](mailto:val@ioc.ac.ru))
- **Issues and questions:** [GitHub Issues](https://github.com/Ananikov-Lab/Digitcal-co-Expert/issues)
- **Lab website:** [ananikovlab.ru](https://ananikovlab.ru)

---

## 📜 License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- QM9 dataset: [Ramakrishnan et al., *Sci Data* 2014](https://doi.org/10.1038/sdata.2014.22)
- rxnfp transformer: [Schwaller et al., *Nat Mach Intell* 2021](https://doi.org/10.1038/s42256-020-00284-w)
- Funding: Russian Science Foundation (grant XX-XX-XXXXX)

---

**Last updated:** December 2025
