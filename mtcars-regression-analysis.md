---
output: pdf_document
---
Factor analysis of car miles per gallon (MPG)
==========================


This document is my Course Project submission for the class "Regression Models" on Coursera.

# Summary

This study is an analysis of the very famous [mtcars dataset](https://stat.ethz.ch/R-manual/R-devel/library/datasets/html/mtcars.html), available in R. It aims at explaining the possible relationships between a car's miles per gallon measure, and the different other characteristics of the car. In particular, we will focus our attention on the transmission in order to give an answer to the two following questions:

* Is an automatic or manual transmission better for MPG?
* How can we quantify the MPG difference between automatic and manual transmissions?

First of all, we will explore the data to get a better overview of the underlying patterns in the dataset, and then we will build several linear models to explain and quantify the relationships between transmission and MPG.

# Analysis

## Data exploration

### Loading data


```r
data(mtcars)
```

Our main focus is the relationship between transmission "am" variable (binary: 0 = automatic, 1 = manual) and the number of miles per US gallon "mpg" variable. For ease of use, let's build a "transmission" factor variable out of the "am"" variable:


```r
mtcars$transmission <- factor(mtcars$am, labels = c("automatic", "manual"))
mtcars <- subset(mtcars, select = -c(9))
```

### Data visualisation

Visualising "transmission" and "MPG" variables can help us understand the main patterns in the data. The figure 1 shows that manual cars tend to have higher MPG values. This is confirmed by Figure 2, as it appears clearly that the distribution of MPG for manual cars is higher than for automatic cars.


```r
library(ggplot2)
g1 <- ggplot(mtcars, aes(x=mpg, fill = transmission)) + 
  geom_histogram(binwidth = 1, col = "black") +
  ggtitle('Figure 1: Histogram of Miles per US Gallon values')
g2 <- ggplot(mtcars, aes(x=transmission, y=mpg, fill = transmission)) +
  geom_boxplot(adjust = 1) + geom_jitter(size = 2) +
  ggtitle('Figure 2: Boxplot of MPG values for each transmission')
```

Thus, this gives an indication about the answer of the first question: manual cars seem to be better for MPG.

However, this MPG difference is not necessarily explained by the transmission only. The transmission might indeed be related to other variables in the dataset that may explain that difference. We can visualise the relationships between pairs of variables with a pair plot. In the figure 3 we notice that the transmission is highly related to "disp" (Displacement), "drat" (Rear axle ratio) and "cyl" (number of cylinders), and those variables are all highly correlated to MPG. We have to be careful when quantifying the effect of transmission on MPG and make sure we also take into account the adjustments from the other variables.


```r
library(stats);library(graphics);library(GGally)
g3 <- ggpairs(subset(mtcars, select = c(1,2,3,5,9,11)),
              colour = 'transmission', alpha = 0.6, 
              title = "Figure 3: Pair relationships in the mtcars dataset. Blue: manual, Red: automatic")
```

Thus, we will first model with only "transmission" as a predictor, and then we will add other variables in a more complex regression model to isolate and esimate the effect of transmission on MPG.

## Simple model

In this section, we will simply give an unadjusted estimate of the effect of transmission on MPG. We just need to fit a linear model and compute the confidence interval.


```r
unadj.model <-lm(mpg ~ transmission, data=mtcars)
coeff <- summary(unadj.model)$coefficients
unadj.interv <- coeff[2,1] + c(-1,1) * qt(0.975, df = unadj.model$df) * coeff[2,2]
coeff
```

```
##                    Estimate Std. Error t value  Pr(>|t|)
## (Intercept)          17.147      1.125  15.247 1.134e-15
## transmissionmanual    7.245      1.764   4.106 2.850e-04
```

The above code chunk fits a linear model (with slope) of MPG vs transmission, and shows that a manual transmission is marginally related to a greater MPG (p-value: 2.8502 &times; 10<sup>-4</sup> < 0.95). When considering the unadjusted model, compared to an automatic transmission, a manual tranmission increases MPG by 7.2449 on average, and the 95% confidence interval for this increase is [3.6415, 10.8484].

When plotting the residuals of the unadjusted model, we observe a larger variability of MPG residuals for manual transmission than for automatic transmission. This heteroskedaticity may indicate that some model terms are missing. Thus, we should try to include other variables into our model.


```r
g4 <- ggplot(mtcars, aes(x = transmission, y = resid(unadj.model), fill = transmission)) +
  geom_boxplot(adjust = 1) + geom_jitter(size = 2) +
  ylab("Residuals") +
  ggtitle("Figure 4: Residuals of the unadjusted linear model of MPG")
```

## Multivariate model

Our strategy for selecting the best model will rely on variance inflation. We will discard all the variables too redondant with the other variables by computing variance inflation for each of the variables in the dataset.


```r
library(car)
fit <- lm(mpg ~ ., data=mtcars)
vif(fit)
```

```
##          cyl         disp           hp         drat           wt 
##       15.374       21.620        9.832        3.375       15.165 
##         qsec           vs         gear         carb transmission 
##        7.528        4.966        5.357        7.909        4.648
```

We observe that "cyl", "disp" and "wt" variables have a high variance inflation coefficient. We decide to discard them to make our model more parcimonious. Now, we try to add sequentially each of the remaining variables and compute the increase of the model variance.


```r
a <- summary(unadj.model)$cov.unscaled[2,2]
vars = list()
hp.model <- update(unadj.model, mpg ~ transmission + hp)
vars <- append(vars, summary(hp.model)$cov.unscaled[2,2] / a)
drat.model <- update(unadj.model, mpg ~ transmission + drat)
vars <- append(vars, summary(drat.model)$cov.unscaled[2,2] / a)
qsec.model <- update(unadj.model, mpg ~ transmission + qsec)
vars <- append(vars, summary(qsec.model)$cov.unscaled[2,2] / a)
vs.model <- update(unadj.model, mpg ~ transmission + vs)
vars <- append(vars, summary(vs.model)$cov.unscaled[2,2] / a)
gear.model <- update(unadj.model, mpg ~ transmission + gear)
vars <- append(vars, summary(gear.model)$cov.unscaled[2,2] / a)
carb.model <- update(unadj.model, mpg ~ transmission + carb)
vars <- append(vars, summary(carb.model)$cov.unscaled[2,2] / a)
str(vars)
```

```
## List of 6
##  $ : num 1.06
##  $ : num 2.03
##  $ : num 1.06
##  $ : num 1.03
##  $ : num 2.71
##  $ : num 1
```

The above result shows us that we can safely include the "carb" variable into our model. Let us repeat this operation with another variable. The variance inflation from "gear" is high, so we decide to discard it.



```r
summary(cyl2.model)$cov.unscaled[2,2] / b
```

```
## [1] 1.738
```

```r
hp2.model <- update(carb.model, mpg ~ transmission + hp + carb)
summary(hp2.model)$cov.unscaled[2,2] / b
```

```
## [1] 1.231
```

```r
drat2.model <- update(carb.model, mpg ~ transmission + drat + carb)
summary(drat2.model)$cov.unscaled[2,2] / b
```

```
## [1] 2.09
```

```r
qsec2.model <- update(carb.model, mpg ~ transmission + qsec + carb)
summary(qsec2.model)$cov.unscaled[2,2] / b
```

```
## [1] 1.07
```

```r
vs2.model <- update(carb.model, mpg ~ transmission + vs + carb)
summary(vs2.model)$cov.unscaled[2,2] / b
```

```
## [1] 1.064
```

Following those results, we decide to include "qsec" and "vs" in the model. We discard the "drat" variable. Our final model then contains "qsec", "transmission" and "carb" variables.


```r
adj.coeff <- summary(qsec2.model)$coefficients
adj.interv <- adj.coeff[2,1] + c(-1,1) * qt(0.975, df = qsec2.model$df) * adj.coeff[2,2]
```

The R^2 for this model is 0.7638, which is satisfying. We plotted the residual plots in Figure 5, and there is not any strong unusual pattern. The adjusted estimated increase of MPG explained by a manual tranmission is 8.4353. The adjusted 95% confidence interval is [6.0812, 10.7895].


# Appendix
<img src="figure/exploration1.png" title="plot of chunk exploration" alt="plot of chunk exploration" style="display: block; margin: auto;" /><img src="figure/exploration2.png" title="plot of chunk exploration" alt="plot of chunk exploration" style="display: block; margin: auto;" />

<img src="figure/pairplot.png" title="plot of chunk pairplot" alt="plot of chunk pairplot" style="display: block; margin: auto;" />

<img src="figure/residuals1.png" title="plot of chunk residuals" alt="plot of chunk residuals" style="display: block; margin: auto;" /><img src="figure/residuals2.png" title="plot of chunk residuals" alt="plot of chunk residuals" style="display: block; margin: auto;" />
