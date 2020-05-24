---
title: "PML Project"
author: "Lucas Teixeira"
date: "23/05/2020"
output: 
  html_document:
    keep_md: true
---

## Executive summary

The goal of this study is to provide a method that can predict how well an exercise is done while looking only to the output of sensors that are used to record motion.  

It is highly based on a previous study that can be found in this website: http://web.archive.org/web/20161224072740/http:/groupware.les.inf.puc-rio.br/har

One should look on the section *Weight Lifting Exercise Dataset*.

### The dataset

The dataset is composed by two subsets: training and testing sets.  

Features represents the output of sensors, as well as some calculations (*mean*, *stddev*, etc), additional informations and the output of the activity.

**A** means that the exercise was done properly, while other classes mean the opposite.

### The method

It is possible to infer that, since there are multiple measurements for a single observation, and also some calculation is done, there may be high correlation between features.

In order to avoid this - and also gain the benefit of reducing the number of variables - **PCA** was applied to all dataset.

Training data was also divided in 3:

- Training set (75%) - used to train two models  
- Validation set 1 (12,5%) - used to train one stacked model  
- Validation set 2 (12,5%) - used to estimate test accuracy

After pre-processing data using **PCA** method, two parallel models were computed using Training set: Random Forest and kNN Classifier.

Then, both models were stacked using Random Forest.

## Computations

### Important libraries


```r
library(caret)
library(doParallel)
library(fastAdaboost)
```

## Loading dataset and preparing data

Calculated variables were discarded.


```r
training <- read.csv("pml-training.csv")
testing <- read.csv("pml-testing.csv")

exclude <- grep("^(kurtosis|skewness|max|min|amplitude|var|avg|stddev)",names(training),ignore.case=T)
exclude <- c(exclude,1:7)

training <- training[,-exclude]
testing <- testing[,-exclude]
```

Our prepared datased has now 53 variables, including *classe*.

### Dividing and Pre-processing data


```r
training.ind <- createDataPartition(training$classe,p=0.75,list=F)

validation <- training[-training.ind,]
training <- training[training.ind,]


pProc <- preProcess(training,method="pca",thresh=0.95)
PCA.train <- predict(pProc,training)
PCA.valid <- predict(pProc,validation)
PCA.test <- predict(pProc,testing)

validation.ind <- createDataPartition(PCA.valid$classe,p=0.5,list=F)

PCA.valid1 <- PCA.valid[validation.ind,]
PCA.valid2 <- PCA.valid[-validation.ind,]
```

### Random forest with parallel computing

Here, a random forest model was trained with the training set. It is important to highlight that this is quite time consuming, since the dataset is huge.

There is no need to apply cross-validation when training *rf* models since they are already an ensemble of many different CARTs.


```r
## Random Forest with Parallel computing

cl <- makePSOCKcluster(4)
registerDoParallel(cl)
model.rf <- train(classe~.,data=PCA.train,method="rf")
stopCluster(cl)
```

### kNN with parallel computing

A kNN model is trained as well. This isn't time consuming and we will see that this prediction model has an accuracy very similar to random forest for this dataset.

Cross-validation is applied in order to reduce bias.


```r
tr.ctrl <- trainControl(method="cv", number=10)
cl <- makePSOCKcluster(4)
registerDoParallel(cl)
model.knn <- train(classe~., data=PCA.train,method="knn",trControl = tr.ctrl)
stopCluster(cl)
```

## Accuracies on Training Set

Accuracies on both training and validation set 1 are computed for both models.


```r
pred.rf <- predict(model.rf,PCA.train)
pred.kNN <- predict(model.knn,PCA.train)

acc.rf <- confusionMatrix(pred.rf,PCA.train$classe)$overall[1]
acc.kNN <- confusionMatrix(pred.kNN,PCA.train$classe)$overall[1]

pred.rf.valid1 <- predict(model.rf,PCA.valid1)
pred.kNN.valid1 <- predict(model.knn,PCA.valid1)

acc.rf.valid1 <- confusionMatrix(pred.rf.valid1,PCA.valid1$classe)$overall[1]
acc.kNN.valid1 <- confusionMatrix(pred.kNN.valid1,PCA.valid1$classe)$overall[1]
```

- Random forest accuracies:  
  + 100% - Training;  
  + 97.5% - Validation set 1  
- kNN accuracies:  
  + 98.3% - Training;  
  + 95.6% - Validation set 1

As we can see, they are very similar for both sets, suggesting that for this specific problem we can use either.

But, to reduce bias even more, one can stack both methods using a third method. Here, another random forest was chosen.

### Stacking models


```r
data.stacked <- data.frame(rf = pred.rf.valid1, kNN = pred.kNN.valid1,classe=PCA.valid1$classe)

cl <- makePSOCKcluster(4)
registerDoParallel(cl)

model.stacked <- train(classe~., data=data.stacked, method="rf")

stopCluster(cl)
rm(cl)
```

Validation set 1 is used to train the stacked model. So, it is important to compute both Validation set 1 accuracy and Validation set 2 accuracy. Since it wasn't used to train any model, the latter is used to predict the test accuracy.



```r
pred.stk.valid1 <- predict(model.stacked,data.stacked)
acc.valid1 <- confusionMatrix(pred.stk.valid1,PCA.valid1$classe)$overall[1]

pred.rf.valid2 <- predict(model.rf,PCA.valid2)
pred.kNN.valid2 <- predict(model.knn,PCA.valid2)

data.stacked.valid2 <- data.frame(rf = pred.rf.valid2,kNN = pred.kNN.valid2,classe = PCA.valid2$classe)

pred.valid2 <- predict(model.stacked,data.stacked.valid2)

acc.valid2 <- confusionMatrix(pred.valid2,PCA.valid2$classe)$overall[c(1,3,4)]
```

Validation set 1 accuracy is 97.6% and Validation set 2, and our test accuracy estimate, is 98% with a 95% CI of [97.4%,98.5%].

Therefore, we can assume that the model's out-of-sample error is: 2%.



