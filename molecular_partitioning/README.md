# Molecular Partitioning

Datasets quantifying the partitioning of small molecules in MED1, NPM1 and HP1α droplets were collected, consisting of 1143, 1055, and 963 molecules, respectively. To predict the partitioning ratio of molecules, we trained a random forest classifier and a directed message-passing neural network (MPNN) separately and aggregated their predictions. Given a molecule’s SMILES string, the models aimed to predict if the molecule’s partition ratio was above a preset threshold. A molecule’s partitioning ratio was predicted to be above a given threshold if both the random forest and MPNN models predicted a score greater than 0.5 and was predicted below the given threshold otherwise. **In cases where the random forest yielded no positive molecules, only the MPNN predictions were used.**

Partitioning ratios for molecules were given in 3 condensate scaffolding proteins: HP1a, MED1, and NPM1. 

For each molecule-protein pair, we trained a model to predict whether the partitioning ratio ($\rho$) was above or below a threshold ($\tau$). For each condenste, 3 thresholds were used:

```
HP1a: [3.4, 2.7, 2.0]
MED1: [8.0, 5.4, 2.7]
NPM1: [4.5, 3.5, 2.7]
```

For a given threshold, the classification task is to predict $y$ from the molecule $x$:
    
\[ y = P(\rho > \tau | x) \]

## Dataset 

The datasets are structured as CSV files with the following columns: `["SMILES", "PR", "THRESHOLD1", "THRESHOLD2", "THRESHOLD3"]`. The `SMILES` column contains the SMILES string for the molecule. The `PR` column contains the continuous partitioning ratio for the molecule. The `THRESHOLD{i}` column contains the binary target label using the $i^{th}$ threshold. 

For example, the a column in the HP1a dataset would look like:

| SMILES | PR | THRESHOLD1 | THRESHOLD2 | THRESHOLD3 |
|--------|----|------------|------------|------------|
| C1CC1  | 3.0 | 0 | 1 | 1 |


## Deep Learning Models

Install chemprop from source as described in the [chemprop docs](https://github.com/chemprop/chemprop#option-2-installing-from-source).

1. Featurize molecule data

        python  scripts/save_features.py \ 
        --data_path /path/to/dataset.csv  
        --smiles_column SMILES \
        --save_path /path/to/rdkit_features.npz \
        --features_generator rdkit_2d_normalized
    

2. Training the models

        python train.py \
        --data_path /path/to/dataset.csv \
        --dataset_type classification \
        --target_columns Threshold{i} \       # target column, i = 1, 2, 3
        --batch_size 50 \
        --features_path /path/to/rdkit_features.npz \
        --split_type scaffold_balanced \
        --class_balance \
        --no_features_scaling \
        --split_sizes 0.8 0.1 0.1 \
        --smiles_column SMILES \
        --metric auc \
        --extra_metrics accuracy prc-auc \
        --epochs 50 \
        --ensemble 10 \
        --save_dir /path/to/models/condensate_name_tau_{i} # i = 1, 2, 3

3. Evaluating on held-out sets

        python predict.py \               
        --smiles_column SMILES  \
        --no_features_scaling \
        --features_path /path/to/eval_data_rdkit_features.npz \
        --test_path /path/to/eval_data.csv \
        --preds_path /path/to/eval_data_predictions/dataset.csv \
        --checkpoint_dir /path/to/models/

4. Extracting rationales

        python  interpret.py \
        --property_id 1 \
        --no_features_scaling  \
        --features_generator rdkit_2d_normalized  \
        --smiles_column SMILES \
        --data_path /path/to/eval_data_predictions/npm1_positive_smiles.csv  \
        --checkpoint_dir /path/to/models/ \

Similarly for the *in vivo* datasets.

## Random Forest Model

The random forest classifiers were trained using the scikit-learn package (v0.24.2) in Python (v3.8.10), setting “n_estimators” to 200, “min_samples_leaf” to 2, and “n_jobs” to 4.26 Each molecule was transformed into a 1024-dimensional vector using the Chem.RDKFingerprint method from the open-source package RDKit (v2021.03.2).25 Each classifier was trained on 90% of the data. To train the MPNN models on the classification tasks, we used Chemprop (v1.3.1).27 The models took as input both the SMILES string representation of each molecule as well as a 200-dimensional vector generated using Chemprop and setting “features_generator” to rdkit_2d_normalized.28 Molecules were assigned to either the training set (80 %), validation set (10 %), or test set (10 %) using a scaffold split. All MPNNs were trained with a batch size of 50 for 50 epochs with an ensemble of 10 models per task. 

Predictions for a held-out dataset of 1,498 fluorescent molecules were determined by majority voting. 
