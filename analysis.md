# Analysis of Storm Damage across the US
Oliver Hofkens  
22 April 2017  



## Synopsis

In this report the damage impact of storms on the population and economy is analysed 
using the U.S. National Oceanic and Atmospheric Administration's (NOAA) storm database.
This database tracks characteristics of major storms and weather events in the United States, 
including when and where they occur, as well as estimates of any fatalities, injuries, and property damage.
Answers are sought for two major questions:   

 * Across the United States, which types of events are most harmful with respect to population health?  
 * Across the United States, which types of events have the greatest economic consequences?  

## Data Processing

Load the packages used in this analysis:

```r
library(dplyr)
library(ggplot2)
```


Download the data and load it into R: 

```r
if(!dir.exists("data")){
    dir.create("data")
}

download.file("https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2", "data/StormData.csv.bz2")

# Supplying the nrows and (empty) comment.char arguments reduces memory usage and speeds up the loading.
stormData <- read.csv('data/StormData.csv.bz2', nrows = 902297, comment.char = "")
```

Calculate the actual cast of damages by multiplying the cost by the exponent (K, M, B):


```r
# Calculate actual costs (not K, M, B) of property damage and crop damage
calculateCost <- function(cost, exp){
    switch(toupper(as.character(exp)),
            K = cost * 1000,
            M = cost * 1000000,
            B = cost * 1000000000,
           cost
           )
}

calculateCost <- Vectorize(calculateCost)

stormData <- stormData %>% mutate(propertyDamage = calculateCost(PROPDMG, PROPDMGEXP), cropDamage = calculateCost(CROPDMG, CROPDMGEXP))
```

Then calculate a total value for 'population' and 'economic' damage.
For population damage, a fatality will be counted 20 times more harmful than an
injury (a weighted sum). For economic damage, the sum of crop and property damage will be taken.


```r
stormData <- stormData %>% mutate(economicDamage = propertyDamage + cropDamage, populationDamage = FATALITIES * 20 + INJURIES)
```

Calculate the mean and sum of damages by event type.

```r
summaryByType <- stormData %>% group_by(EVTYPE) %>% summarise(
    avgEconomicDmg = mean(economicDamage),
    avgPopulationDmg = mean(populationDamage),
    sumEconomicDmg = sum(economicDamage),
    sumPopulationDmg = sum(populationDamage)
    ) 
```

## Results

### Population damage analysis

The following plot show the total population damage (weighted sum of fatalities + injuries) of
the storm types with the highest total and average impact on population. The green line represents the average damage per observation 
of the storm.


```r
mostPopulationDmgTotal = summaryByType %>% arrange(desc(sumPopulationDmg)) %>% head(n=3L)
mostPopulationDmgMean = summaryByType %>% arrange(desc(avgPopulationDmg)) %>% head(n=3L)
mostPopulationDmg = rbind(mostPopulationDmgMean, mostPopulationDmgTotal) %>% distinct(EVTYPE, .keep_all = TRUE) %>% arrange(desc(sumPopulationDmg))

ggplot(data = mostPopulationDmg, aes(reorder(EVTYPE, sumPopulationDmg))) + 
    geom_col(aes(y = sumPopulationDmg)) +
    geom_errorbar(aes(ymin = avgPopulationDmg, ymax = avgPopulationDmg), color = "green") +
    scale_y_log10() +
    ggtitle('Population Storm Damage by Storm Type') +
    labs(y = "Population Damage (injuries & casualties)", x = "Storm Type") +
    theme(axis.text.x = element_text(angle = 30, hjust = 0.5, vjust = 0.5)) 
```

![](analysis_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

 * 3 of 6 results were the only observation of their type.
 * The average lightning storm seems to be pretty harmless, but the total damage is significant.
    * NOTE: One explanation is that the mortality rate for lightning strikes is higher than for most other storm types.
 * Tornadoes (sometimes in combination with other storms) seem to be the most dangerous to population health,
    having both a high average damage and the biggest total damage.

### Economic damage analysis

As with the population damage analysis, we plot the storm types with the highest total and the highest average economic impact.


```r
mostEconomicDmgTotal = summaryByType %>% arrange(desc(sumEconomicDmg)) %>% head(n=3L)
mostEconomicDmgMean = summaryByType %>% arrange(desc(avgEconomicDmg)) %>% head(n=3L)
mostEconomicDmg = rbind(mostEconomicDmgMean, mostEconomicDmgTotal) %>% distinct(EVTYPE, .keep_all = TRUE) %>% arrange(desc(sumEconomicDmg))

ggplot(data = mostEconomicDmg, aes(reorder(EVTYPE, sumEconomicDmg))) + 
    geom_col(aes(y = sumEconomicDmg / 1000000)) +
    geom_errorbar(aes(ymin = avgEconomicDmg / 1000000, ymax = avgEconomicDmg / 1000000), color = "green") +
    scale_y_log10() +
    ggtitle('Economic Storm Damage by Storm Type') +
    labs(y = "Economic Damage (Million $)", x = "Storm Type") +
    theme(axis.text.x = element_text(angle = 30, hjust = 0.5, vjust = 0.5)) 
```

![](analysis_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

 * Water damage is not to be underestimated with floods and heavy rains making a very big impact on the economy.
 * The average tornado doesn't do much damage, but the total damage of tornadoes is huge. 
    * Note: This can be explained by tornadoes in rural areas dragging the mean down, but tornadoes in villages and cities having massive consequences.
    
