# Predicting Recipe Ratings on Food.com

**By Kylan Huynh**

### Overall components of this website

1. [Introduction](#introduction)
2. [Data Cleaning and Exploratory Data Analysis](#data-cleaning-and-exploratory-data-analysis)
3. [Assessment of Missingness](#assessment-of-missingness)
4. [Hypothesis Testing](#hypothesis-testing)
5. [Framing a Prediction Problem](#framing-a-prediction-problem)
6. [Baseline Model](#baseline-model)
7. [Final Model](#final-model)
8. [Fairness Analysis](#fairness-analysis)

---

## Introduction

Cooking websites live and die by their recipes, and on [food.com](https://www.food.com/) the clearest signal of a great recipe is its star rating. This project explores a large public dataset of recipes and user reviews posted to food.com since 2008. After cleaning, the data covers roughly **83,000 recipes**. Each recipe comes with metadata describing how it was written — how long it takes to make, how many steps and ingredients it has — along with a nutritional breakdown (calories and the percent-daily-value of fat, sugar, sodium, protein, saturated fat, and carbohydrates). A separate dataset of user reviews supplies the star ratings, which we average into a single rating per recipe.

The central question driving this project is:

> **Can we predict whether a recipe will be highly rated — earning a perfect 5.0 average — using only the information available the moment it is posted, before a single review comes in?**

This question matters for recipe recommendation. If a recipe's basic metadata already hints at how well it will be received, a platform could surface promising new recipes before they have accumulated any reviews, and home cooks could get a sense of which recipes are worth their time.

The columns most relevant to our analysis are the preparation time (`minutes`), the number of steps (`n_steps`), the number of ingredients (`n_ingredients`), the calorie count and other nutrition values, and the average star rating (`avg_rating`) we compute for each recipe.

---

## Data Cleaning and Exploratory Data Analysis

Getting the data ready involved several steps, each chosen to make the later analysis trustworthy:

- **Combining recipes with their reviews.** We merged every recipe with all of its reviews so that we could compute an average rating per recipe.
- **Fixing "zero" ratings.** On food.com a rating of 0 does not mean a reviewer hated a recipe — it is the value recorded when someone left a comment but no star rating at all. Leaving these in would unfairly drag ratings down, so we treated every 0 as "no rating" before averaging.
- **Unpacking the nutrition information.** The nutrition data was stored as a single text string (for example, `"[138.4, 10.0, 50.0, ...]"`). We split it into seven separate numeric columns — calories, fat, sugar, sodium, protein, saturated fat, and carbohydrates — so each could be analyzed on its own.
- **Removing impossible cooking times.** A handful of recipes claimed preparation times of hundreds of thousands of minutes (one listed nearly two years). These are clearly data-entry mistakes or jokes, so we kept only realistic recipes that take between one minute and 24 hours.
- **Defining "highly rated."** Finally, we labeled each recipe as highly rated if its average rating was a perfect 5.0, and not highly rated otherwise.

Here is a peek at a few rows of the cleaned data:

| name                                 |   minutes |   n_steps |   n_ingredients |   calories |   protein |   avg_rating | high_rating   |
|:-------------------------------------|----------:|----------:|----------------:|-----------:|----------:|-------------:|:--------------|
| 1 brownies in the world    best ever |        40 |        10 |               9 |      138.4 |         3 |            4 | False         |
| 1 in canada chocolate chip cookies   |        45 |        12 |              11 |      595.1 |        13 |            5 | True          |
| 412 broccoli casserole               |        40 |         6 |               9 |      194.8 |        22 |            5 | True          |
| millionaire pound cake               |       120 |         7 |               7 |      878.3 |        20 |            5 | True          |
| 2000 meatloaf                        |        90 |        17 |              13 |      267   |        29 |            5 | True          |

To understand the ratings themselves, we first looked at how they are distributed across all recipes:

<iframe src="assets/rating_distribution.html" width="800" height="500" frameborder="0"></iframe>

The ratings are heavily skewed toward the top of the scale: the vast majority of recipes sit at or near five stars, with very few low scores. This tells us that food.com reviewers are a generous, self-selecting crowd — people tend to review recipes they already liked — which is exactly why predicting a *perfect* score is more interesting than predicting a merely "good" one.

Next we asked whether more complicated recipes are rated differently from simple ones. The plot below compares the ratings of recipes with relatively few steps against those with many steps:

<iframe src="assets/rating_by_steps.html" width="800" height="500" frameborder="0"></iframe>

The two groups look remarkably similar — recipe complexity, at least as measured by step count, does not have a dramatic effect on how recipes are rated.

Finally, we grouped recipes by two characteristics at once — whether they are quick (30 minutes or less) or long, and whether they use few (under 10) or many ingredients — and looked at the average rating in each group:

| speed            |   Few (<10) |   Many (>=10) |
|:-----------------|------------:|--------------:|
| Long (>30 min)   |      4.6005 |        4.6161 |
| Quick (<=30 min) |      4.6462 |        4.6405 |

The differences are small, but a pattern emerges: quick recipes tend to edge out long ones in average rating, regardless of how many ingredients they use. This nudge motivated the formal hypothesis test below.

---

## Assessment of Missingness

One column stands out as a likely case of **NMAR** ("Not Missing At Random") — meaning whether a value is missing depends on the missing value itself. That column is the recipe `description`. A description is absent when a submitter simply chose not to write one, and that choice probably reflects how much effort and care they put into the submission overall — an attribute we never directly observe. A cook who quickly dumps a recipe onto the site is both less likely to write a description and, plausibly, the very reason the description is missing. Because the missingness is tied to this hidden "effort" rather than to anything we recorded, it qualifies as NMAR. If we could additionally collect data on each submitter's overall activity — how many recipes they post, how complete their other submissions are — that hidden effort would become observable, and the missingness might then be explainable from the data (making it "Missing At Random" instead).

We then studied the missingness of the **average rating**, which is missing precisely for recipes that never received any reviews. To see what that missingness depends on, we used a **permutation test** — a technique that repeatedly reshuffles the data to simulate what we would expect to see purely by chance, then checks whether the real data looks unusual against that backdrop.

- **Does it depend on preparation time?** Yes. Recipes with no reviews have meaningfully different preparation times from reviewed ones, and this gap is far larger than chance could explain (p ≈ 0). In other words, whether a recipe gets reviewed is related to how long it takes to make.
- **Does it depend on sodium content?** No. A recipe's sodium level looks essentially the same whether or not it was ever reviewed (p ≈ 0.75) — well within the range of random variation.

The chart below shows the "by chance" distribution for the preparation-time test; the red line marks what we actually observed, sitting far out in the tail:

<iframe src="assets/missingness_minutes.html" width="800" height="500" frameborder="0"></iframe>

---

## Hypothesis Testing

The exploratory analysis hinted that quicker recipes might be rated more highly. We tested two questions formally, again using permutation tests and a 5% significance level.

**Test 1: Do quick recipes (30 minutes or less) receive higher ratings than long recipes?**

- **Null hypothesis:** Quick and long recipes have the same average rating; any observed difference is due to chance.
- **Alternative hypothesis:** Quick recipes have a higher average rating than long recipes.
- **Test statistic:** the difference in mean rating (quick minus long).
- **Result:** observed difference ≈ +0.035 stars, p-value ≈ 0.000.
- **Conclusion:** At the 5% significance level, we reject the null hypothesis. The data is consistent with quick recipes tending to receive higher ratings than long ones. Because this is observational data, we cannot claim that being quick *causes* higher ratings — only that the two are associated.

**Test 2: Is a recipe's rating associated with how many ingredients it has?**

- **Null hypothesis:** Recipes with many ingredients (10 or more) and few ingredients have the same average rating.
- **Alternative hypothesis:** The two groups have different average ratings.
- **Result:** observed difference ≈ −0.003 stars, p-value ≈ 0.39.
- **Conclusion:** We fail to reject the null hypothesis. The data does not provide evidence that ingredient count is associated with rating.

The chart below shows the chance distribution for the first test, with the observed difference marked in red — far enough into the tail to be very unlikely under the null hypothesis:

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

We turned the central question into a **binary classification** problem: predict whether a recipe will earn a **perfect 5.0 average rating** or not. Every feature we use — preparation time, number of steps, number of ingredients, and the nutrition figures — is known the instant a recipe is posted. We deliberately avoid anything that only exists after people start reviewing (the rating itself, the number of reviews, or the review text), because using those would be "cheating": they would not be available at the moment we actually want to make a prediction.

About **59%** of rated recipes earn a perfect 5.0 and **41%** do not, so the two groups are somewhat imbalanced. For that reason we evaluate models with the **weighted F1-score** rather than plain accuracy. Accuracy can be misleadingly high if a model simply always guesses the more common answer; the F1-score instead rewards a model for correctly identifying recipes in *both* groups, which is what we care about here.

---

## Baseline Model

Our baseline is a simple, well-understood model — **logistic regression** — using four straightforward numeric features: preparation time, number of steps, number of ingredients, and calories. All four are already numbers, so no special encoding was needed.

The baseline is intentionally bare-bones, and its performance reflects that: it achieves a weighted F1-score of about **0.44**, leaning heavily toward guessing the majority ("perfect") answer and doing a poor job of telling perfect and non-perfect recipes apart. It is a starting point — a yardstick the more sophisticated model has to beat.

---

## Final Model

For the final model we engineered four new features, each motivated by *how recipes and reviews actually come about* rather than by trial and error:

- **Log of preparation time.** People judge time on a relative scale — the gap between 10 and 20 minutes feels much bigger than between 100 and 110. Taking the logarithm reflects that perception and tames a small number of very long recipes that would otherwise dominate.
- **Calorie density (calories per ingredient).** This separates rich, indulgent recipes (lots of calories from a few ingredients, like butter and chocolate) from light, ingredient-heavy ones — a distinction that may shape how satisfied reviewers feel.
- **Protein-to-sugar ratio.** This captures a recipe's character along a savory/healthy versus dessert-like spectrum, which tends to attract different audiences.
- **A "quick recipe" flag.** Because our hypothesis test showed quick recipes are rated higher, we hand the model that finding directly as a simple yes/no feature.

We fed these into a **random forest** — a model that combines many decision trees and can capture interactions between features that a straight line cannot. We tuned its settings (the number of trees, how deep each tree can grow, and how many recipes each branch must contain) using cross-validation, which tests each combination on held-out slices of the training data. The best settings were 100 trees, unlimited depth, and a minimum of one recipe per leaf.

On the same held-out test set used for the baseline, the final model reached a weighted F1-score of about **0.53 — an improvement of roughly 0.09 over the baseline.** Just as importantly, it stopped simply guessing the majority and began correctly identifying a meaningful share of the non-perfect recipes the baseline missed entirely. The confusion matrix below shows how its predictions break down across the two groups:

<iframe src="assets/confusion_matrix.html" width="700" height="600" frameborder="0"></iframe>

---

## Fairness Analysis

Finally, we asked whether the model is **fair** — specifically, whether it works equally well for quick recipes (30 minutes or less) and long recipes (more than 30 minutes).

- **Null hypothesis:** The model's weighted F1-score is the same for quick and long recipes; any difference is due to chance.
- **Alternative hypothesis:** The model's weighted F1-score differs between the two groups.

Using a permutation test on the difference in F1-scores, we found the model scored 0.533 on quick recipes and 0.527 on long ones — a gap of only about 0.006, with a p-value of **0.45**. At the 5% significance level we **fail to reject** the null hypothesis: the data provides no evidence that the model treats quick and long recipes differently. As far as this test can tell, the model performs even-handedly across both groups.

<iframe src="assets/fairness_permutation.html" width="800" height="500" frameborder="0"></iframe>
