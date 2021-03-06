---
title: "Basic Inferential Data Analysis"
author: "Clément Brutti-Mairesse"
date: "27/04/2020"
output:
  html_document:
    df_print: paged
---

```R
knitr::opts_chunk$set(echo = TRUE)
```

## About the dataset
We make an analysis on an experiment made on 60 pigs, trying to find the effect of `Orange Juice` and `Ascorbic Acid` on tooth growth. The metric is the length of odontoblast, evaluated in milligrams/day.

## Exploratory Data Analysis
```R
library(ggplot2)
library(dplyr)
library(knitr)
library(e1071)
```

Here is a table with the count, variance, quartiles of each group.
```R
data <- ToothGrowth
levels(data$supp) <- c("Orange Juice", "Ascorbic Acid")
groupData <- data %>% group_by(supp, dose)
kable(groupData %>% summarise(n = n(),  mean = mean(len), variance = var(len), 
                              q25 = quantile(len)[2], 
                              median = quantile(len)[3], 
                              q75 = quantile(len)[4],
                              skewness = skewness(len),
                              kurtosis = kurtosis(len)))
```
![EDA](eda.png)
We can already see that groups do not have the same variance.

```R
ggplot(data, aes(x=factor(dose), y=len)) + 
 facet_grid(.~supp) +
 geom_violin(aes(fill = supp), show.legend = FALSE) +
 labs(title="Tooth growth by dosage and by supplement", 
      x="Dose (mg/day)",
      y="Tooth growth")
```
![Violin Plot](violinplot.png)


## Growth analysis
We now compare the six different groups with themselves. We suppose this is a randomized experiment, and that the groups are independents and that the variance is **not** constant between groups. We use a 95% T confidence interval to compare the groups. Here is a reminder of the detail of the t confidence interval calculation with an unequal variance, (we use `t.test(, var.equal=FALSE)`)

![ttest](t-test.png)

with t_df : t quantile and df equals to
![df](df.png)

Here is a table summing up the confidence intervals. Rows are compared with columns. For example: *Orange Juice 2mg/day* (OJ 2) is `15.8 to 20.36` mg/day better than *Ascorbic Acid 0.5mg/day* (AA 0.5).

```R
combinations <- list(c("Orange Juice", 0.5), 
                     c("Orange Juice", 1), 
                     c("Orange Juice", 2),
                     c("Ascorbic Acid", 0.5), 
                     c("Ascorbic Acid", 1), 
                     c("Ascorbic Acid", 2))

testStat <- function(combination) {
  baseSubset <- subset(data, supp == combination[1] & dose == combination[2])
  lapply(combinations, function(comb){
    comparisonSubset <- subset(data, supp == comb[1] & dose == comb[2])
    
    tTestResult <- t.test(baseSubset$len - comparisonSubset$len, 
                          var.equal = FALSE, conf.level = 0.95)
    
    min <- tTestResult$conf.int[1]
    max <- tTestResult$conf.int[2]
    
    if (is.nan(min)){
      return("NA")
    }
    
    formatConfidence <- paste("[", format(min, digits = 4), ";", format(max, digits=4), "]", sep = "")
    
    return(formatConfidence)
  })
}
confidences <- lapply(combinations, testStat)
confidenceMatrix <- data.frame(matrix(unlist(confidences), nrow=6, byrow=6))
colName <- c("OJ 0.5", "OJ 1", "OJ 2", "AA 0.5", "AA 1","AA 2")
rownames(confidenceMatrix) <- colName
colnames(confidenceMatrix) <- colName

kable(confidenceMatrix)
```
![confidence intervals](confidenceintervals.png)

Similarly, this a table sum up the p-Values in percentage
```R
testStat <- function(combination) {
  baseSubset <- subset(data, supp == combination[1] & dose == combination[2])
  lapply(combinations, function(comb){
    comparisonSubset <- subset(data, supp == comb[1] & dose == comb[2])
    
    tTestResult <- t.test(baseSubset$len - comparisonSubset$len, 
                          var.equal = FALSE, conf.level = 0.95)
    
    pValue <- tTestResult$p.value
    
    if (is.nan(pValue)){
      return("NA")
    }
    
    return(paste(formatC(pValue * 100, digit = 2), "%", sep = ""))
  })
}
confidences <- lapply(combinations, testStat)
confidenceMatrix <- data.frame(matrix(unlist(confidences), nrow=6, byrow=6))
colName <- c("OJ 0.5", "OJ 1", "OJ 2", "AA 0.5", "AA 1","AA 2")
rownames(confidenceMatrix) <- colName
colnames(confidenceMatrix) <- colName

kable(confidenceMatrix)
```
![pvalues](pvalues.png)

## Conclusions
We can see that Ascorbic Acid had relevant improvement as the dose increase. In opposition as the dose of Orange Juice increase the growth does not increase significantly. Comparing the two supplements with equal dose does not show a relevant difference of growth either. For example the confidence interval comparing *Orange Juice 2mg/day* (OJ 2) with *Ascorbic Acid 0.5mg/day* (AA 2) is *-4.329:4.169* it contains zero, therefore the difference is not significant with a 95% confidence interval.
