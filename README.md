# Predicting Song Popularity from Musical Composition Features: A Dual-Model Approach

MSc coursework project (University of Sheffield), Grade: 78 (Distinction). Tests whether 9 acoustic features plus explicitness can predict whether a song is popular, using two model families (logistic regression and Random Forest), each run with and without the explicitness variable, plus a stepwise-reduced logistic regression.

> **Reproducibility note:** the raw dataset (merged Spotify-style track, artist, and popularity files) is not included in this repo, so `song_pop_code.Rmd` cannot be rerun end-to-end from this repo alone. All headline numbers below were traced to specific lines of the code and cross-checked against my coursework results tables rather than taken from my write-up's prose; see the Verification note, which covers two bugs I found in my submitted code.

## Key results

Five models were fit in total: two logistic regressions (with and without `explicit`), a stepwise-reduced version of the better logistic regression, and two Random Forests (with and without `explicit`).

| Model | Accuracy | Specificity | Sensitivity | AUC |
|---|---|---|---|---|
| Logistic Regression 1 (no explicit) | 63.08% | 100% | 0% | 0.514 |
| Logistic Regression 2 (+ explicit) | 62.58% | 100% | 0% | 0.525 |
| Stepwise Regression (SWR)\* | n/a | n/a | n/a | 0.524 |
| Random Forest 1 (no explicit)* | 59.63% | 91.67% | **9.03%** | 0.506 |
| Random Forest 2 (+ explicit) | 58.93% | 91.26% | 7.87% | 0.511 |

\* Marked as the best-performing model of its type in my coursework results tables.
† SWR accuracy, specificity and sensitivity are shown as n/a because my code reused another model's predictions (see Verification note); only its AUC (0.524) was computed correctly.

**Main finding:** every model performs close to chance (AUC 0.506-0.525). Both logistic regressions never predicted a popular song (0% sensitivity), so their high accuracy and specificity only reflect the 62% not-popular majority. The Random Forests do flag some songs as popular, but RF1's hit rate (9.0% sensitivity) is barely above its false-alarm rate (8.3%, from 91.67% specificity), which is what chance would give. RF1 has higher sensitivity than RF2, while RF2 has the higher AUC (0.511 vs 0.506), but the differences are small, come from single test splits, and have no confidence intervals.

![Random Forest 1 vs Random Forest 2: AUC comparison](figures/rf1_vs_rf2_auc_comparison.png)

**Feature importance (Random Forest, explicit-inclusive model):** loudness and energy were the strongest predictors by Mean Decrease Accuracy; tempo and speechiness by Mean Decrease Gini. Note this chart is only available for Random Forest 2, since my code never generated a separate importance plot for Random Forest 1.

![Random Forest 2 variable importance](figures/rf2_feature_importance.png)

**Musical features by popularity outcome:** across all 9 acoustic features, the boxplots for popular vs. non-popular songs overlap almost completely, visually consistent with the near-chance AUC values above.

![Distribution of musical features by song popularity](figures/musical_features_by_popularity.png)

**Multicollinearity check:** energy and loudness are correlated at r = 0.7 (the strongest pairwise correlation among the 9 features), consistent with the "loudness war" literature cited in the write-up. All other pairs are weak.

![Correlation between musical characteristics](figures/correlation_heatmap.png)

## Methods & tools

- **Data:** MusicOSet (Silva et al., 2019). I merged 4 of its 13 tables on song ID, kept 2000-2018 (loudness is inconsistently scaled before 2000), removed collaborations to keep solo songs only, and dropped the 10 rows with missing values left after the joins, leaving 6,662 songs. "Popular" (`is_pop`) is MusicOSet's Billboard-based label: each song gets a year-end score from its peak position and weeks on the Hot 100, songs scoring above that year's average are labelled popular, and the lowest scorers are labelled not popular (2,509 popular, 37.7%; 4,153 not popular, 62.3%).
- **Language:** R (R Markdown)
- **Data split:** 70/30 train/test, shuffled with a fixed seed
- **Models:** logistic regression (`glm`, binomial), bidirectional stepwise selection (`step`), Random Forest (`randomForest`, 500 trees, permutation-based importance)
- **Evaluation:** confusion matrix (accuracy, sensitivity, specificity), AUC-ROC (chosen over raw accuracy because the outcome classes are imbalanced, 62% not-popular vs. 38% popular), Nagelkerke pseudo-R²
- **Other checks:** Pearson correlation matrix and VIF for multicollinearity; chi-squared test for the association between explicitness and popularity

## Repo structure

```
song-popularity-dual-model/
├── README.md                                    ← you are here
├── song_pop_code.Rmd                                  ← full analysis code
└── figures/
    ├── rf1_vs_rf2_auc_comparison.png
    ├── rf2_feature_importance.png
    ├── rf1_auc_curve.png
    ├── stepwise_roc_curve.png
    ├── musical_features_by_popularity.png
    ├── correlation_heatmap.png
    └── extra/
        ├── lgr1_roc_curve.png
        ├── lgr2_roc_curve.png
        ├── rf2_auc_curve.png
        ├── lgr2_vs_rf2_auc_comparison.png
        ├── confusion_matrix_lgr2.png
        ├── confusion_matrix_rf2.png
        ├── confusion_matrix_comparison_lgr2_vs_rf2.png
        ├── lgr2_feature_importance_zscore.png
        ├── rf2_feature_importance_gini_single_panel.png
        └── correlation_heatmap_viridis_alt.png
```

## How to run

1. Install R and the packages `song_pop_code.Rmd` loads at the top (tidyverse, caret, pROC, randomForest, viridis, patchwork, DescTools, car, sjPlot).
2. The script expects several pre-cleaned source files (song, artist, musical-feature, and popularity data) that are not included in this repo; see Data, below.
3. Run top to bottom. A fixed seed is set before each train/test split and each model fit.

## Data

The underlying track/artist/popularity dataset is not redistributed here (Spotify-derived data, not mine to redistribute). `song_pop_code.Rmd` is included in full so the methodology is auditable even though it can't be rerun without the source files.

## Limitations

- Every model's AUC sits close to 0.5 (random chance). My write-up concludes that a weak non-linear signal exists because Random Forest identified some popular songs (9% sensitivity), but with AUC ~0.51 and accuracy below the no-information rate (59.6% vs 61.2%), that evidence is thin, and the linear models found none (Nagelkerke R² < .01).
- Both full logistic regressions scored 0% sensitivity: they classified every single test-set song as not-popular. Their accuracy and specificity numbers reflect the class balance (62% not-popular), not genuine predictive skill.
- Random Forest 1 (the higher-sensitivity RF model) never got its own feature-importance plot in the code; the only importance chart available is for Random Forest 2.
- Each model was evaluated on its own 70/30 split (test-set no-information rates differ: 63.1%, 62.6%, 61.2%) with no confidence intervals, so small differences between models (e.g. AUC 0.506 vs 0.511) should not be over-read, and comparisons across model families are indicative only.
- The 6,662 rows include 414 with duplicated song names (radio edits, remasters, regional releases), which I kept deliberately. Because the data was shuffled before a random split, variants of the same song can fall in both training and test sets, which may flatter the models.

## Verification note

While tracing each figure back to the code, I found two bugs in my submitted coursework. (1) The stepwise model's predicted classes reuse Logistic Regression 2's probabilities (`test_probabilities_2` instead of `test_probabilities_swr`), so its accuracy, specificity and sensitivity are unreliable; only its AUC (0.524) comes from its own ROC computation. The reported figures (62.58% accuracy, 100% specificity, 0% sensitivity) are identical to Logistic Regression 2's, consistent with this bug, so I show them as n/a in the table. (2) The original final comparison chart labelled Random Forest 1's curve as "Random Forest 2"; that chart is not included here. All other figures matched the code that produced them.
