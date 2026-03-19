# Do Faster Recipes Receive Higher Ratings?

**Authors:** Roxanne Wang & Ryan Zhang  
**Course:** DSC 80 — Final Project  
**Live Website:** [https://wwroxanne219.github.io/food-recipe-analysis/](https://wwroxanne219.github.io/food-recipe-analysis/))

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

**Key derived columns:**

- `avg_rating` — the mean rating across all user interactions for a given recipe (ratings of 0 were treated as missing, since 0 is not a valid rating value on Food.com)
- `under_30` — a binary indicator for whether a recipe's preparation time is 30 minutes or less

---

## Data Cleaning and Exploratory Data Analysis

### Cleaning Steps

1. Loaded the raw recipes and interactions datasets
2. Replaced ratings of `0` with `NaN` (since 0 is not a valid rating)
3. Computed `avg_rating` for each recipe by grouping interactions by `recipe_id`
4. Merged average ratings back onto the recipes dataframe
5. Created `under_30` to group recipes by preparation time

### Univariate Analysis

**Distribution of Recipe Preparation Time**

![Distribution of Recipe Preparation Time](plot1_minutes_dist.png)

The distribution of preparation time is strongly right-skewed. Most recipes are concentrated at lower preparation times, with a long right tail — meaning there are many relatively quick recipes and far fewer very time-consuming ones. Extreme outliers (above 300 minutes) were excluded for readability.

**Distribution of Average Recipe Ratings**

![Distribution of Average Recipe Ratings](plot2_rating_dist.png)

Average ratings are heavily concentrated near the top of the scale, with most recipes falling between 4 and 5. This ceiling effect suggests that Food.com users tend to rate recipes they try quite positively.

### Bivariate Analysis

**Recipe Preparation Time vs. Average Rating**

![Recipe Preparation Time vs Average Rating](plot3_scatter.png)

This scatterplot shows the relationship between preparation time and average rating. Although there is substantial variation at all time levels, recipes with shorter preparation times appear slightly more likely to have higher ratings.

**Average Rating by Preparation Time Group**

![Average Rating by Preparation Time Group](plot4_boxplot.png)

This boxplot directly compares average ratings for the two preparation-time groups. The group taking 30 minutes or less has a slightly higher center, suggesting that quicker recipes may tend to receive better ratings overall.

### Interesting Aggregates

Grouping recipes by preparation-time category reveals the following summary:

| Group | Mean Avg Rating | Recipe Count | Mean Minutes | Mean N Ingredients |
|---|---|---|---|---|
| 30 min or less | 4.6447 | 36,419 | 17.92 | 7.81 |
| Over 30 min | 4.6097 | 44,754 | 193.00 | 10.34 |

Shorter recipes not only have a slightly higher mean rating, but also tend to use fewer ingredients, while longer recipes represent the larger portion of the dataset.

---

## Assessment of Missingness

### NMAR Analysis

The `description` column contains missing values. This missingness may be **NMAR (Not Missing at Random)** — contributors with little to say, or with weaker recipe descriptions, may be more likely to leave the field blank. Since this may depend on the unseen missing content itself, the mechanism could be NMAR.

### Missingness Dependency

Permutation tests were run to check whether the missingness of `description` depends on other observed columns:

| Column Tested | Observed Difference | P-value | Conclusion |
|---|---|---|---|
| `n_steps` | 0.9953 | 0.188 | Missingness does **not** depend on `n_steps` |
| `submitted_year` | −0.5153 | 0.020 | Missingness **does** depend on `submitted_year` |

Since the p-value for `submitted_year` is below 0.05, we reject the null hypothesis for that test and conclude that the missingness of `description` is associated with when the recipe was submitted.

---

## Hypothesis Testing

**Research question:** Do recipes taking 30 minutes or less tend to receive higher average ratings?

| | |
|---|---|
| **Null hypothesis** | The distribution of average recipe ratings is the same for recipes that take 30 minutes or less and recipes that take more than 30 minutes |
| **Alternative hypothesis** | Recipes taking 30 minutes or less tend to have higher average ratings |
| **Test statistic** | Difference in mean average ratings between the two groups |
| **Method** | Permutation test, 1,000 permutations |
| **Significance level** | 0.05 |

| Statistic | Value |
|---|---|
| Observed difference in means | 0.0377 |
| P-value | < 0.001 |

**Conclusion:** We reject the null hypothesis. The evidence supports that recipes taking 30 minutes or less tend to receive slightly higher average ratings than recipes taking more than 30 minutes.

---

## Framing a Prediction Problem

The prediction target is `avg_rating` — a continuous variable, making this a **regression** problem. Only features available at the time of prediction (i.e., before any user ratings are submitted) are used. These include recipe-level attributes such as preparation time, number of steps, number of ingredients, and parsed nutrition values.

**Evaluation metric:** RMSE (Root Mean Squared Error), chosen because it measures typical prediction error in the same units as the target variable.

---

## Baseline Model

| Detail | Value |
|---|---|
| Model | Linear Regression (sklearn Pipeline) |
| Features | `minutes`, `n_steps` |
| Test RMSE | **0.6359** |

A simple linear regression using preparation time and number of steps predicts average ratings with an RMSE of approximately 0.636 — meaning predictions are off by about 0.636 rating points on average.

---

## Final Model

The final model is a **Random Forest Regressor** implemented in a single sklearn Pipeline and tuned via `GridSearchCV`. Compared to the baseline, it adds:

- `n_ingredients` — number of recipe ingredients
- Parsed nutrition variables: `calories`, `sugar`, `protein`, `carbs`
- `log_minutes` — log-transformed preparation time to reduce the influence of extreme outliers
- `is_quick` — binary indicator for recipes under 30 minutes

**Hyperparameters tuned:** `max_depth` (controls tree complexity) and `n_estimators` (number of trees for stability)

| Hyperparameter | Best Value |
|---|---|
| `max_depth` | 5 |
| `n_estimators` | 200 |

| Model | Test RMSE |
|---|---|
| Baseline (Linear Regression) | 0.6359 |
| **Final (Random Forest)** | **0.6349** |

The final model improves on the baseline through more thoughtful feature engineering and hyperparameter tuning.

---

## Fairness Analysis

We evaluated whether the final model performs equitably for the two preparation-time groups using RMSE as the fairness metric.

| | |
|---|---|
| **Null hypothesis** | The model's RMSE is roughly the same for both groups; any observed difference is due to chance |
| **Alternative hypothesis** | The model's RMSE is higher for recipes taking more than 30 minutes |
| **Method** | Permutation test, 1,000 permutations |

| Group | RMSE |
|---|---|
| Recipes ≤ 30 minutes | 0.6012 |
| Recipes > 30 minutes | 0.6533 |
| Observed difference | 0.0521 |
| P-value | < 0.001 |

**Conclusion:** We reject the null hypothesis. The model performs significantly worse for longer recipes, indicating a fairness concern across preparation-time groups that would be worth addressing in future work.

