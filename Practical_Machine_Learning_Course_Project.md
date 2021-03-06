

# Practical Machine Learning : Course Project


## Introduction

In their 2013 paper, *"Qualitative Activity Recognition of Weight Lifting Exercises"*" [1], the authors identify that people regularly quantify how much of a particular activity they do, but they rarely quantify how well they do it.  The paper seeks to use data collected from accelerometers on the belt, forearm, arm, and dumbell of 6 participants to identify whether they were performing  barbell lifts correctly or in one of 5 different incorrect ways.  More information is available from the website <a href = http://groupware.les.inf.puc-rio.br/har>Human Activity Recognition</a> (see the section on the Weight Lifting Exercise Dataset).

The objective of this analysis is to demonstrate that it is possible to use machine learning techniques to predict the manner in which an exercise was performed based on the data collected from this study.

An analysis was performed to determine if there was a significant association between the activity being performed and the recorded parameters.  Using decision tree analysis, the analysis shows that there is a sufficient relationship to be able to predict the activity being performed based on the recorded parameters of the movement with an accuracy over 99%.

To verify this finding, the model was used to predict the exercise technique using 20 test cases which had not previously been used.  Prediction accuracy on these test cases was 100%.


## Methods

### Data Collection

This analysis is based on data provided by staff at John Hopkins Bloomberg School of Public Health at https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv and https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv.

The data consist of 19622 observations of 154 sensor parameters together with a record of the activity being performed, the time that the measurements were recorded and the name of the volunteer performing the activity.  The data were downloaded from the online repository on 24 August 2014 using the R programming language [2].


```r
# set Working directory for all files
setwd( "D:/Users/Michael/R Working Directory/Data Science Specialization/8.  Practical Machine Learning" )

if( !file.exists( "pml-training.csv" ) ) {
 
    fileUrl <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
    download.file( fileUrl, destfile = "pml-training.csv" )
    dateDownloaded <- date()
 
}

train <- read.csv( "pml-training.csv", header = TRUE )

if( !file.exists( "pml-testing.csv" ) ) {
 
    fileUrl <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
    download.file( fileUrl, destfile = "pml-testing.csv" )
    dateDownloaded <- date()
 
}

test <- read.csv( "pml-testing.csv", header = TRUE )
```


### Exploratory Analysis

Initial exploratory analysis was performed by constructing tables and examining plots of the observed data, comparing the activity being performed to other variables reported in the data.  With such a large number of variables to be examined, a number of other analysis techniques were applied, including decision tree analysis and singular value decomposition.

Exploratory analysis was used :

+ to identify missing values,
+ to identify any values which appear to deviate markedly from others in the data, known as outliers [3],
+ to determine whether any variables required transformation to facilitate analysis,
and
+ to determine the most appropriate technique to use for the prediction model.

There were a number of variables for which data only existed for one sample in each exercise window and these were excluded from the analysis.  No missing values or outliers were identified in the data, and the variable values needed no transformation, as they were all numeric in value.


```r
# remove all columns containing NA or blank
train <- train[ , !is.na( train[ 1, ] ) & train[ 1, ] != "" ]

# remove date and time columns
train <- train[ , 7 : ncol( train ) ]
```


### Statistical Modelling

To identify the relationship between activity and the recorded sensor parameters, a random forest analysis [4] was performed. The random forest approach is reported to be one of the most accurate learning algorithms currently available [5], using repeated bootstrapping to create a series of decision trees and a voting process to determine the reported classification [6].

A random forest approach was chosen for this analysis because of the reported high degree of accuracy in the classifications it produces.  The reported accuracy was borne out in the exploratory analysis where it provided the most accurate prediction model from all the different techniques assessed.  However, the output of a random forest is widely acknowledged to be difficult to interpret, and random forests have been reported to overfit for some datasets [5].

The random forest approach also inherently includes cross-validation, as described in the *Random Forests* website [6]  -  "In random forests, there is no need for cross-validation or a separate test set to get an unbiased estimate of the test set error. It is estimated internally, during the run ...".

Data was provided in the form of a training and a test data set.  The use of multiple data sets allows the development of the prediction model using the training data set and testing of the model using the test data set.  As the desired output is the correct classification of the activity being performed, the accuracy of the output was reported as an error rate, calculated as the number of incorrect classifications reported by the model expressed as a percentage of the total number of classifications performed on each of these data sets.


## Results

A random forest analysis was performed to model the relationship between the remaining 53 measured parameters and the activity being performed at the time of the measurement.


```r
# install doParallel package if not installed
if( "doParallel" %in% rownames( installed.packages() ) == FALSE ) {
 
    install.packages( "doParallel" )
 
}

library( "doParallel" )

registerDoParallel( cores = detectCores() )

# install caret package if not installed
if( "caret" %in% rownames( installed.packages() ) == FALSE ) {
 
    install.packages( "caret" )
 
}

library( caret )

modFit <- train( classe ~ ., data = train, method = "rf", trControl = trainControl( method = "cv", number = 4, allowParallel = TRUE ) )
modFit$finalModel
```

```
## 
## Call:
##  randomForest(x = x, y = y, mtry = param$mtry) 
##                Type of random forest: classification
##                      Number of trees: 500
## No. of variables tried at each split: 27
## 
##         OOB estimate of  error rate: 0.14%
## Confusion matrix:
##      A    B    C    D    E class.error
## A 5578    1    0    0    1   0.0003584
## B    4 3790    2    1    0   0.0018436
## C    0    6 3416    0    0   0.0017534
## D    0    0   10 3205    1   0.0034204
## E    0    0    0    2 3605   0.0005545
```

```r
errRate = round( modFit$finalModel$err.rate[ nrow( modFit$finalModel$err.rate ), 1 ] * 100, 2 )
```

The random forest analysis tool includes an in-built estimate of the test set error, which is calculated during the analysis and is referred to as the "out-of-bag (OOB) error estimate".  For this analysis, the OOB estimate of error rate was reported as 0.14%.  Due to the way the cross-validation is performed, this is the expected out-of-sample error rate.

It can be shown that this is different from the in-sample error rate.


```r
trainPredict <- predict( modFit, train )
table( trainPredict, train$classe )
```

```
##             
## trainPredict    A    B    C    D    E
##            A 5580    0    0    0    0
##            B    0 3797    0    0    0
##            C    0    0 3422    0    0
##            D    0    0    0 3216    0
##            E    0    0    0    0 3607
```

```r
isError <- round( sum( trainPredict != train[ , 'classe' ] ) / length( train[ , 'classe' ] ) * 100, 2 )
```

So the in-sample error is actually 0%.

Finally, the random forest model was used to predict the values of activity for each of the 20 observations in the test data set. The results of the prediction were submitted to the Coursera *Practical Machine Learning* Prediction Assignment Submission page and were reported as being 100% correct.


```r
answers <- predict( modFit, test )

pml_write_files = function( x ) {
 
    n = length( x )
 
    for(i in 1 : n ) {

        filename = paste0( "problem_id_", i, ".txt" )
        write.table( x[ i ], file = filename, quote=FALSE, row.names=FALSE, col.names=FALSE )
 
    }
 
}
 
pml_write_files( answers )
```


##  Conclusions

The analysis suggests that the information captured from the belt, forearm, arm, and dumbell sensors can be used to predict the activity being performed at the time, from a small range of possible activities, using the measured and calculated parameters describing the movement of the subject.  The analysis demonstrated the ability to predict the actual activity with an accuracy of over 99%.

 
## References

[1]  Velloso, E.; Bulling, A.; Gellersen, H.; Ugulino, W.; Fuks, H. Qualitative Activity Recognition of Weight Lifting Exercises. Proceedings of 4th International Conference in Cooperation with SIGCHI (Augmented Human '13) . Stuttgart, Germany: ACM SIGCHI, 2013.

[2]  R Core Team (2012) - "R: A language and environment for statistical computing."  
URL: http://www.R-project.org  

[3]  Wikipedia "Outlier" Page  
URL: http://en.wikipedia.org/wiki/Outlier  
Accessed 24 August 2014  

[4]  Coursera "Practical Machine Learning" course - Video Lecture "3 - 3 - Random Forests"  
URL: https://class.coursera.org/predmachlearn-004/lecture  
Accessed 19 August 2014  

[5]  Wikipedia "Random forest" Page  
URL: http://en.wikipedia.org/wiki/Random_forest  
Accessed 24 August 2014  

[6]  Leo Breiman and Adele Cutler - "Random Forests"  
URL: http://www.stat.berkeley.edu/~breiman/RandomForests/cc_home.htm  
Accessed 24 August 2014  

