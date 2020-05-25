# tdaImage

## Overview

Novel method for image classification using [Topological Data Analysis
(TDA)](Slides.pdf). Using TDA in the image contenxt, one is able to
perform dimension reduction on a dataset to improve runtime of the
analysis as well as to avoid the risk of overfitting. 

## Intuition

Inspired from Topological Data Analysis, TDAsweep defines components in a more simplified way. Specifically, TDAsweep casts thresholding on the original image (each pixel value above the threshold will be denoted as 1 and 0 otherwise). Then, TDAsweep counts contiguous components in horizontal, vertical, and the two diagonal directions of a pixel matrix. The counts of the components in each direction will serve as the new set of features describing the original image.

An example should help illustrate the process more clearly:

Say, after thresholding some toy image, we have the following matrix:

                                10011101
                                10111100
                                10101101

Then, we would count the number of components in each rows, columns, and diagonals.
An example of counting the components for each rows would be:

                There are 3 components in vector “10011101”
                There are 2 components in vector “10111100”
                There are 4 components in vector “10101101”

Here, [3,2,4] will be included as the new set of features. We repeat this process for columns and the two diagonal directions (NW to SE and NE to SW).

The typical pattern involved is:

1.  Perform dimension reduction, by some means -- PCA, the 'C' in "CNN,"
our **TDAsweep** presented here, etc.

2.  Feed the results of 1) above into one's favorite machine learning
    method, such as NNs, SVM, logit or random forests.

A fast, non-iterative method for dimension reduction of images would be
quite useful.

## TDAsweep

The [**regtools** package](https://github.com/matloff/regtools) is required. 

**Usage:**

tda_wrapper_func(images, labels, nr, nc, rgb=TRUE, thresholds=0 , intervalWidth=1, cls, prep=FALSE, rcOnly=FALSE)

* `image`: a pixel intensity matrix of images. 
* `labels`: a vector of labels each row corresponding to each image. 
* `nr`: number of rows of the input image pixels.
* `nc`: number of columns of the input image pixels.
* `rgb`: TRUE if rgb image is used. FALSE otherwise.
* `thresholds`: the minimum pixel intensity to be included in each sweep. 
* `intervalWidth`: should be set to an integer greater than 1 to achieve dimension reduction. Represent this many rows by taking the mean of them.
* `cls`: self-defined clusters for parallelization. Uses half of the total available cores of the local machine by default.
* `prep`: TRUE if prepImgSet has already be run on the image dataset to avoid redundancy in code. FALSE otherwise.
* `rcOnly`: 


The return value of **tda_wrapper_func()** is a list of number of components in row, column, and diagonals.
*Example: TDAsweep + Support Vector Machine* 

This example uses the [MNIST dataset](http://heather.cs.ucdavis.edu/mnist.csv) and perform dimension reduction with TDA, then predict the results using the  [**caret  package's SVM function**](https://cran.r-project.org/web/packages/caret/index.html). 

```R
library(tdaImage)
library(doMC)
library(caret)

#---- data preparation ----#
mnist <- read.csv("~/Downloads/mnist.csv")
sample_n <- sample(nrow(mnist))
mnist <- mnist[sample_n, ]
mnist$y <- as.factor(mnist$y)
trainIndex = createDataPartition(mnist$y, p=0.7, list=FALSE)
train_set <- mnist[trainIndex, -785]  # exclude label if doing tda
train_y_true <- mnist[trainIndex, 785]
test_set <- mnist[-trainIndex, -785]
test_y_true <- mnist[-trainIndex, 785]

#---- parameters for performing TDAsweep ----#
nr = 28  # mnist is 28x28
nc = 28
rgb = FALSE  # mnist is grey scaled
thresholds = c(50)  # set one threshold, 50
intervalWidth = 1  # set intervalWidth to 1

#---- performing tda on train set ----#
tda_train_set <- tda_wrapper_func(image=train_set, labels=train_y_true, 
                                        nr=nr, nc=nc, rgb=rgb, thresh=thresholds,
                                        intervalWidth=intervalWidth)
dim(tda_train_set)  # 784 -> 166 features after TDAsweep
tda_train_set <- as.data.frame(tda_train_set)
tda_train_set$labels <- as.factor(tda_train_set$labels)

#---- performing tda on test set ----#
tda_test_set <- tda_wrapper_func(image=test_set, labels=test_y_true,
                                        nr=nr, nc=nc, rgb=rgb, thresh=thresholds,
                                        intervalWidth=intervalWidth)
dim(tda_test_set)
tda_test_set <- as.data.frame(tda_test_set)
tda_test_label <- tda_test_set$labels
tda_test <- tda_test_set[, -167]  # take out labels for testing the svm model later

#---- training and predicting using caret's svm model ----#
registerDoMC(cores=3)
tc <- trainControl(method = "cv", number = 4, verboseIter = F, allowParallel = T)
svm_model <- train(labels ~., data=tda_train_set, method = "svmRadial", trControl = tc)
predict <- predict(svm_model, newdata=tda_test)

#---- Evaluation ----#
confusionMatrix(as.factor(predict), as.factor(tda_test_label))
```


*Example: Polynomial Regression* 

This example uses the [MNIST dataset](http://heather.cs.ucdavis.edu/mnist.csv) and perform dimension reduction with TDA, then predict the results using [**polyreg**](http://github.com/matloff/polyreg). 

```R
# initialization
mnist <- read.csv("../mnist.csv")   # get dataset
img <- mnist[, -785]
label <- mnist[, 785]
nr <- 28                            # height of one image
nc <- 28                            # width of one image
rgb <- FALSE
thresh <- 20                        # ignore all pixels with intensity lower than this 
intervalWidth <- 4

... # shuffle and take a small chunk of images

tdaout <- tda_wrapper_func(img, label, nr=nr, nc=nc, rgb=FALSE, thresh=thresh, 
                            intervalWidth=intervalWidth)
# look at first output
head(tdaout, 1)   
# [1,] 0 0.5 2 2.25 2.00 1 0.5 0 0.25 1.25 2.75 1.50 0.25 0 3 1.75 1 0.00 0 0 0
# [1,] 2.5 2.25 1.25 0.25 0 0 0 0 0 0 0 0.25 1.50 1.75 1.25 1.00 1.25 0 0 0 0
#     labels
# [1,]      3
tdaout$labels <- as.character(tdaout$labels)
pfout <- polyFit(res[-c(1:5),],2)   # fit quadratic model
newx <- tdaout[c(1:5),]             # test on the 5 rows we omitted before
newx <- newx[,-43] 
predict(pfout, newx)

```

## Parallelization
Parallelization is supported by TDAsweep. Users can input self-created clusters to TDAsweep. If the parameter was not specified, the code will use the default option of creating clusters with half of the available cores of the local machine.

**Example of creating cluster:**
library(partools)
cls<- makeCluster(2)  # creates two clusters
TDAsweep(…, cls=cls,…)

## Analysis of TDAsweep on the MNIST dataset

The results of running TDAsweep on the MNIST dataset before classification was very encouraging. We were able to achieve ~78.8% feature reduction in exchange for less than 1% accuracy loss. As a result, the runtime of training the Support Vector Machine was drastically decreased.

![alt text](https://github.com/matloff/tdaImage/tree/tdapar/table.png)

    (Table 1. Speed Comparison of SVM before and after TDAsweep)




