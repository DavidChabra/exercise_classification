---
title: "Exercise classification"
author: "David Chabra"
date: "8/4/2021"
output: 
    html_document: 
        toc: true
---



## Summary

In this project I will make predictions about the classe variable in the test data set. We will do this by:  
1. Downloading and pre processing the data using selected variables as well as knn imputation and principal components analysis.  
2. We will then fit a random forest model to our data test its accuracy on in and out of sample data. This model ended up having very strong results.  
3. Finally we will use this model to make the predictions on our test data. our final predictions are: B A C A A E D B A A B C B A E E A B B B.

## Downloading Data and Data processing

First lets download the training and testing data. Both were downloaded on 7/30/2021 at 12:21 PM PST.


```r
library(caret)
library(dplyr)

urlTraining <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
urlTesting <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"

training <- read.csv(urlTraining)
testing <- read.csv(urlTesting)
```

There appears to be many missing values, so it may be very beneficial to get rid of variables with too many missing values. We will set a high threshold before dropping a variable so we don't lose information, lets use 50%. After this we'll impute our data's missing values.


```r
set.seed(5050)

evalNas <- function(x) {
    if(sum(is.na(x)) / length(x) >= .5) {
        FALSE
    } else {
        TRUE
    }
}
keepDrop <- sapply(training, evalNas)

training <- training[, keepDrop]
testing <- testing[, keepDrop] # we need to perform same pre processing to testing
```

Now that we've paired down our data we can look at it in more detail. Since the first six observations appear to be mostly based around individual identification, they will probably be mostly irrelevant to this analysis and bog down our model, so we'll remove them.  

We should also make our classe variable a factor so it will work  with out model.


```r
training <- training[, -(1:6)]
testing <- testing[, -(1:6)]

training$classe <- factor(training$classe)
```

Importantly for our model it'll be very helpful if all values are non-missing. In order to do this we'll Impute the missing values using the k nearest neighbors algorithm. We will also compress our data using principal component analysis to capture approximately 95% of variability. In order for knnImpute to work we must make sure every variable has at least one observation. So If any variable of testing has all missing values we will remove that varriable from training and testing.


```r
training <- training[, sapply(testing, function(x) !all(is.na(x)))]
testing <- testing[, sapply(testing, function(x) !all(is.na(x)))]

preProc <- preProcess(training, method = "knnImpute")
training <- predict(preProc, training)
testing <- predict(preProc, testing)

indexes <- names(training) != "classe"
preProc2 <- preProcess(training[, indexes], method = "pca")
training2 <- predict(preProc2, training[, indexes])
testing2 <- predict(preProc2, testing[, indexes])

training2 <- mutate(training2, classe = training$classe)
```

Finally now that our data is processed, we will split up our training data into two data sets so that we have some data to get an estimate for an out of sample error.


```r
inTrain <- createDataPartition(training2$classe, 
                               p = .75, 
                               list = F)
trainingTraining <- training2[inTrain, ]
trainingTesting <- training2[-inTrain, ]
```

We can now work on our model with all of our processing done.

## Fitting model

Since we are looking to classify our data Random forest algorithms seem like a strong contender for our model. Using the ranger package we can quickly fit a traditional random forest.


```r
set.seed(7030)
library(ranger)
model <- ranger(classe ~ ., data = trainingTraining)
```

Next, let's take a look at our predictions and get an estimate for in sample and out of sample error.


```r
inSamp <- predict(model, trainingTraining)

outSamp <- predict(model, trainingTesting)

confusionMatrix(inSamp$predictions,
                trainingTraining$classe)$overall
```

```
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper 
##      1.0000000      1.0000000      0.9997494      1.0000000 
##   AccuracyNull AccuracyPValue  McnemarPValue 
##      0.2843457      0.0000000            NaN
```

```r
confusionMatrix(outSamp$predictions,
                trainingTesting$classe)$overall
```

```
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper 
##      0.9808320      0.9757508      0.9765937      0.9844832 
##   AccuracyNull AccuracyPValue  McnemarPValue 
##      0.2844617      0.0000000            NaN
```

From these results we can see that the in sample error appears to be very low. Our estimate for out of sample error appears to be quite small as well.  
With such high numbers for accuracy we can expect this model to perform quite well on the testing data.

## Testing data

Finally let's come up with our predictions for a testing data!


```r
pred <- predict(model, testing2)

pred$predictions
```

```
##  [1] B A C A A E D B A A B C B A E E A B B B
## Levels: A B C D E
```

These are our final predictions for the classe for these observations.
