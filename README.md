# biocoding
# Digital Twin Project (Control-Trajectory Forecaster for UPDRS-III Scores in Parkinson's Disease)

Minimal digital-twin prototype that predicts longitudinal patient UPDRS-III motor scores, if untreated, using baseline clinical and cerebrospinal fluid (CSF) proteomic data. 

## Project Overview

This project investigates whether a patient's baseline clinical and proteomic characteristics can be used to predict future disease trajectory.

Part III of the Unified Parkinson's Disease Rating Scale (UPDRS-III) is used as the primary longitudinal outcome, a clinical measure of motor symptom severity.

This was achieved through training a model on untreated/control patient data to forecast individual UPDRS-III score trajectory. 

This project is a research prototype and is not intended for clinical decision-making or patient diagnosis.

## Research Question

Can baseline clinical and CSF proteomic data be used to meaningfully predict individual longitudinal Parkinson's disease progression, measured through UPDRS-III scores?

## Dataset & Data Source Citation

- Zenodo - AMP-PD merged clinical and proteomics
  https://zenodo.org/records/14848598
  Longitudinal clinical visits (UPDRS scores, medication state) combined with baseline peptide abundance profiles for Parkinson's disease patients.

- Kaggle competition - AMP® Parkinson's Disease Progression Prediction
  https://www.kaggle.com/competitions/amp-parkinsons-disease-progression-prediction/data
  Original competition data (clinical visits, peptide measurements).

Training, testing, and evaluation of the model was conducted with a dataset created from merging these two datasets, ultimately containing:

- 248 patients 
- 1,068 longitudinal visits
- Up to 108 months (9 years) of follow-up
- Approximately 1,195 CSF proteomic features (peptides analysed through mass spectrometry)
- Longitudinal UPDRS Parts I-IV scores
- Medication-state (On/Off medication at time of visit)
- Baseline proteomic measurements
  
## Variables & Group Definition

Clinical Variables
- patient_id = patient identifier
- visit_id, visit_month, visit_year = visit timing
- updrs_1, updrs_2, updrs_3, updrs_4, updrs_total = UPDRS sub-scores and total
- upd23b_clinical_state_on_medication = medication state at visit ("On", "Off", or missing)

Outcome = updrs_3

Peptide Features
- All columns not in the clinical list are measured peptide abundances

Group Splits
- Patients classified by medication pattern:
  - Control = all recorded visits are "Off" medication
  - Unrecorded = no medication data recorded
  - Treated = at least one "On" visit
  - Joint = both "On" and "Off" visits recorded

For training the group-trajectory model:
 - Control group = control and unrecorded patients (used for training and testing)
 - Treated group = treated and joint patients (used for later evaluation)

Within control patients:
- 80/20 split into train/test
- Treated patients are kept separate (split = "treated")

## Repository Structure

.gitignore
README.md
digi-twin-public-release.ipynb
requirements.txt

## Environment & Dependencies

Python = 3.10+ recommended

Libraries = numpy, pandas, scipy, scikit-learn, statsmodels, patsy, matplotlib, seaborn

NB: print statements for output file saving use Windows-style backslashes and may appear differently on non-Windows systems. 

Setup:

conda create -n /env-name/ python=3.12 -y
conda activate /env-name/
conda install -c conda-forge numpy pandas scipy scikit-learn statsmodels patsy matplotlib seaborn jupyterlab ipykernel -y

No GPU required; model runs on CPU.

## Data Pre-processing

The script does as follows:

1. Filter visits with missing updrs_3
2. Classify patients into control, unrecorded, treated, joint based on upd23b_clinical_state_on_medication
3. Define cohorts:
   - Training patients = control + treated
   - Treated patients = treated + joint
4. Split control patients into train/test (80/20) at the patient level
5. Build baseline peptide matrix (first visit per patient)
6. Remove peptides with >30% missingness at baseline
7. impute remaining missing peptide values with median (per peptide)
8. standardise peptides (zero mean, unit variance)
9. Apply PCA (10 components) to baseline peptides
10. Construct modelling table (long format) with:
    - visit_month, visit_year, updrs_3, baseline_updrs_3
    - baseline_PC1-PC10
    - split (train/test/treated)
    - training_patient (yes/no)

## Modelling Approach

Mixed-Effects Model:
- Outcome = updrs_3
- Predictors = updrs_3 ~ visit_year + baseline_updrs_3 + PC1-PC5 + visit_year : baseline_updrs_3 
(first 5 PCs used in the formula; all 10 PCs available for analyses)
- Random Effects:
  - Primary = random intercept + random slope for visit_year per patient
  - Fallback = random intercept only if the random-slope model fails to converge 
- Fitting = statsmodels MixedLM (LBFGS, up to 2000 iterations)
- Forecasts = fixed-effect predictions 𝑋β for each visit

Feedforward Neural Network (MLP Regressor):
- Inputs = baseline_updrs_3, visit_year, baseline_PC1-PC10 (12 features)
- Architecture:
  - Hidden layers = [64, 32], ReLU activation
  - Regularisation = L2 (alpha=0.01), early stopping (validation fraction 0.15)
- Training = on control train patients; evaluated on control test and treated patients
- Pre-processing = inputs are standardised per feature

Evaluation Metrics:
- For each split (train, test, treated):
  - RMSE, MAE, R² between predicted and recorded updrs_3
  - 5-fold patient-level cross-validation (GroupKFold) for both models, reporting mean±SD of RMSE and MAE

## How to Run the Pipeline

1. Download digi-twin-public-release.ipynb
2. Open in JupyterLab
3. Lines that begin "# (˶˃ᆺ˂˶) !!!!!!!!!!" have dedicated space to insert file paths of the dataset and output directory, after which the beginning section should be removed.
4. The script will then:
   - Load provided dataset
   - Perform all pre-processing, modelling, and evaluation
   - Save outputs to chosen output directory 

Parameters such as "principeptiden" (principal component number) may be freely changed.

## Outputs

After running the pipeline your chosen output directory will contain:

Data files:
- digi-twin_prototype_LMT.csv = longitudinal modelling table containing all cleaned-up and created data columns for model training, testing, and evaluation
- dtp_model_setup.json = summary statistics, PCA variance, model performance, CV results
- dtpmodel_data.csv = long-format modelling table (one row per visit)
- baseline_data.csv = baseline visit per patient
- baseline_pca.csv = baseline PC scores for all patients

Figures:
- updrs3_trajectories.png = spaghetti plots of recorded UPDRS-III over time for control and treated patients
- updrs3_baseline_vs_followup.png = scatter of baseline vs. follow-up UPDRS-III scores; Pearson r reported
- baseline_PCA_scatter.png = PC1 vs PC2 scatter, coloured by split (train/test/treated)
- pvr_mvsf.png = predicted vs. recorded UPDRS-III for mixed-effects and feedforward neural network, across train/test/treated.
- sample_trajectories.png = recorded vs. predicted trajectories for a small sample of control test and treated patients
- cross-val-comp.png = cross-validation RMSE by fold for both models

## Results Summary

Typical outputs (exact numbers will vary):
- Group sizes (patients/visits) for train, test, and treated splits
- PCA variance explained by 10 components
- Model performances (RMSE, MAE, R²) for:
  - Mixed-effects model (train/test/treated)
  - Feedforward neural network (train/test/treated)
- Cross-validation (5-fold, patient-level) RMSE and MAE for both models

These are printed to the console and stored in dtp_model_setup.json.

## Reproducibility

- Random seed = "tstate = 0" (used for NumPy, train/test split, PCA, MLP, CV)
- Determinism = all pre-processing and modelling steps are deterministic given the same input CSV and seed
- Config snapshot = model configuration and results are stored in dtp_model_setup.json

## Limitations

- Only baseline peptide profiles are used; no time-varying molecular features.
- Simple linear mixed-effects structure; no mechanistic PK/PD or disease-progression models. 
- Peptide missingness handled via median imputation; no advanced proteomics-specific methods.
- Treated patients are not used for training; counterfactuals are not yet generated.
- Evaluation is internal (train/test/treated splits); no external validation group.

## Next Steps

- Use an autoencoder from baseline peptides and/or clinical features; improve forecasting model using (embedding, time) to outcome.
- Generate counterfactual control trajectories for treated patients using the trained model; compare predicted vs. recorded outcomes to estimate patient-level treatment effects.
- Build a minimal interactive interface (Colab/Streamlit): input baseline to see personalised forecast. Package into a clean GitHub repo.

## License and Data Usage Note

This code is free-use for usage, modification, and reupload.

Data Acknowledgment:
Data used in this project were obtained from the Accelerating Medicines Partnership® (AMP®) Parkinson's Disease (AMP® PD) Knowledge Platform (https://www.amp-pdrd.org). 
The AMP® PD programme is a public-private partnership managed by the Foundation for the National Institutes of Health and funded by NINDS, FDA, NIA, ASAP, Celgene/BMS, GSK, MJFF, AbbVie, Pfizer, Sanofi, and Verily. Clinical data and biosamples were contributed by cohorts including PPMI, PDBP, BioFIND, and others (see https://www.amp-pdrd.org for details).

## Contact

Agatha Roberts, @virtual-gecko on GitHub.

Rachel Roberts, @rachel080wq on GitHub.
