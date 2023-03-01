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
    -   Classification:
        -   Accuracy is common, but not great for skewed datasets
        -   Look at the **confusion matrix**: true pos, true neg, false pos, false neg
        -   Precision and recall
            -   Precision (accuracy of positive predictions): TP / (TP + FP)
            -   Recall (how many positives were predicted): TP / (TP + FN)
            -   F1 score (harmonic mean of precision and recall): (P x R)/(P + R)
            -   Should choose this based on the domain and business needs! Sometimes precision is more important than recall, etc.
            -   And there's a tradeoff: optimizing for higher precision means lowering recall, etc. Choose a decision threshold by plotting P and R vs. threshold and making a business decision. "Let's do 99% precision." => "At what recall?"
        -   ROC plots true pos vs. false pos. Compare classifiers with AOC (area under ROC curve)
        -   When to use which?
            -   Accuracy if data is balanced and you care about both pos and neg predictions
            -   Precision/recall curve when positives are rare (or you care more about FP than FN)
            -   ROC curve and AUC otherwise
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

**Note**:

-   If the data is huge, sample smaller training sets. This will penalize complex models though!
-   Automate!!

Steps:

1. Train many quick-and-dirty models from different categories (e.g. linear, naive Bayes, SVM, random forest, neural net, etc) using standard parameters
2. Measure and compare their performance
    - Use N-fold cross-validation and compute mean/stdev of perf measure on the folds
    - `cross_val_score(model, X, Y, scoring="neg_root_mean_squared_error", cv=10)` uses 10 subsets (folds), trains w/ 9 and cross-validates with 1, and does this 10 times
3. Analyze the most significant variables of each algorithm
4. Analyze the types of errors the models could make/does make
    - What data would a human have used to avoid these errors?
    - Do we need to gather more training data? Do **data augmentation**?
    - Consider specific slices of validation sets for relevant subgroups (e.g. disadvantaged groups, specific metros, etc)
5. Quick round of feature selection and engineering
    - Which attributes have the highest weights/importances
    - `feature_importances_` for trees
    - Maybe remove some, add new ones, etc
6. Repeat the above one or two more times - quickly
7. Shortlist the top three to five promising models, preferring models that make different types of errors

## Fine-tune your model

Use as much data as possible - and automate!

1. Fine-tune the hyperparameters using cross-validation
    - Treat data transformation choices as hyperparameters (e.g. impute with zero or median, or drop, etc)
    - Prefer `RandomSearchCV` over `GridSearchCV` unless there are very few options to test. Consider Bayesian optimization instead if it takes very long
2. Try ensemble methods - combining best models can produce better results
3. Pick best model
    - compare cross-validation scores
    - get tuned model and score via `best_estimator_` and `best_score_` from `XSearchCV`
4. Estimate generalization error by measuring performance on the test set
    - This will likely be worse than your cross-validation error, since you fine-tuned to the training set
    - Don't go tweak hyperparameters to make this number look good - it won't generalize, you'll overfit the test set!

## Present your solution

1. Document what you have done
2. Create a nice presentation - highlight the big picture first
3. Explain why your solution achieves the business objective
4. Present interesting points noticed along the way
    - What worked and what didn't?
    - List your assumptions and your system's limitations
5. Ensure the key findings are communicated through beautiful visualizations and easy-to-remember statements (e.g. "the median income is the #1 predictor of housing prices")

## Launch, monitor, and maintain your system

1. Get your solution ready for production
    - Save the best model: `joblib.dump`
    - How to use it in production?
        1. Load it in your app with `joblib.load` and make predictions in your app
        2. Split the loading and predictions into a prediction service/API
        3. Upload to Cloud options (e.g. Google Vertex AI) that you call via API
    - Plug into production data inputs, write unit tests, etc
2. Write monitoring code to check the system's live performance at regular intervals and trigger alerts when it drops:
    - Can monitor via downstream metrics (e.g. product sales)
    - May require a human eval pipeline / **human raters** (crowdsourcing) to monitor performance directly
    - _Also monitor the input data quality!_ E.g. a sensor sending bad/random values, an input dependency data becoming stale, mean/stdev drifts too far from training set, more missing values than expected, new categorical values, etc. Particularly important for online learning systems!
    - Beware of slow degradation - models tend to "rot" as data evolves
3. Retrain the models on a regular basis on fresh data - AUTOMATE!
    - Schedule collecting fresh data and label it (e.g. w/human raters again)
    - Schedule re-train: load fresh data, train model, fine-tune hyperparameters
    - Schedule conditional deployment: compare live model performance with new model, deploy if not worse. If worse, alert and investigate why!
    - **Keep backups of every model and every dataset version and support easy rollbacks!**
