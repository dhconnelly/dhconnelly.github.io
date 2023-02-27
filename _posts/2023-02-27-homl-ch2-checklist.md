---
layout: post
title: "Notes: ML Project Checklist"
date: 2023-02-27
permalink: /ml-project-checklist.html
---

Notes from Chapter 2 and Appendix A of [Hands-On Machine Learning with Scikit-Learn, Keras & TensorFlow](https://www.oreilly.com/library/view/hands-on-machine-learning/9781098125967/) by Aurélion Géron, which cover
"End-to-End Machine Learning Project" and "Machine Learning Project Checklist"

## Project Checklist

1. Look at the big picture
2. Get the data
3. Explore and visualize the data to gain insights
4. Prepare the data for ML algorithms
5. Select a model and train it
6. Fine-tune your model
7. Present your solution
8. Launch, monitor, and maintain your system

## Look at the big picture

1.  Define the problem in business terms
2.  How will the solution be used?
3.  What are the current solutions/workarounds (if any)?
4.  How should you frame this problem?
    -   Supervised, unsupervised, semi-supervised, self-supervised, reinforcement learning?
    -   Classification, regression, or something else?
    -   Single or multiple regression (i.e. inputs), univariate or multivariate (i.e. outputs) regression?
    -   Batch or online learning?
    -   Batch or online predictions?
    -   Interpretability vs. accuracy?
5.  How should performance be measured?
    -   RMSE generally preferred for regression
6.  Is the performance measure aligned with the business objective?
7.  What would be the minimum performance needed to reach the business objective?
8.  What are comparable problems? Can you reuse experience or tools?
9.  Is human expertise available?
10. How would you solve the problem manually?
11. List the assumptions you/others have made so far
12. Verify assumptions if possible

## Get the data

**_Note_**: Automate as much as possible so you can easily get fresh data!

-   List the data you need and how much you need
-   Find and document where you can get that data
    -   Could be in a relational DB or other data store, spread across tables, etc.
-   Check how much space it will take
-   Check legal obligations, and get authorization if necessary
-   Get access authorizations
-   Create a workspace (with enough storage space)
    -   Colab etc
-   Get the data
    -   Put this in a function/script
    -   Maybe schedule a job to do this at regular intervals
-   Convert the data to a format you can easily manipulate (without changing the data itself)
    -   CSV to pandas DataFrame, `pd.read_csv`
-   Ensure sensitive information is deleted or protected (e.g. anonymized)
-   Check the size and type of data (time series, sample, geographical, etc.)
    -   Not to gain insights about the underlying distribution, but rather to understand what kind of data this is
    -   Column types, categorical values, numerical stats, nulls, etc.
    -   `df.head`, `info`, `value_counts`, `describe`, `hist` w/`plt.show`
    -   Identifies units for numerical values, caps, scaling, skew
    -   If your labels have issues (e.g. are capped but you need accurate predictions outside the range in the data, but your model can't learn them), you can:
        -   Collect proper labels, or
        -   Remove the examples from the training and test sets
-   Sample a test set, put it aside, and never look at it (no data snooping!)

**Seriously, stop here and put aside a test set before you look at the data any further!!!**

### Test set

Why? You might notice some patterns that lead you to choose a certain model, and
your generalization error estimate will be too optimistic. This is known as
_data snooping bias_.

-   Shuffling and sampling on every execution will produce a different test set every time, and eventually you will see the whole dataset
    -   Saving the test set in the first run and loading it, or setting a rng seed, don't solve the issue of fetching new data
    -   Could hash example identifiers and assign to test set if hash value is less than e.g. 20% of maximum hash value
-   **Random sampling** can introduce _sampling bias_
-   **Stratified sampling** tries to guarantee that the test set is representative of the overall data
    -   `train_test_split(stratify=df[cols])`
    -   Based on domain knowledge you may want to create strata by binning continuous features. (But not too many and not too large)
    -   Can add a new feature column, stratify and split test/train, then delete column

## Explore and visualize the data to gain insights

**Note**: Try to get insights from a field expert for these steps.

1. Create a copy of the data for exploration (downsample to manageable size if necessary)
2. Create a Jupyter notebook to keep a record of the explorations
3. Study each attribute and its characteristics:
    - Name
    - Type (categorical, int/float, bounded/unbounded, text, unstructured, etc)
    - % of missing values
    - Noisiness and type of noise (stochastic, outliers, rounding errors, etc)
    - Usefuless for the task
    - Type of distribution (Gaussian, uniform, logarithmic, etc)
4. For supervised learning: identify target attributes
5. Visualize the data (`df.plot`, `hist`, etc)
6. Study correlations (`df.corr`, `scatter_matrix`)
7. How would you solve manually?
8. Identify promising transformations
    - E.g. nonlinear correlation (1/x, etc), attribute combinations
9. Identify extra data that would be useful (go back to ["Get the data"](#get-the-data))
10. Document what you have learned

## Prepare the data for ML algorithms

**Note!**

-   Work on copies of the data (not original dataset)
-   Write functions for all transformations, for five reasons:
    1. Easily handle fresh/new datasets
    2. Can apply as a library in future projects
    3. Clean and prepare the test set
    4. Clean and prepare live data (new data instances) in production
    5. Easily treat choices as hyperparameters (find best combinations, etc)

Steps:

0. Revert to a clean training set and separate predictors from labels
1. Clean the data
    - Fix or remove outliers (optional)
    - Fill in missing values (**imputation**: w/zero, mean, median, etc, `fillna`), drop the instances (`dropna`), or even drop the entire attribute (`drop`)
    - Note that you might need to e.g. impute values for attributes in the live data that didn't need it at training time, so build your transformers to handle this possibility
    - `SimpleImputer`, `KNNImputer`, `IterativeImputer`
2. Feature selection (optional)
    - Drop attributes with no useful information for the task
3. Feature engineering
    - Bucket/discretize continuous features
        - `cut`
        - if multimodal, treat cut bucket as categorical. e.g. home age -> generation bucket (not comparable w.r.t. home value) -> generation as cat attribute
        - another option: RBF (`rbf_kernel(data, compare_to, gamma=)`) can compare data to a specific set of fixed values (e.g. metro areas, generation medians, etc) and return the "distance" to those fixed points
    - Decompose features (e.g. categorical via **one-hot encoding**, date/time, etc)
        - `OneHotEncoder` will remember the categories it was trained on - useful for prod (rather than `df.get_dummies`)
        - Large number of categories for an attribute -> can slow down training and prediction. Maybe replace the attribute with a useful numerical feature?
        - Options: split it up into other data (country code -> gdp, population, etc?), `category_encoders` from sklearn-contrib, or an **embedding** in a neural network. This is **representation learning**
    - Add promising transformations of features (based on ["explore the data"](#explore-and-visualize-the-data-to-gain-insights)), e.g. log, sqrt, polynomial, inverse, etc
    - Aggregate features into new features (feature crosses)
    - Feature scaling: ML algorithms don't perform well when the input numerical attributes have very different scales. _Only fit to the training data!!_
        - Normalization: min/max scaling. `MinMaxScaler`
        - Standardization: (val-mean)/stdev, i.e. value -> # of stdevs away from the mean. `StandardScaler`
        - What if the feature distribution has a **heavy tail** (i.e. values far from the mean aren't exponentially rare)? First, transform (sqrt, log) the feature and _then_ scale it

**Note** that you may need to transform the target values too! E.g. `log`. But
then you'll need to `inverse_transform` the scaled predictions to get your final
predictions (or use `TransformedTargetRegressor`).

### New transformers

-   If it doesn't require training:
    -   write functions f [and g=f^-1] : np.ndarray -> np.ndarray
    -   build `FunctionTransformer(f, [inverse_func=g])`. E.g. combining features w/ratio, etc
-   Else, write a class:
    -   implement `fit` (sets `feature_names_in_`, returns `self`), `transform`, and `fit_transform` (or inherit from `TransformerMixin` to get the latter)
    -   inherit from `BaseEstimator` to get `get_params` and `set_params` to support automatic hyperparameter tuning
    -   implement `get_feature_names_out` and `inverse_transform`
    -   use `check_array` and `check_is_fitted` from `sklearn.utils.validation` in `transform`

You can use other estimators in the implementation!

Use `check_estimator` to make sure you respect the sklearn API!

### Transformation Pipelines

-   to visualize: `sklearn.set_config(display="diagram")`
-   to chain transformers: `sklearn.pipeline.make_pipeline(*tranformers)`
-   `Pipeline` has the same interface as the final estimator
    -   `fit` calls and propagates results from all `fit_transform` methods (but only `fit` on the last one)
    -   `transform` and `predict` similar (but call that method on the last one)
-   To handle different columns separately, use `sklearn.compose.make_column_transformer(tuples)`: takes a sequence of (transformer, columns) tuples and applies each transformer to its specified columns (or specify with `make_column_selector`)

## Select a model and train it

## Fine-tune your model

## Present your solution

## Launch, monitor, and maintain your system
