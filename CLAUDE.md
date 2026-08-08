# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of educational Jupyter notebooks for Oracle Cloud Infrastructure (OCI) Data Science labs. There is no application code, build system, test suite, or linter — the notebooks and their datasets are the entire repository.

## Notebooks

- **Lab 1 Linear Regression (OCI).ipynb** — house price prediction with scikit-learn linear regression (EDA with seaborn/matplotlib, StandardScaler, train/test split, MSE evaluation). Uses `housesales.csv`.
- **house-price-prediction.ipynb** — an extended version of the same house price workflow with deeper data exploration and feature engineering (skew correction, outlier handling). Also uses `housesales.csv`.
- **Auto MLlab100-bonus-1.ipynb** — house price prediction using Oracle's Accelerated Data Science (ADS) SDK AutoML (`ads`, `oci`, `DatasetFactory`). Also uses `housesales.csv`.
- **Lab 2 Neural Networks (OCI).ipynb** — MNIST digit classification with Keras/TensorFlow. Loads the raw IDX-format MNIST files via `idx2numpy`.

## Data Files

- `housesales.csv` — King County house sales dataset (shared by the three house-price notebooks).
- `train-images-idx3-ubyte`, `train-labels-idx1-ubyte`, `t10k-images-idx3-ubyte`, `t10k-labels-idx1-ubyte` — raw MNIST training/test sets (used by Lab 2).

Notebooks reference these files by relative path, so they must be run with the repository root as the working directory.

## Environment

All notebooks target an OCI Data Science notebook session using the conda environment `generalmachinelearningforcpusv1_0` (Python 3.6). Key dependencies: pandas, numpy, scipy, scikit-learn, seaborn, matplotlib, tensorflow/keras, idx2numpy, and the Oracle `ads` and `oci` SDKs (AutoML notebook only). The `ads`/`oci` SDKs assume OCI resource-principal or config-based authentication available inside an OCI notebook session; the other notebooks run in any standard Jupyter environment with the scientific Python stack installed.

To work locally, run `jupyter notebook` (or `jupyter lab`) from the repository root and open the desired notebook.

## Conventions

- The `.ipynb_checkpoints/` directory is currently committed to the repository. Do not edit checkpoint files; make changes to the top-level notebooks only.
- Lab notebooks begin with a standard "OCI Data Science - Useful Tips" markdown cell (internet-access check, conda environment upgrade instructions). Preserve this header when editing the lab notebooks.
