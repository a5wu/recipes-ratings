---

# layout: home
# theme: jekyll-theme-cayman
---

# Predicting Recipe Ratings With Content and Language Models

🤯

Table of Contents

1. [Introduction](#introduction)
2. [Data Cleaning and Exploratory Data Analysis](#data-cleaning-and-exploratory-data-analysis) 
  - [Data Cleaning](#data-cleaning)
  - [Univariate Analysis](#bivariate-analysis)
  - [Bivariate Analysis](#bivariate-analysis)
  - [Interesting Aggregates](#interesting-aggregates)
3. [Assessment of Missingness](#assessment-of-missingness)
  - [NMAR Analysis](#nmar-analysis)
  - [Missingness Dependency](#missingness-dependency)
4. [Hypothesis Testing](#hypothesis-testing)
  - [Setting Up the Test](#setting-up-the-test)
  - [Permutation Test](#permutation-test)
5. [Framing a Prediction Problem](#framing-a-prediction-problem)
6. [Baseline Model](#baseline-model)
7. [Final Model](#final-model)
8. [Fairness Analysis](#fairness-analysis)

## Introduction

Building models that can predict ratings is a problem that falls under the recommendation system domain. Collaborative filtering models work best in recommendations, but that does not mean that conventional content-based ML models can't perform well too. Indeed, many of the largest companies use a combination of both. In this project I will build a model that can predict the ratings of recipes using a combination of a tree-based model and a natural language model. The dataset of focus will be ratings and recipes scraped from Food.com(cite). The dataset consists of 84K recipes and 730K reviews between 2000 and 2018. I will show the process of building a model that can effectively predict user ratings. I will look into what factors contribute to a recipe's ratings. This ability is very useful and can be generalized to any type of domain where recommendations are useful, such as social media, video, product recommendations. 

After inner joining the recipes and reviews datasets, here are the list of all the initial relevant features in the dataset that will be used to build the model:

|Column          |Description                        |
|----------------|-----------------------------------|
|`name`          |Recipe Name                        |
|`recipe_id`     |Recipe ID                          |
|`minutes`       |Minutes to prepare recipe          |
|`contributor_id`|User ID who submitted this recipe  |
|`submitted`     |Date recipe was submitted          |
|`tags`          |Food.com tags for recipe           |
|`nutrition`     |Nutrition information in the form [calories (#), total fat (PDV), sugar (PDV), sodium (PDV), protein (PDV), saturated fat (PDV), carbohydrates (PDV)]; PDV stands for “percentage of daily value”|
|`n_steps`       |Number of steps in recipe          |
|`steps`         |Text for recipe steps, in order    |
|`description`   |User-provided description          |    
|`ingredients`   |Ingredients name                   |
|`n_ingredients` |Number of ingredients for recipe   |
|`user_id`       |User ID                            |
|`date`          |Date of review                     |
|`rating`        |Rating given                       |
|`review`        |Review text                        |

The merged dataset has 234,428 total rows

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

In order to perform EDA on the dataset, it must first be cleaned and preprocessed. 

The `ratings` column contains many values that have 0 stars, meaning that there was a review but no rating. This would inadvertently affect the calculation of the average ratings, so we will replace all instances of 0 with `nan`:

```
df['rating'] = df['rating'].replace(0, np.nan)
```

Here we convert some columns to their correct types:
```
df['tags'] = df['tags'].apply(literal_eval)
df['nutrition'] = df['nutrition'].apply(literal_eval)

df[['submitted', 'date']] = df[['submitted', 'date']].apply(pd.to_datetime)

df[['name','description', 'review']] = df[['name','description', 'review']].astype(str)
```

The following feature columns are added:

|Column                |Description                       |
|----------------------|----------------------------------|
|`rating_missing`      |Whether the rating was originally missing|
|`recipe_avg_rating`   |Recipe's average rating           |
|`user_avg_rating`     |User's average rating             |
|`contributor_id_count`|Occurences of contributor_id      |
|`user_id_count`       |Occurences of user_id             |
|`recipe_id_count`     |Occurences of recipe_id           |
|`n_tags`              |Number of tags                    |
|`description_length`  |Length of description (characters)|
|`review_length`       |Length of review (characters)     |


The `nutrition` column is one-hot encoded to get the following columns:

|Column                |
|----------------------|
|`calories (#)    `       |
|`total fat (PDV)`        |
|`sugar (PDV)`            |
|`sodium (PDV)`           |
|`protein (PDV)`          |
|`saturated fat (PDV)`    |
|`carbohydrates (PDV)`    |


<!-- |`calories (#)`|`total fat (PDV)`|`sugar (PDV)`|`sodium (PDV)`|`protein (PDV)`|`saturated fat (PDV)`|`carbohydrates (PDV)`| -->


There are more columns that can be one-hot encoded like `tags` and `ingredients`, but with so many unique values they would take up too much space in memory. A sparse matrix will be more difficult to work with, so we'll keep this in mind for now and one-hot encode them later. 

```
df.head(5)
```

|name|minutes|contributor_id|submitted|...|sodium (PDV)|protein (PDV)|saturated fat (PDV)|carbohydrates (PDV)|
|----------------------------------|--|-------|-----------|---|----|----|-----|----|
|1 brownies in the world best ever |40|985201	|2008-10-27	|...|3.0 |3.0	|19.0	|6.0 |
|1 in canada chocolate chip cookies|45|1848091|2011-04-11	|...|22.0|13.0|51.0	|26.0|
|412 broccoli casserole            |40|50969  |2008-05-30	|...|32.0|22.0|36.0	|3.0 |
|412 broccoli casserole	           |40|50969	|2008-05-30	|...|32.0|22.0|36.0	|3.0 |
|412 broccoli casserole            |40|50969  |2008-05-30	|...|32.0|22.0|36.0	|3.0 |

Plotting the rating distribution:

### Univariate Analysis

<iframe
  src="fig1.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

As we can see the data is imbalanced, as is often the case in the real world. There are many things we can do to address this, including oversampling, undersampling, synthetic data, etc., but all of those would come with their own challenges/drawbacks. We'll leave it like this. 

### Bivariate Analysis

As we can see, the majority of ratings on Food.com are 5 stars, with some 4 stars, and few 1-3 stars. 0 stars mean that the rating is missing. We will deal with this in the next section.

<iframe
  src="fig2.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Here we can see that for the most popular recipes, the calorie count of the recipe is mostly under 500

### Interesting Aggregates

Here is a pivot table of the 5 most popular recipes:

|name|average_rating|review_count|average_review_length (char)|		
|---|----------------|------------|---------------------------|
|microwave chocolate mug brownie|	4.24|	295	|228.21|
|mexican stack up rsc	|4.99|	217	|108.79|
|traditional irish shepherd s pie	|4.76|	216	|334.27|
|bacon lattice tomato muffins rsc	|5.00|	192|	85.26|
|quick cinnamon rolls no yeast|	3.96|	162|	351.85|

We can see that even the top recipe has no more than 300 reviews, suggesting that the site has a lot of categories and niches to satisfy users. 


## Assessment of Missingness

### NMAR Analysis

How we deal with missing data can greatly influence the performance of our model. First we must determine what kind of data is missing. In our merged dataframe, the only significant column that is missing values is ratings, with around 15k values missing. 

Why is all this important. Only when data is MAR can we use model imputation

```
missing = df[]
df.sample(5)
```

Sampling small pieces of data from the entire dataset, at first glance there appears to be both positive and negative sentiment reviews when the rating is missing.

This likely makes `rating` MAR (missing at random) rather than NMAR. This is because the missingness of `rating` likely depends on other columns in the data, like `review`. There are many reasons why people would leave a review but not a rating, and you can figure out why some of the ratings are missing by looking at the reviews. Some people left a review only to ask questions. Some haven't done the recipe. Maybe they didn't like the recipe but also didn't want to bring down the score. 

### Missingness Dependency

Let's do a test to see if this is true with permutation testing:

Distribution of column `minutes` when `rating` is missing

<iframe
  src="fig3.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

It doesn't appear that the `minutes` influences whether `rating` is missing in any statistically significant way never use language that implies an absolute conclusion; since we are performing statistical tests and not randomized controlled trials, we cannot prove that either hypothesis is 100% true or false.

Distribution of column `review_length` when `rating` is missing

<iframe
  src="fig4.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

As we can see, this plot demonstrates that `review_length` influences whether `rating` is missing in a statistically significant way.

For our purposes, since we are trying to predict `rating`, it would make sense not to impute missing values and train on them. Therefore, we will drop all rows that have missing ratings. After we inspect the dataframe, we no longer have any missing values.

## Hypothesis Testing

### Setting Up the Test

To understand the data better before building a model, we will perform a hypothesis test.

Null Hypothesis: The average preparation time for the top 5% most popular recipes is equal to the average preparation time for the remaining recipes

Alternate Hypothesis: The average preparation time for the top 5% most popular recipes is not equal to the average preparation time for the remaining recipes.

Test Statistic: Difference in mean preparation time

Significance Level: 0.05

### Permutation Test

After performing the permutation test, the resulting p-value of 0.002 is well below the signifance level of 0.05, so we can say that we can reject the null hypothesis and say there is sufficient evidence to support the fact that the average preparation time for the top 5% most popular recipes is not equal to the average preparation time for the remaining recipes.


## Framing a Prediction Problem

The prediction problem we will tackle is to build a model that can predict what rating a user would give a recipe given that they already gave a review. This is useful due to sentiment analysis and that can predict what user's will interact with so that we can serve recommendations, and also impute missing data. This task can be generalized to a variety of real-world recommendations tasks, such as recommending similar movies/music or similar social media posts. This will be a classification problem as it will predict `rating`. The metric used to evaluate this would be the f1_score, as the dataset is imbalanced. 

## Baseline Model

We add the following tfidf features for all the non-numeric columns:

|Column                |
|----------------------|
|`name_tfidf`         |
|`steps_tfidf`        |
|`description_tfidf`  |
|`reviews_tfidf`      |


Afterwards we One-hot encode the features of columns `tags` and `ingredients`. Since they have so many unique values we will convert the features into a sparse matrix and also sparsify the entire dataframe.

The baseline model will be trained using a CatBoostRegressor, a type of GBM (Gradient Boosting Machine) model. This type of model is state of the art on many tabular-related tasks. Since the model has such high dimensionality and is so large, it is impractical to train on it as it'll be too slow and we would quickly run out of memory. Therefore, we will perform linear dimensionality reduction through truncated SVD, which, unlike regular PCA, can work on sparse matrices efficiently. After training, the model achieves an accuracy of 0.72. This is pretty bad(If we predicted everything to be 5, we would get 77%).

## Final Model

<iframe
  src="model.png"
  width="800"
  height="340"
  frameborder="0"
></iframe>

For the final model it is a combination of two models, a CatBoostRegressor and a fine-tuned DistilBert model. Intuitively the best predictor of ratings are the reviews. To extract data from the reviews, we need to make use of natural language processing. I finetuned the DistilBert model on Sequence Classification, so that it can predict the ratings from 1-5 based off only the written reviews in the training data. This model achieved an accuracy score of 84% on the testing data.

For the CatBoostRegressor model, temporal and cyclical features were added to improve the capturing of time series data. There are many recipes centered around holidays, which can affect ratings. After doing gridsearch to find the optimal hyperparameters, a CatBoost Regressor with learning_rate=0.01 and depth=6 was trained again, this time achieving an accuracy of 88%. What a surprise! I expected the language model to perform better. I ensembled the predictions of the CatBoostRegressor with the DistilBert model, achieving a final accuracy of 90%. 

Even though both models scored under 90%, they were able to capture different enough aspects of the data that when their predictions combined, performance was improved.

## Fairness Analysis

Knowing if the model performs better or worse from one group to another is helpful for a variety of reasons. Let's try to perform a fairness analysis. Let's see if it performs worse for individuals with low engagement(<=10 reviews) vs individuals with high engagement.

Null hypothesis: model has the same RMSE on low engagement users as it does high engagement users

Alternate hypothesis: model does not have the same RMSE on low engagement users as it does high engagement users

After performing the permutation test, with the p-value of 0.001, we reject the null hypothesis and say that there is a statistically significant difference in model performance between high engagement and low engagement users