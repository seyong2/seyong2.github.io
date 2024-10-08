---
layout: post
title: Ridge Regression 
subtitle: Predict Weight of Fish Species using Multiple Variables
gh-repo: seyong2
gh-badge: [star, fork, follow]
tags: [machine learning, ridge regression]
comments: true
---

In the previous post, we modeled the relationship between the weight of bream fish and several characteristics using the least squares method. However, we also saw that some variables were highly correlated with each other (see figure below). As I mentioned in the first [post](https://seyong2.github.io/2022-07-06-eda_fish/), multicollinearity can cause problems regarding fitting the model and interpreting the results. For example, estimate of a regression coefficient using the least squares method is interpreted as the average change in the dependent variable due to unit change in that predictor when the other independent variables remain constant. But, if the predictors are correlated, shifts in one variable also lead to changes in the others. Then, it becomes complicated for a model to estimate the relationship between the outcome variable and each independent variable because depending on the variables included in the model, the coefficient estimates will alter a lot. Therefore, in today's post, we will have a look at the ridge regression model which is one of the ways to overcome this problem of multicollinearity. 

![corr_mat](https://github.com/seyong2/seyong2.github.io/blob/master/assets/img/figures_multivariate_regression/corr_mat.png?raw=true)

We start by loading necessary libraries and data as usual.

```
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.linear_model import RidgeCV
from sklearn.model_selection import train_test_split

df = pd.read_csv('fish.csv')
df_bream = df.loc[df['Species']=='Bream', :]
df_bream.head()
```

![df_bream_head](https://github.com/seyong2/seyong2.github.io/blob/master/assets/img/figures_multivariate_regression/df_bream_head.png?raw=true)

As in the previous post, we assign $Weight$ to the dependent variable, $y$ and the remaining variables except $Species$ to $X$. To see how the ridge regression model works on the data not used for training, we split the data into training and test set. The test data will contain 33% of the total observations. We also set the seed for the random generator to 1 for reproducibility. Different values for this argument lead to different outcomes. Then, the data for training and testing have 23 and 12 observations, respectively. 

```
X = df_bream.iloc[:, 2:]
y = df_bream['Weight']

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.33, random_state=1)
```

As previously mentioned, Ridge regression is a technique used in machine learning and statistics to analyze multiple regression data that suffer from multicollinearity (when independent variables are highly correlated). It is a type of regularization method that introduces a penalty term to the ordinary least squares (OLS) regression to prevent overfitting and improve the model's generalization to unseen data.

Recall the multivariate regression model from the last post.

$\hat{Weight}=\hat{\beta}_0 + \hat{\beta}_1Length1 + \hat{\beta}_2Length2 + \hat{\beta}_3Length3 + \hat{\beta}_4Height + \hat{\beta}_5Width$

To find the coefficient values that best describes the data, we used the least squares method by minimizing the following cost function.

$min_{\beta} \sum_{i=1}^{n}(y_i-X_i\beta)^2$

Instead of minimizing the sum of squared residuals (SSR) that does the least squares method, ridge regression estimates the parameters minimizing not only the SSR but also $\lambda\sum_{j=1}^{p}\beta_j^2$ where $\lambda$ is the regularization penalty that determines the amount of penalty given to the least squares method. $\lambda$ can take a value between 0 and positive infinity and the larger its value is, the more severe the penalty is. To obtain the optimal value for $\lambda$, we use 10-fold cross-validation and find the value that produces the lowest variance in the coefficients.

$min_{\beta} \sum_{i=1}^{n}(y_i-X_i\beta)^2+\lambda\sum_{j=1}^{p}\beta_j^2$

The parameter estimates using ridge regression are in general smaller than those using the least squares method. This indicates that predictions made by the ridge regression model are usually less sensitive to changes in predictors than the least squares model. Then, now we fit a ridge regression model to the training data.

```
reg_ridge = RidgeCV(alphas=np.arange(0, 10, 0.1), scoring='neg_mean_squared_error', cv=10).fit(X_train, y_train)
reg_ridge.alpha_
```

The argument 'alphas' in function *RidgeCV* is the list of possible values for the regularization parameter, $\lambda$. We use 10-fold cross-validation to obtain the one that produces the smallest average mean squared error (MSE) as specified in argument 'scoring'. After the training, the optimal value of $lambda$ is found to be 1.0, meaning that imposing a penalty on the coefficients improves the model performance. To verify it, we also fit a multivariate regression model to the training data and compare the results.

```
reg_no_ridge = LinearRegression().fit(X_train, y_train)

beta_hat_ridge = pd.DataFrame([reg_ridge.intercept_]+list(reg_ridge.coef_), index=['Intercept']+list(X.columns), columns=['beta_hat_ridge']).T
beta_hat_no_ridge = pd.DataFrame([reg_no_ridge.intercept_]+list(reg_no_ridge.coef_), index=['Intercept']+list(X.columns), columns=['beta_hat_no_ridge']).T
pd.concat([beta_hat_ridge, beta_hat_no_ridge])
```

![beta_hat_comparison](https://github.com/seyong2/seyong2.github.io/blob/master/assets/img/figures_ridge_regression/beta_hat_comparison.png?raw=true)

As we look at the estimates for the regression coefficients, those estimated by the ridge regression model are smaller in absolute values compared to the least squares ones.

```
def SSR(y, y_hat):
    return ((y-y_hat)**2).mean()

y_hat_ridge = reg_ridge.predict(X_test)
SSR_ridge = SSR(y_test, y_hat_ridge)

y_hat_no_ridge = reg_no_ridge.predict(X_test)
SSR_no_ridge = SSR(y_test, y_hat_no_ridge)

pd.DataFrame.from_dict({"SSR_ridge": [SSR_ridge], "SSR_no_ridge": [SSR_no_ridge]})
```

![ssr_comparison](https://github.com/seyong2/seyong2.github.io/blob/master/assets/img/figures_ridge_regression/ssr_comparison.png?raw=true)

Finally, let's see how the two models work on the test data. The SSR values indicate that the ridge regression model has less variance than the multivariate regression one. In other words, since the ridge regression model produces better predictions, it is recommended that we opt for a ridge regression model instead of a multivariate regression model in case we observe multicollinearity in data. However, there are some limitations when using a ridge regression model and one of them is that the coefficients are shrunk but not eliminated, which can make model interpretation challenging when dealing with many variables. For this reason, in the next post, we will explore lasso regression (which uses L1 penalty) which performs variable selection.
