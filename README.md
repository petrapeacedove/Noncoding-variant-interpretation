Non-Coding Variant Interpretation

Predicting the pathogenicity of non-coding genetic variants using evolutionary conservation, transcription factor motif disruption, and Mendelian disease-gene context.

Abstract

Non-coding genetic variants account for a significant fraction of clinically observed mutations but remain poorly interpreted compared to coding variants, contributing to diagnostic uncertainty in rare Mendelian diseases. This project builds a binary classifier to predict the pathogenicity of non-coding single nucleotide variants (SNVs) using evolutionary conservation (phyloP, phastCons), transcription factor motif disruption, and Mendelian disease-gene relevance as features. Using a balanced dataset of 44,748 ClinVar-derived variants, a logistic regression model achieved 0.916 ROC-AUC, with SHAP analysis confirming evolutionary conservation as the dominant predictive signal — consistent with established variant interpretation biology.

Motivation / Background

Approximately 98% of the human genome is non-coding, yet the majority of variant interpretation tools and clinical pipelines remain optimized for coding regions, where the effect of a mutation on protein structure is relatively direct to assess. Non-coding variants can disrupt gene regulation — altering promoter activity, transcription factor binding, or transcript stability — without changing the protein sequence itself. This regulatory disruption can be equally pathogenic, particularly in Mendelian and rare diseases, but remains substantially harder to interpret. A large fraction of non-coding ClinVar submissions carry "Uncertain Significance" labels, reflecting this diagnostic gap. This project builds a lightweight, interpretable framework to help close that gap using biologically-grounded features rather than black-box sequence modeling alone.

Workflow

flowchart TD
    A[ClinVar VCF - GRCh38] --> B[Filter: non-coding consequence + confident label]
    B --> C[Filter: SNVs only, indels excluded]
    C --> D[Balance dataset - 1:1 undersampling]
    D --> E[Feature engineering]
    E --> E1[phyloP / phastCons - UCSC conservation tracks]
    E --> E2[Motif disruption score - JASPAR, 10-TF shortlist]
    E --> E3[Mendelian gene flag - HPO gene-disease associations]
    E1 --> F[Train/test split - 80/20, stratified]
    E2 --> F
    E3 --> F
    F --> G[Models: Logistic Regression | Random Forest]
    G --> H[Evaluation: ROC-AUC, McNemar's test, SHAP interpretability]

Dataset & Methods

Data source: ClinVar VCF (GRCh38 build), NCBI, accessed July 2026.

Filtering pipeline:

StepFilterVariants remaining1Non-coding consequence + confident clinical label (Pathogenic/Likely Pathogenic vs Benign/Likely Benign)643,4262SNVs only (indels excluded)558,2883Drop missing conservation scores558,2534Balanced via undersampling (1:1 pathogenic:benign)44,748

Class imbalance note: restricting to SNVs increased the benign:pathogenic ratio from ~13:1 to ~24:1, since indels are disproportionately pathogenic in non-coding regions — a documented tradeoff of this scoping decision.

Features engineered:


phyloP — per-base evolutionary conservation score (UCSC 100-way vertebrate alignment)
phastCons — per-base conservation probability (UCSC 100-way vertebrate alignment)
motif_disruption — maximum predicted transcription factor binding disruption across a 10-TF shortlist (SP1, CTCF, GATA1, NFKB1, TP53, MYC, STAT3, YY1, EGR1, FOXA1), scored via simplified PWM matching against JASPAR 2024 motifs
is_mendelian_gene — binary flag for genes with curated Mendelian disease associations (HPO genes_to_disease.txt); 94% of the balanced dataset overlapped this list


Models: Logistic Regression (scaled features) and Random Forest (200 trees, max depth 10), 80/20 stratified train-test split.

Results

ModelAccuracyROC-AUCConfusion Matrix (TN, FP / FN, TP)Logistic Regression0.860.916[3998, 477 / 802, 3673]Random Forest0.860.918[4013, 462 / 835, 3640]

McNemar's test comparing the two models: χ² = 1.70, p = 0.192 (not statistically significant) — indicating comparable performance despite Random Forest's added complexity. Logistic Regression is preferred as the primary model for its interpretability.

Feature importance (consistent across both models): phyloP >> phastCons > is_mendelian_gene ≈ motif_disruption.

Note on false negatives: both models missed ~18% of true pathogenic variants (802–835 false negatives out of 4,475). In a clinical screening context, this is the costlier error type — a direction for future threshold tuning is noted in Limitations.

Key Findings / Interpretability

SHAP (SHapley Additive exPlanations) analysis was used to interpret individual predictions, not just global feature importance. The SHAP summary plot confirmed phyloP as the dominant driver of predictions across the test set, with a clear separation: low-conservation variants pushed predictions toward benign, while high-conservation variants pushed toward pathogenic — directly reflecting the underlying evolutionary biology principle that functionally important genomic positions are conserved across species.

Example case (correctly classified pathogenic variant): phyloP = 7.53, phastCons = 1.0 → predicted 96.0% pathogenic probability, driven primarily by conservation signal (SHAP contribution: phyloP +3.23, phastCons +0.61).

Example case (correctly classified benign variant): phyloP = -0.52, phastCons = 0.0 → predicted 12.4% pathogenic probability, with both conservation features pulling toward benign (SHAP contribution: phyloP -1.82, phastCons -0.57).

Notably, the is_mendelian_gene feature occasionally worked against correct classification — in the pathogenic example above, the variant's gene was not in the curated Mendelian list, contributing a SHAP value of -1.02 toward benign despite the variant being genuinely pathogenic. This suggests the binary Mendelian-gene flag may be too coarse a signal; a continuous disease-relevance score could be a stronger feature in future iterations.

Limitations


Motif scoring is simplified: the PWM-based motif disruption score uses raw position-weight matrix summation rather than background-corrected log-odds scoring (as used in tools like motifbreakR), and covers only a 10-transcription-factor shortlist rather than the full JASPAR database. This likely explains its weak individual contribution to the model.
SNV-only scope: indels were excluded to keep feature engineering well-defined, which increased class imbalance in the source data (benign:pathogenic ratio rose from ~13:1 to ~24:1 after this filter), since pathogenic non-coding variants are disproportionately indels.
Undersampling tradeoff: balancing via undersampling discards a large majority of benign variants (~535,000 of ~558,000), which improves class balance but reduces the benign class's representational diversity.
False negative rate: both models missed ~18% of true pathogenic variants in testing — in a clinical deployment context, this would need threshold tuning toward higher recall, accepting more false positives as a tradeoff.
ClinVar label circularity: like most ClinVar-based studies, some submitted pathogenicity labels may themselves have been informed by conservation-based tools, introducing a potential circularity that inflates apparent conservation-score performance.


Installation / Quick-start

This project was built and run entirely in Google Colab. To reproduce:


Clone this repository:


bash   git clone https://github.com/petrapeacedove/Noncoding-variant-interpretation.git


Open notebooks/noncoding_variant_pathogenicity_classifier.ipynb in Google Colab.
Run cells sequentially. The notebook will download required public data automatically:

ClinVar VCF (NCBI)
phyloP / phastCons conservation tracks (UCSC, ~9GB each)
hg38 reference genome (UCSC, ~3GB)
JASPAR 2024 motifs (via pyjaspar)
HPO gene-disease associations


Note: large reference files (~20GB total) are downloaded fresh each session and are not stored in this repository. Expect the full pipeline to take 45–60 minutes end-to-end, primarily due to conservation score lookups and motif scanning across ~44,000–558,000 variants.
Pre-trained models are available in models/ (logistic_regression_model.pkl, random_forest_model.pkl, scaler.pkl) if you want to skip retraining and just explore predictions.


Dependencies: cyvcf2, pyBigWig, pyfaidx, pyjaspar, biopython, scikit-learn, shap, pandas, numpy, tqdm — all installable via pip install (see first cells of the notebook).

Repo Structure

noncoding-variant-interpretation/
├── README.md
├── notebooks/
│   └── noncoding_variant_pathogenicity_classifier.ipynb
├── data/
│   └── sample_filtered_variants.csv   # 500-row sample (full dataset not included, see Installation)
├── models/
│   ├── logistic_regression_model.pkl
│   ├── random_forest_model.pkl
│   └── scaler.pkl
├── results/
│   └── model_comparison.json
└── .gitignore

Future Work


Replace simplified PWM motif scoring with background-corrected log-odds scoring (e.g., following motifbreakR's approach) and expand beyond the 10-TF shortlist to the full JASPAR database
Incorporate chromatin accessibility (ATAC-seq) or Hi-C-derived enhancer-gene linkage data to extend beyond promoter/UTR/intronic variants to distal regulatory elements
Replace the binary is_mendelian_gene flag with a continuous disease-relevance score (e.g., gene-level constraint metrics like pLI/LOEUF, or HPO phenotype-similarity scores)
Explore threshold tuning to reduce the false negative rate for higher-recall clinical screening use cases
Extend to indels, which were excluded in this iteration but are disproportionately represented among pathogenic non-coding variants


Data Sources


ClinVar — NCBI
UCSC Genome Browser — phyloP100way, phastCons100way, hg38 reference
JASPAR — transcription factor binding motifs
Human Phenotype Ontology (HPO) — gene-disease associations


Author

Petra Peace Dove X
