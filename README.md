# Do Faster Recipes Receive Higher Ratings?

**Authors:** Roxanne Wang & Ryan Zhang  
**Course:** DSC 80 — Final Project  
**Live Website:** [food-recipe-analysis](https://wwroxanne219.github.io/food-recipe-analysis/)

---

## Overview

This project analyzes the **Food.com Recipes and Ratings dataset** to explore the relationship between recipe preparation time and user-submitted ratings. The dataset combines recipe characteristics — such as preparation time, number of steps, ingredients, tags, and nutrition information — with interaction data capturing how users have rated individual recipes.

Our central research question is:

> **Do recipes that take 30 minutes or less to prepare tend to receive higher average ratings than recipes that take more than 30 minutes?**

This question is practically motivated: preparation time is one of the most visible attributes of a recipe and may directly influence how satisfied users are with their cooking experience. We investigate this relationship through data cleaning, exploratory data analysis, missingness analysis, hypothesis testing, and a predictive model for average recipe ratings.

---

## Dataset

The analysis uses two raw data files from Food.com:

| File | Description |
|---|---|
| `RAW_recipes.csv` | Recipe-level information: preparation time, number of steps, number of ingredients, tags, nutrition, and more |
| `interactions.csv` | User-submitted ratings and reviews linked to recipe IDs |

**Key derived column:**

- `avg_rating` — the mean rating across all user interactions for a given recipe (ratings of 0 were treated as missing, since 0 is not a valid rating value on Food.com)
- `under_30` — a binary indicator for whether a recipe's preparation time is 30 minutes or less

---

## Research Question & Hypothesis Testing

**Null hypothesis:** The distribution of average recipe ratings is the same for recipes that take 30 minutes or less and recipes that take more than 30 minutes.

**Alternative hypothesis:** Recipes that take 30 minutes or less tend to have higher average ratings than recipes that take more than 30 minutes.

**Test statistic:** Difference in mean average ratings between the two preparation-time groups  
**Method:** Permutation test with 1,000 permutations  
**Significance level:** 0.05

| Statistic | Value |
|---|---|
| Observed difference in means | 0.0377 |
| P-value | < 0.001 |

**Conclusion:** We reject the null hypothesis. Recipes taking 30 minutes or less tend to receive slightly higher average ratings than recipes taking more than 30 minutes.

---

## Methods

### Data Cleaning

1. Loaded the raw recipes and interactions datasets
2. Replaced ratings of `0` with `NaN` (since 0 is not a valid rating)
3. Computed `avg_rating` for each recipe by grouping interactions by `recipe_id`
4. Merged the average ratings back onto the recipes dataframe
5. Created a `under_30` column to group recipes by preparation time

### Exploratory Data Analysis

- **Univariate analysis:** The distribution of preparation time is strongly right-skewed — most recipes are quick, but a long tail of very time-consuming recipes exists. Average ratings are heavily concentrated near 4–5.
- **Bivariate analysis:** A scatterplot of minutes vs. average rating shows substantial variation, but shorter recipes appear slightly more likely to have higher ratings. A boxplot comparing the two preparation-time groups confirms that quicker recipes have a slightly higher center in their rating distribution.
- **Aggregates:** Grouping by preparation-time category reveals that shorter recipes have higher mean ratings, higher recipe counts, lower mean preparation time, and slightly fewer mean ingredients.

### Assessment of Missingness

The `description` column has missing values. This missingness may be **NMAR (Not Missing at Random)** — contributors with little to say, or weaker descriptions, may be more likely to leave the field blank, meaning the missingness depends on the unseen content itself.

To test missingness dependency, permutation tests were run against two candidate columns:

| Column tested | Observed difference | P-value | Conclusion |
|---|---|---|---|
| `n_steps` | 0.9953 | 0.188 | Missingness does **not** depend on `n_steps` |
| `submitted_year` | −0.5153 | 0.020 | Missingness **does** depend on `submitted_year` |

### Prediction Problem

The prediction target is `avg_rating` — a continuous variable, making this a **regression** problem. Only features available at the time of prediction (i.e., before any user ratings are submitted) are used.

**Evaluation metric:** RMSE (Root Mean Squared Error), chosen because it measures typical prediction error in the same units as the target variable.

---

## Results & Findings

### Baseline Model

| Detail | Value |
|---|---|
| Model | Linear Regression |
| Features | `minutes`, `n_steps` |
| Test RMSE | **0.6359** |

A simple linear regression using preparation time and number of steps predicts average ratings with an RMSE of approximately 0.636.

### Final Model

The final model is a **Random Forest Regressor** tuned via `GridSearchCV`. In addition to the baseline features, the final model includes:

- `n_ingredients` — number of recipe ingredients
- Parsed nutrition variables: `calories`, `sugar`, `protein`, `carbs`
- Engineered feature `log_minutes` — log-transformed preparation time to reduce the influence of outliers
- Engineered feature `is_quick` — binary indicator for recipes under 30 minutes

**Hyperparameters tuned:**
- `max_depth` — controls tree complexity and guards against overfitting
- `n_estimators` — number of trees for improved stability

| Hyperparameter | Best value |
|---|---|
| `max_depth` | 5 |
| `n_estimators` | 200 |

| Model | Test RMSE |
|---|---|
| Baseline (Linear Regression) | 0.6359 |
| Final (Random Forest) | **0.6349** |

The final model achieves a modest improvement over the baseline through more thoughtful feature engineering and hyperparameter tuning.

### Fairness Analysis

We evaluated whether the final model performs equitably for recipes in the two preparation-time groups using RMSE as the fairness metric.

**Null hypothesis:** The model's RMSE is roughly the same for both groups; any observed difference is due to chance.  
**Alternative hypothesis:** The model's RMSE is higher for recipes that take more than 30 minutes.  
**Method:** Permutation test with 1,000 permutations

| Group | RMSE |
|---|---|
| Recipes ≤ 30 minutes | 0.6012 |
| Recipes > 30 minutes | 0.6533 |
| Observed difference | 0.0521 |
| P-value | < 0.001 |

**Conclusion:** We reject the null hypothesis. The model performs significantly worse for longer recipes, indicating a fairness concern across preparation-time groups.

---

## Conclusion

Our analysis provides evidence that preparation time is associated with recipe ratings on Food.com. Recipes taking 30 minutes or less tend to receive slightly higher average ratings, and this difference is statistically significant. However, the effect size is small (a mean difference of ~0.04 rating points), so the practical significance is limited.

The final predictive model improves marginally over a simple linear baseline, but the fairness analysis reveals that the model is less accurate for longer recipes — a concern worth addressing in future modeling efforts.


