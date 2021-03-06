Supervised Machine Learning in R
================
Kasper Welbers & Wouter van Atteveldt
2021-01

  - [Introduction](#introduction)
      - [Packages used](#packages-used)
  - [Obtaining data](#obtaining-data)
  - [Training and test data](#training-and-test-data)
  - [Model 1: a decision tree](#model-1-a-decision-tree)
      - [Validating the model](#validating-the-model)
  - [Using different models: Support vector
    machine](#using-different-models-support-vector-machine)
  - [Parameter tuning](#parameter-tuning)
      - [Grid search](#grid-search)

# Introduction

## Packages used

In this tutorial, we use the following packages: (and probably some
others, feel free to add)

``` r
install.packages(c("tidyverse", "caret", "e1071", "kernlab"))
```

The main library we use is `caret`, which is a high-level library that
allows you to train and test many different models using the same
interface. This pacakage then calls the underlying libraries to train
specific algorithms (such as `kernlab` for kernel-based SVMs).

``` r
library(caret)
```

# Obtaining data

For this tutorial, we will use the ‘german credit data’ dataset:

``` r
library(tidyverse)
d = read_csv("https://www.openml.org/data/get_csv/31/dataset_31_credit-g.arff") 
```

This dataset contains details about past credit applicants, including
why they are applying, checking and saving status, home ownership, etc.
The last column contains the outcome variable, namely whether this
person is actually a good or bad credit risk.

To explore, you can cross-tabulate e.g. home ownership with credit risk:

``` r
table(d$housing, d$class) %>% prop.table(margin = 1)
```

So, home owners have a higher chance of being a good credit risk than
people who rent or who live without paying (e.g. with family), but the
difference is lower than you might expect (74% vs 60%).

Put bluntly, the goal of the machine learning algorithm is to find out
which variables are good indicators of one’s credit risk, and build a
model to accurately predict credit risk based on a combination of these
variables (called features).

Note that this dataset (with only 1000 cases) is pretty small for a
machine learning data set. This has the advantage of making it easy to
download and the models quick to compute, but prediction accuracy will
be relatively poor.

So, after going through the tutorial, you might also want to try to
download a larger dataset (and of course you can also do so right away
and adapt the examples as needed). A good source for this can be
[kaggle](https://www.kaggle.com/datasets), which has lots of datasets
suitable for machine learning.

# Training and test data

The first step in any machine learning venture is to split the data into
training data and test (or validation) data, where you need to make sure
that the validation set is never used in the model training or selection
process.

Fort his, we use the `createDataPartition` function in the `caret`
package, although for a simple case like this we could also directly
have used the base R `sample` function. See the `createDataPartition`
help page for details on other functionality of that function,
e.g. creating multiple folds.

``` r
set.seed(99)
train = createDataPartition(y=d$class, p=0.7, list=FALSE)
train.set = d[train,]
test.set = d[-train,]
```

# Model 1: a decision tree

The most easily explainable model is probably the decision tree. A
decision tree is a series of tests which are run in series, ending in a
decision. For example, a simple tree could be: If someone is a house
owner, assume good credit. If not, if they have a savings account,
they’re good credit, but otherwise they are a bad credit risk.

In caret, we can train a decision tree using the `rpart` method. The
following code trains a model on the data, prediction species from all
other variables (`class ~ .`), using the `train.set` created above.

``` r
library(caret)
tree = train(class ~ ., data=train.set, method="rpart")
```

Unlike many other algorithms, the final model of the decision tree is
interpretable:

``` r
tree$finalModel
```

Each numbered line contains one node, starting from the root (all data).
The first question is whether someone has `checking_status` ‘no
checking’ (i.e. that person has no checking account). If they indeed
do not have a checking account (i.e. `no_checking>=0.5`), you go to node
3, and conclude that they have a good credit risk. If they do have a
checking account, you go to node 2, and then look at the duration of the
requested loan, etc.

To make it easier to inspect, you can also plot this tree:

``` r
plot(tree$finalModel, uniform=TRUE, main="Classification Tree")
text(tree$finalModel, use.n=TRUE, all=F, cex=.8)
```

It might seem counterintuitive that not having a checking account is a
sign of creditworthiness, but apparently that’s what the data says, at
least in the training data:

``` r
table(train.set$checking_status, train.set$class) %>% prop.table(margin=1)
```

So it turns out that people with ‘good’ credit risks either have a full
checking account, or don’t use a checking account at all; but this
implementation of decision trees only looks at one value at a time (in
fact, all categorical variables are turned into dummies before the
learning starts, hence the cryptic `>=0.5`).

## Validating the model

So, how well can this model predict the credit risk? Let’s first see how
it does on the training set:

``` r
train.pred = predict(tree, newdata = train.set)
acc = sum(train.pred == train.set$class) / nrow(train.set)
print(paste("Accuracy:", acc))
```

So, on the training set it gets 76% accuracy, which is not very good
given that just assigning everyone ‘good’ rating would already get you
70% accuracy. Let’s see how it does on the validation set:

``` r
pred = predict(tree, newdata = test.set)
acc = sum(pred == test.set$class) / nrow(test.set)
print(paste("Accuracy:", acc))
```

So, as expected it does even worse on the test set. To see which
outcomes are misclassified, you can create a confusion matrix by
tabulating the predictions and actual outcomes:

``` r
table(pred, test.set$class)
```

Finally, we can use various functions from the `caret` package to get
more information on the performance. The most important one is probably
`confusionMatrix`. Note that this expects the output class to be a
factor, so we convert the `class` column (which tidyverse imported as a
character, possibly not the best choice in this case)

``` r
confusionMatrix(pred, factor(test.set$class), mode="prec_recall")
```

The top displays the confusion matrix, showing that of the actual ‘bad’
credit risks 23 where correctly predicted, but 67 were wrongly predicted
as good. The ‘good’ category does much better, with 196 correct
predictions out of 210. (Note that overpredicting the most common class
is a normal problem for some ML algorithms).

Below that it gives a statistical confirmation that 73% is indeed not
very good, by showing that it is not significantly higher than guessing
(No information).

Finally, the last block gives the common information retrieval metrics
assuming `bad` as the reference class (this can be changed by setting
`positive='good'` on the call). It has decent prediction (if the model
predicts bad, it’s correct in 62% of cases), but really bad recall: only
26% of the bad credit risks are identified.

# Using different models: Support vector machine

Let’s try another model, this time a support vector machine.

``` r
set.seed(1)
m = train(class ~ ., data=train.set, method="svmRadial")
pred = predict(m, newdata = test.set)
acc = sum(pred == test.set$class) / nrow(test.set)
print(paste("Accuracy:", acc))
```

Of course, the same functions given above for e.g. diagnostics and
predicting can be used here as well. Check out the [caret
documentation](http://topepo.github.io/caret/available-models.html) for
a full list of available models, and try some out.

# Parameter tuning

The SVM model trained above, like many ML algorithms, has a number of
(hyper)parameters that need to be set, including sigma (the ‘reach’ of
examples, i.e. how many examples are used as support vectors) and C (the
tradeoff between increasing the margin and misclassifications).

Although the defaults are generally reasonable, there is no real
theoretical reason to choose any value of these parameters. So, the
common approach is to test various options and pick the best performing
one.

In the section on ‘grid search’ below you will learn the recommended way
to do this with a grid search using cross-validation. Before doing that,
however, it is instructive to try some different settings ourselves.

First, It is very important here to not use the test set for that, as
picking the model based on performance on the test set gives a bias on
the accuracy reported by testing on the test set. So, we will split a
small set for selecting models from the train set:

``` r
tune = createDataPartition(y=train.set$class, p=0.25, list=FALSE)
tune.set = train.set[tune,]
train.set2 = train.set[-tune,]
```

To do this, we will create a for loop over different settings of C,
store the results in a list, and then turn the list into a data frame.
Note that we use `as.character(C)` for the list key as fractional
numbers are not allowed as keys.

``` r
Cs = c(.5, 1, 2, 4, 8, 16, 32, 64)
result = list()
for (C in Cs) {
  m = train(class ~ ., data=train.set2, method="svmRadial",tuneGrid = data.frame(.C=C, .sigma=.01) )
  pred = predict(m, newdata = tune.set)
  cm = confusionMatrix(pred, factor(tune.set$class), mode="prec_recall")
  result[[as.character(C)]] = cm$byClass 
  as_tibble(cm$byClass)
}
result = bind_rows(result, .id="C") %>% mutate(C = as.numeric(C))
result
```

And of course we can plot e.g. the effect of C on f-score:

``` r
ggplot(result) + geom_line(aes(x=C, y=F1))
```

## Grid search

The example above showed that the area around `C=16` might be
interesting to explore further. However, we didn’t test the effect of
different values of `sigma`, and it’s quite possible that these
parameters interact.

So, the common thing to do is to do a **grid search**, i.e. an
exhaustive search of a range of possible values of each parameter.
Moreover, rather than using a fixed set for tuning as above, it is
better to use crossvalidation: With a 5-fold cross-validation, each
model is trained 5 times on 80% of the data and tested on the remaining
20%. This is then repeated 5 times with a different split, until every
case has been used in testing once. This has the benefit of getting a
better estimate of model performance with a relatively small tuning set,
while also having multiple trials to control for chance and to get a
measure of variance as well as the mean performance.

Luckily, doing a grid searc with cross-validation is supported out of
the box in caret, and is actually easier than the manual example above.
For example, The following code uses `caret` to run a crossvalidation
(`repeatedcv`) to test different settings of `sigma` and `C`: (See
[chapter 5 of the caret
documentation](https://topepo.github.io/caret/model-training-and-tuning.html)
for more details)

``` r
set.seed(1)
paramgrid = expand.grid(sigma = c(.001, .01, .1, .5), C =  c(1, 2, 4, 8, 16, 32, 64))
traincontrol <- trainControl(method = "repeatedcv", number = 5,repeats=5,verbose = FALSE)
m = train(class ~ ., data=train.set, method="svmRadial", tuneGrid=paramgrid, 
          trControl=traincontrol, preProc = c("center","scale"))
ggplot(m)
```

The `plot` function above by default gives a plot of accuracy against
the used parameters. Looking at this plot, it might be interesting to
zoom in around `sigma=0.01` and `C=8` in a second round.

Note that the plot can be tweaked based on the information in
`m$results`, and other ggplot elements can be added as usual:

``` r
ggplot(m, metric="Kappa", plotType="level") + geom_text(aes(label=round(Kappa,2)))
```

Note finally that when using the `m` for prediction, caret automatically
uses the best model:

``` r
pred = predict(m, newdata = test.set)
acc = sum(pred == test.set$class) / nrow(test.set)
print(paste("Accuracy:", acc))
```

Although arguably after selecting the optimal model, it is best to
retrain that model on the whole training set to get optimal performance.
