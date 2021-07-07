---
title: "Disaster Relief Project: Part I"
author: "Christian Schroeder"
date: "Jul 07, 2021"
output:
  html_document:
    keep_md: true
    number_sections: true    
    toc: true
    toc_float: true
    theme: cosmo
    highlight: espresso
---

<!--- Below are global settings for knitr. You can override any of them by adding the changes to individual chunks --->



<!--- Change font sizes (or other css modifications) --->

```{=html}
<style>
h1.title {
  font-size: 2.2em; /* Title font size */
}
h1 {
  font-size: 2em;   /* Header 1 font size */
}
h2 {
  font-size: 1.5em;
}
h3 { 
  font-size: 1.2em;
}
pre {
  font-size: 0.8em;  /* Code and R output font size */
}
</style>
```
**DS 6030 \| Spring 2021 \| University of Virginia**

------------------------------------------------------------------------

# Introduction

In early 2010 the Caribbean nation of Haiti was devastated by a magnitude 7.0 earthquake. This catastrophe leveled many buildings, and resulted in numerous lives lost. Most people around the world are familiar with this disaster and its level of destruction, but few are as familiar with the after-effects it had on those that lived but their homes didn't.

In the wake of the earthquake, an estimated five million people, more than 50% of the population at the time, were displaced, with 1.5 million of them living in tent camps (<https://www.worldvision.org/disaster-relief-news-stories/2010-haiti-earthquake-facts>). This wide-spread displacement of people across a country with worsened infrastructure made relief efforts more difficult. Teams needed an accurate way to locate these individuals so they could provide aid.

In an effort to assist the search, a team from the Rochester Institute of Technology (RIT) collected aerial imagery of the country. These images were then converted into datasets of Red, Green, and Blue (RGB) values. Using this RGB data from the imagery, with the knowledge that many of the displaced people were using distinguishable blue tarps as their shelter, I attempted to predict the locations of these blue tarps using several classification models.

My goal in this analysis was to determine the optimal model for locating displaced people. To determine those models, I focused on two statistics; accuracy, and false negative rate (FNR). Given the context of the situation, I believed the FNR to be a very important metric, much more than the false positive rate (FPR), because I wanted to make sure no displaced individual was being overlooked. I would much rather have over-classified and found no one at a certain location than under-classify and not provide aid to someone in need. But, it is important to note that these efforts still needed to be made in a timely manner, so grossly over-classifying to get the smallest FNR was not the optimal solution. So, a combination of accuracy, FNR, and FPR was used to determine these models.

# Training Data / EDA

### Packages

Several packages were used for this analysis, but the most important was the Caret package. This package allowed me to easily reproduce different trained models, use the same 10-fold cross-validation on each, and iteratively test different threshold, alpha, and lambda values when needed.


```r
# Load Required Packages
library(tidyverse)
library(htmlTable)
library(ggplot2)
library(GGally)
library(ROCR)
library(knitr)
library(caret)
library(plotly)
library(gridExtra)
```

### The Data

The data provided for this analysis consists of four fields. "Class" defines land-classification of the pixel, whether that be vegetation, soil, rooftop, various non-tarp, or blue tarp. The other three classes, "Red," "Green," and "Blue," pertain to the RGB color values of the pixel. The RGB values are what were used as the predictors in the models.


```r
data <- read.table("HaitiPixels.csv", sep=",", header=TRUE)
```

To make the analysis easier, a new binary field was added to the dataset representing whether or not the pixel was a "Blue Tarp." This was done to simplify the modeling process, because we are not interested in predicting the other classes. The new field, "ClassTarp," was used as the response in the models.


```r
data$ClassTarp <- ifelse(data$Class == "Blue Tarp", "Yes", "No")
data$ClassTarp <- factor(data$ClassTarp, levels = c("No", "Yes"))
```

### Exploratory Analysis

In exploring the correlations of the color values, I noticed it was less likely to see any Blue Tarp pixels with Red values above 200. Also, only pixels with a Blue value above \~100 were classified as Blue Tarp. This may cause issues in areas that had strong shadows from trees or buildings.

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-3-1.png" width="75%" style="display: block; margin: auto;" />

Because the pixel values are comprised of three colors, I thought a 3D scatter plot would be more appropriate for showing how all the values played into each classification. In the plot below, it is easy to see a strong distinction between the Blue Tarp pixels and the others. There is very little bleed into the other category's areas. Also, I noticed the volume of Blue Tarp pixels grows in size as all values increase. This could be because at lower color values, the pixels get very dark, making it more difficult to classify a pixel as Blue Tarp.


```{=html}
<div id="htmlwidget-313a45fc3190c5cc19ee" style="width:75%;height:480px;" class="plotly html-widget"></div>
```

The above plots show that it is unlikely to find a Blue Tarp pixel with a high Red value, but not as unlikely for a pixel with a high Green value. I think this is because Green and Blue are more visually related than Red and Blue.

After looking at the relationships between the three color values, and how they interact with the land classifications, I believe there are cases where a model with an added interaction between Green and Blue could be beneficial to the overall accuracy.

# Model Training

## Set-up

#### Variables

Several values were set and functions created to make the model building and analysis process more efficient. This included setting a universal seed to make sure everything is reproducible. Folds were created from the data and a trainControl() object so the same 10-fold cross validation was performed on all models.


```r
useed <- 13
folds <- createFolds(data$ClassTarp, k=10, list = TRUE, returnTrain = TRUE)
```

The trainControl() object was a very useful part of the Caret package to use for this analysis. It is used to control how the models are trained and what information to return from that training. By setting method to "cv," number to 10, and then using the previously created folds as the index, I was able to easily reproduce the same 10-fold cross-validation on all models being trained. Being able to save the prediction values was also very useful to plotting the ROC curves. ClassProbs was also set to TRUE so that the probabilities for both "No" and "Yes" classes were calculated and returned.


```r
# https://www.rdocumentation.org/packages/caret/versions/6.0-88/topics/trainControl
control <- trainControl(method="cv",
                        number=10,
                        index=folds,
                        savePredictions=TRUE,
                        classProbs=TRUE)
```

#### Functions

The threshold_test() function was created to produce a dataframe of summary statistics from the inputted model using multiple thresholds, in order to determine the most desirable value. The function makes use of the thresholder() function from the Caret package, which takes in a model, a sequence of threshold values, and a list of summary statistics, and returned a dataframe of those statistics calculated for each threshold value. Because I was also interested minimizing the false negative rate, I added that to the dataframe before it was returned.


```r
# https://www.rdocumentation.org/packages/caret/versions/6.0-88/topics/thresholder
th <- seq(0.1,0.9, by=0.1)
statsWanted <- c("Accuracy","Kappa","Sensitivity","Specificity","Precision")

threshold_test <- function(model) {
  set.seed(useed) # just in case
  stats <- thresholder(model,
                       threshold=th,
                       statistics=statsWanted)
  stats$falseNeg <- 1 - stats$Sensitivity
  stats$falsePos <- 1 - stats$Specificity
  return(stats)
}
```

The plotROC() function was created to make ROC curve plotting easier for each model, and to calculate and append the AUROC value to the stats generated by the threshold_test() function. This function takes in the model, the statistics of the selected "optimal" model, and a string to add to the title of the plot. Because the model inputted is actually a train() object created by the Caret package, it would already have the prediction values saved as a variable "\$pred," which was then used to calculate the true positive rates (TPR) and false positive rates (FPR) to plot the ROC curve.


```r
plotROC <- function(model, stats.selected, model_name) {
  set.seed(useed)
  
  # https://www.statmethods.net/management/sorting.html
  prob <- model$pred[order(model$pred$rowIndex),]
  
  # https://rviews.rstudio.com/2019/03/01/some-r-packages-for-roc-curves/
  rates <- prediction(prob$Yes,as.numeric(data$ClassTarp))
  roc <- performance(rates, measure="tpr", x.measure ="fpr")
  plot(roc, main=paste("ROC Curve:", model_name))
  lines(x=c(0,1),y=c(0,1),col="red")
  
  auc <- performance(rates,"auc")
  stats.selected <- stats.selected %>% mutate(AUROC = auc@y.values[[1]])
  return(stats.selected)
}
```

#### Formulas

For each model type I started by comparing the measured accuracy, kappa, and other summary statistics of two formulas. The first formula being the basic full formula,

$$
ClassTarp = Redx_1 + Greenx_2 + Bluex_3
$$

and the second being that same formula with an added interaction,

$$
ClassTarp = Redx_1 + Greenx_2 + Bluex_3 + x_4(Green:Blue)
$$

I did this believing there may be a case where the added interaction proved to make the model more accurate, or maybe reduced the FNR a desirable amount.

## Logistic Regression

### Model Training

To create the logistic regression models, I used the train() function of the Caret package and set the family to "binomial" and the method to "glm" for generalized linear model. Also, the trainControl object was brought in to perform 10-fold cross-validation on the model to calculate the accuracy and kappa statistics.


```r
# https://www.youtube.com/watch?v=BQ1VAZ7jNYQ
set.seed(useed)

data.log <- train(ClassTarp~Red+Green+Blue,
                  data=data,
                  family="binomial",
                  method="glm",
                  trControl=control)

data.log.i <- train(ClassTarp~Red+Green+Blue+Green:Blue, data=data,
                  family="binomial",
                  method="glm",
                  trControl=control)

rbind(c("R+G+B",data.log$results[2],data.log$results[3]),
      c("R+G+B+G:B",data.log.i$results[2],data.log.i$results[3]))
```

```
#>                  Accuracy  Kappa    
#> [1,] "R+G+B"     0.9953037 0.9210112
#> [2,] "R+G+B+G:B" 0.9955092 0.9259429
```

Comparing the accuracy and kappa statistics of each model, I saw that the values were identical, implying that adding the interaction would not be beneficial enough to justify the added complexity. However, there was still a chance that the added interaction could benefit other statistics, so I ran the threshold test on both models to compare their performances across multiple threshold values.

### Determining Threshold

Using the threshold_test() function I tested each model's performance and selected the threshold that produced the highest accuracy for each.


```r
data.log.thres <- threshold_test(data.log)
data.log.thres[2:9] %>% slice_max(Accuracy)
```

```
#>   prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1            0.7 0.9956832 0.9287925   0.9984809   0.9109789  0.997064
#>      falseNeg   falsePos
#> 1 0.001519127 0.08902112
```

```r
data.log.i.thres <- threshold_test(data.log.i)
data.log.i.thres[2:9] %>% slice_max(Accuracy)
```

```
#>   prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1            0.8 0.9960469 0.9375549   0.9971904   0.9614349  0.998724
#>      falseNeg   falsePos
#> 1 0.002809582 0.03856509
```

The highest accuracy threshold for the no-interaction model was 0.7, while that of the interaction model was 0.8. The accuracy of the interaction model was higher, but the FNR of the no-interaction model was lower. Because I wanted to reduce the FNR, and the decrease in accuracy was insignificant, I believed the best choice for logistic regression was the no-interaction model, at the threshold 0.7.

### ROC Curve

The ROC curve is a good way to visually represent the classification abilities of a model, plotting the TPR against the FPR at numerous threshold values. The ROC curves for this analysis are built from the out-of-sample data predictions provided from the train() function. I used the plotROC() function to both plot the ROC curve and calculate the AUROC value of the model.


```r
log.selected <- data.log.thres[2:9] %>% slice_max(Accuracy)
log.selected <- plotROC(data.log, log.selected, "Logistic Regression")
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-11-1.png" width="75%" style="display: block; margin: auto;" />

The ROC curve for the selected logistic regression model was very good. The true positive rate reaches 1 barely above a false positive rate of 0. The AUROC for this model was 0.998491.

## Linear Discriminant Analysis (LDA)

### Model Training

To create the LDA models, I used the train() function again, but set the method to "lda." By using the same trainControl() object throughout all the model building, I was able to perform 10-fold cross-validation on the model to calculate the accuracy and kappa statistics, using the same folds every time.


```r
set.seed(useed)

data.lda <- train(ClassTarp~Red+Green+Blue, data=data,
                  method="lda",
                  trControl=control)

data.lda.i <- train(ClassTarp~Red+Green+Blue+Green:Blue, data=data,
                    method="lda",
                    trControl=control)

rbind(c("R+G+B",data.lda$results[2],data.lda$results[3]),
      c("R+G+B+G:B",data.lda.i$results[2],data.lda.i$results[3]))
```

```
#>                  Accuracy  Kappa    
#> [1,] "R+G+B"     0.9839345 0.7532594
#> [2,] "R+G+B+G:B" 0.9943391 0.9031135
```

Unlike what I saw with the logistic regression models, there was a difference in the interaction and no-interaction models for LDA. The kappa value for the no-interaction model was relatively lower. It seemed the best model to use was the interaction model because of the better metrics, but I wanted to be sure, so I ran the threshold test on both models.

### Determining Threshold


```r
data.lda.thres <- threshold_test(data.lda)
data.lda.thres[2:9] %>% slice_max(Accuracy)
```

```
#>   prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1            0.1 0.9846144 0.7470515    0.992633   0.7418378 0.9914836
#>      falseNeg  falsePos
#> 1 0.007366991 0.2581622
```

```r
data.lda.i.thres <- threshold_test(data.lda.i)
data.lda.i.thres[2:9] %>% slice_max(Accuracy)
```

```
#>   prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1            0.7 0.9944498 0.9059849   0.9986932   0.8659806 0.9955873
#>      falseNeg  falsePos
#> 1 0.001306787 0.1340194
```

The previous determination that the interaction model was more accurate was upheld by the threshold test, where it had a higher accuracy and precision for all thresholds tested. The highest accuracy threshold for the no-interaction model was 0.1, while that of the interaction model was 0.7. With a higher accuracy, kappa, and precision, and a much lower FNR, the interaction model at threshold 0.7 was the clear better choice.

### ROC Curve

I used the plotROC() function to both plot the ROC curve and calculate the AUROC value of the model.


```r
lda.selected <- data.lda.i.thres[2:9] %>% slice_max(Accuracy)
lda.selected <- plotROC(data.lda.i, lda.selected, "LDA")
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-14-1.png" width="75%" style="display: block; margin: auto;" />

The ROC curve for the selected LDA model is not as sharp to the top-left corner as the logistic regression model's ROC Curve. There was a sharp increase to a TPR of 0.8, and then a drop-off in that growth to a jagged incline to a TPR of 1. The AUROC for this model was 0.9952404.

## Quadratic Discriminant Analysis (QDA)

### Model Training

To create the QDA models, I used the train() function again, but set the method to "qda." By using the same trainControl() object throughout all the model building, I was able to perform 10-fold cross-validation on the model to calculate the accuracy and kappa statistics, using the same folds every time.


```r
set.seed(useed)

data.qda <- train(ClassTarp~Red+Green+Blue, data=data,
                  method="qda",
                  trControl=control)

data.qda.i <- train(ClassTarp~Red+Green+Blue+Green:Blue, data=data,
                    method="qda",
                    trControl=control)

rbind(c("R+G+B",data.qda$results[2],data.qda$results[3]),
      c("R+G+B+G:B",data.qda.i$results[2],data.qda.i$results[3]))
```

```
#>                  Accuracy  Kappa    
#> [1,] "R+G+B"     0.9945921 0.9056434
#> [2,] "R+G+B+G:B" 0.9891368 0.8374004
```

Expectedly, the accuracies of the no-interaction and interaction models were very similar. But unlike the outcomes of the logistic regression and LDA model comparisons, the QDA models showed the no-interaction model being more accurate and with a higher kappa. From this, I was very doubtful that the added interaction could benefit the model at all, but I thought it was worth verifying through the threshold test.

### Determining Threshold


```r
data.qda.thres <- threshold_test(data.qda)
data.qda.thres[2:9] %>% slice_max(Accuracy)
```

```
#>   prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1            0.6 0.9947503 0.9089959   0.9995916   0.8481808 0.9950084
#>       falseNeg  falsePos
#> 1 0.0004083659 0.1518192
```

```r
data.qda.i.thres <- threshold_test(data.qda.i)
data.qda.i.thres[2:9] %>% slice_max(Accuracy)
```

```
#>   prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1            0.1 0.9943549 0.9048343   0.9986279   0.8649881 0.9955552
#>      falseNeg  falsePos
#> 1 0.001372111 0.1350119
```

The highest accuracy threshold for the no-interaction model was 0.7, while that of the interaction model was 0.1. The no-interaction model was verified as the better choice at this point, with stronger values in almost every metric, especially the FNR. With a higher accuracy, and kappa, and a much lower FNR, the no-interaction model at threshold 0.7 was the better choice.

### ROC Curve

I used the plotROC() function to both plot the ROC curve and calculate the AUROC value of the model.


```r
qda.selected <- data.qda.thres[2:9] %>% slice_max(Accuracy)
qda.selected <- plotROC(data.qda.i, qda.selected, "QDA")
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-17-1.png" width="75%" style="display: block; margin: auto;" />

The ROC curve for the selected QDA model was pretty good, and had a similar point of TPR where the sharp increase stops. It was similar to the others, but not as jagged in the corner as the LDA ROC curve. The AUROC for this model was 0.997053.

## KNN

### Model Training

To create the KNN models, I used the train() function again, but made several changes to the settings. For the KNN models, a list of k values, from 0 to 50 at intervals of 5, was used for the tuneGrid variable in the train() function. The tuneGrid acted as a list of options to train models on and determine from those models the best one. Adding this option made the train() function choose the optimal k-value for the model out of the ones given.

It was important to consider the additional argument for the train() function, preProcess. Adding "center" and "scale" to that argument would normalize the color values, and is a common practice I saw online when using Caret to train a KNN model. However, because the RGB values represent real-world color that was normalized to a scale of 0-255, I did not believe it was necessary to normalize the data again.


```r
set.seed(useed)

klist <- data.frame(k=seq(0,50,5))
klist[1,] <- 1

data.knn <- train(ClassTarp~Red+Green+Blue, data=data,
                  tuneGrid = klist,
                  method="knn",
                  metric="Accuracy",
                  trControl=control)

data.knn.i <- train(ClassTarp~Red+Green+Blue+Green:Blue, data=data,
                    tuneGrid = klist,
                    method="knn",
                    metric="Accuracy",
                    trControl=control)

data.knn$results %>% slice_max(Accuracy)
```

```
#>   k  Accuracy     Kappa   AccuracySD     KappaSD
#> 1 5 0.9972802 0.9560866 0.0005312775 0.008689727
```

```r
data.knn.i$results %>% slice_max(Accuracy)
```

```
#>   k  Accuracy     Kappa   AccuracySD    KappaSD
#> 1 1 0.9946237 0.9116239 0.0009916501 0.01636618
```

#### Tuning Parameter

For all levels of k, the no-interaction and interaction model had very similar outcomes, so I believed it best to work with the no-interaction model for simplicity.

The output of the train() function shows the optimal k value to be 10, based on accuracy. I was hesitant to use 10, because the accuracy remained above 0.99 for all k values tested. I think 10 could prove to be a problem for a larger dataset, possibly causing errors in variance.

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-19-1.png" width="75%" style="display: block; margin: auto;" />

So, I chose to go with k=20 to retain a similar accuracy but with more potential stability. Given the nature of the train() function, I needed to re-run it with 20 as the only available value for k to get the model object.


```r
data.knn20 <- train(ClassTarp~Red+Green+Blue, data=data,
                  tuneGrid = data.frame(k=seq(20,20,1)),
                  method="knn",
                  metric="Accuracy",
                  trControl=control)
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-21-1.png" width="75%" style="display: block; margin: auto;" />

When comparing the metrics of a k=10 (blue) and k=20 (red) model, I see that they are very similar. The k=10 model is technically more accurate, but the FNR, FPR, and TPR are very similar. From that similarity I believe it is fine to sacrifice a small amount of accuracy to handle potential variability in the future.

### Determining Threshold


```r
data.knn.thres <- threshold_test(data.knn20)
data.knn20.thres <- threshold_test(data.knn20)

data.knn.thres %>% slice_max(Accuracy)
```

```
#>    k prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1 20            0.4 0.9969324 0.9499413   0.9987586   0.9416427  0.998074
#>      falseNeg   falsePos
#> 1 0.001241446 0.05835731
```

```r
data.knn20.thres %>% slice_max(Accuracy)
```

```
#>    k prob_threshold  Accuracy     Kappa Sensitivity Specificity Precision
#> 1 20            0.4 0.9969324 0.9499413   0.9987586   0.9416427  0.998074
#>      falseNeg   falsePos
#> 1 0.001241446 0.05835731
```

The highest accuracy threshold for the k=20 model is 0.5. This threshold produced a higher accuracy for the model than before, as well as a very high precision and good FNR and FPR combination. The metrics are not as good for k=20 model when compared to k=10, but the differences are very minor. Because of the added stability, I believe the k=20 model at threshold 0.5 is the optimal model of the two.

### ROC Curve

I used the plotROC() function to both plot the ROC curve and calculate the AUROC value of the model.


```r
knn.selected <- data.knn20.thres[1:9] %>% slice_max(Accuracy)
knn.selected <- plotROC(data.knn20, knn.selected, "KNN")
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-23-1.png" width="75%" style="display: block; margin: auto;" />

The ROC curve for the selected KNN model looks very good. The curve is better than the logistic regression model's ROC curve, and reaches a TPR of 1 very early in the plot. The AUROC for the KNN model was 0.9994968.

## Penalized Logistic Regression (ElasticNet)

### Model Building

Going off of advice from Professor Gedeck, I decided to start with a ridge regression model and work forward from there. To create the ridge model, and other elasticNet models after that, I used the train() function again, with the method set to "glmnet." A tune grid was also added to the train() function containing a sequence of lambda values from 0 to 1 every 0.1.


```r
set.seed(useed)

lambdaGrid <- expand.grid(alpha = 0, lambda = seq(0,1, 0.1))

data.ridge <- train(ClassTarp~Red+Green+Blue, data=data,
                  method="glmnet",
                  tuneGrid=lambdaGrid,
                  trControl=control)

data.ridge.i <- train(ClassTarp~Red+Green+Blue+Green:Blue, data=data,
                  method="glmnet",
                  tuneGrid=lambdaGrid,
                  trControl=control)

data.ridge$results %>% slice_max(Accuracy)
```

```
#>   alpha lambda  Accuracy     Kappa  AccuracySD    KappaSD
#> 1     0      0 0.9778941 0.4615704 0.001633888 0.06059846
```

```r
data.ridge.i$results %>% slice_max(Accuracy)
```

```
#>   alpha lambda Accuracy     Kappa  AccuracySD    KappaSD
#> 1     0      0 0.978495 0.4835771 0.001493932 0.05411579
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-25-1.png" width="75%" style="display: block; margin: auto;" />

The optimal lambda value selected for the ridge regression was 0. This was understandable because each of the color values has proven to be very significant to predicting the response, so neither one should be reduced. Any lambda tested above 0 had the same, lower accuracy and a kappa value of 0. So if there were a lambda value better for the model than 0, it would have been between 0 and 0.1 but that was unlikely. After determining the optimal lambda of 0, I wanted to determine what value of alpha would be the best.

#### Tuning Parameters

A second model was created, this time with a constant lambda of 0, and a sequence of alpha values to run and compare. Running the train() with a sequence of alphas values from 0 to 1 every 0.05.


```r
set.seed(useed)

alphaGrid <- expand.grid(alpha = seq(0,1, 0.05), lambda=0)

data.elastic <- train(ClassTarp~Red+Green+Blue, data=data,
                      method="glmnet",
                      tuneGrid=alphaGrid,
                      trControl=control)   
       
data.elastic$results %>% slice_max(Accuracy)
```

```
#>   alpha lambda  Accuracy     Kappa  AccuracySD    KappaSD
#> 1  0.80      0 0.9952404 0.9195510 0.000634466 0.01102222
#> 2  0.85      0 0.9952404 0.9195966 0.000634466 0.01097306
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-27-1.png" width="75%" style="display: block; margin: auto;" />

With a lambda of 0, the most accurate model produced had an alpha of 0.8. This value gets the model closer to a Lasso regression, but not quite there. With an accuracy of 0.9952563, the optimal tuning parameters for the Penalized Logistic Regression were alpha=0.8 and lambda=0.

### Determining Threshold




```r
data.elastic.thres <- threshold_test(data.elastic2)
data.elastic.thres %>% slice_max(Accuracy)
```

```
#>   alpha lambda prob_threshold  Accuracy     Kappa Sensitivity Specificity
#> 1   0.8      0            0.7 0.9956041 0.9271391   0.9985952   0.9050407
#>   Precision    falseNeg   falsePos
#> 1 0.9968693 0.001404783 0.09495927
```

The most accurate threshold for classification ended up being 0.7. This level produced very high accuracy and precision, as well as low false negative and false positive rates; showing to be a very effective option.

### ROC Curve

I used the plotROC() function to both plot the ROC curve and calculate the AUROC value of the model.


```r
plr.selected <- data.elastic.thres %>% slice_max(Accuracy)
plr.selected <- plotROC(data.elastic2, plr.selected, "Penalized Logistic Regression")
```

<img src="Project-1_Schroeder_Christian_files/figure-html/unnamed-chunk-30-1.png" width="75%" style="display: block; margin: auto;" />

The ROC curve for the penalized logistic regression model was good and looked quite similar to the ROC curve of the logistic regression. It also rounds off at the corner, similar to the QDA curve. The AUROC for this model was 0.9985056.

# Results (Cross-Validation)

<table class='gmisc_table' style='border-collapse: collapse; margin-top: 1em; margin-bottom: 1em;' >
<thead>
<tr><td colspan='8' style='text-align: left;'>
* lambda for PLR (0) not shown</td></tr>
<tr><th style='border-bottom: 1px solid grey; font-weight: 900; border-top: 2px solid grey; text-align: center;'>Model</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>Tuning</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>AUROC</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>Threshold</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>Accuracy</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>TPR</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>FPR</th>
<th style='font-weight: 900; border-bottom: 1px solid grey; border-top: 2px solid grey; text-align: center;'>Precision</th>
</tr>
</thead>
<tbody>
<tr>
<td style='text-align: left;'>Log Reg</td>
<td style='text-align: center;'></td>
<td style='text-align: center;'>0.9985</td>
<td style='text-align: center;'>0.7</td>
<td style='text-align: center;'>0.9957</td>
<td style='text-align: center;'>0.9985</td>
<td style='text-align: center;'>0.089</td>
<td style='text-align: center;'>0.9971</td>
</tr>
<tr style='background-color: #f8f8f8;'>
<td style='background-color: #f8f8f8; text-align: left;'>LDA</td>
<td style='background-color: #f8f8f8; text-align: center;'></td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9952</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.7</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9944</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9987</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.134</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9956</td>
</tr>
<tr>
<td style='text-align: left;'>QDA</td>
<td style='text-align: center;'></td>
<td style='text-align: center;'>0.9971</td>
<td style='text-align: center;'>0.6</td>
<td style='text-align: center;'>0.9948</td>
<td style='text-align: center;'>0.9996</td>
<td style='text-align: center;'>0.1518</td>
<td style='text-align: center;'>0.995</td>
</tr>
<tr style='background-color: #f8f8f8;'>
<td style='background-color: #f8f8f8; text-align: left;'>KNN</td>
<td style='background-color: #f8f8f8; text-align: center;'>20</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9995</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.4</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9969</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9988</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.0584</td>
<td style='background-color: #f8f8f8; text-align: center;'>0.9981</td>
</tr>
<tr>
<td style='border-bottom: 2px solid grey; text-align: left;'>PLR</td>
<td style='border-bottom: 2px solid grey; text-align: center;'>0.8</td>
<td style='border-bottom: 2px solid grey; text-align: center;'>0.9985</td>
<td style='border-bottom: 2px solid grey; text-align: center;'>0.7</td>
<td style='border-bottom: 2px solid grey; text-align: center;'>0.9956</td>
<td style='border-bottom: 2px solid grey; text-align: center;'>0.9986</td>
</tr>
</tbody>
</table>

# Conclusions











