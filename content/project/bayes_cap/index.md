---
title: "Neighborhood Deserts: Transportation Access & Housing Disparities in NYC"
author: "Freddy Barragan, Juthi Dewan, Sam Ding, Vichy Meas"

summary: "A capstone project studying the relationship between transportation access and housing inequities in NYC using Bayesian models. "

tags: 
- Data Science
categories: []

image:
  caption: ""
  focal_point: ""
  preview_only: true

links:
- name: ""
  url: https://twitter.com/freddybarragan_
  icon_pack: fab
  icon: twitter

date: 2021-12-06T14:47:00-05:00
---

# Motivation

In collaboration with Juthi Dewan, Sam Ding, and Vichy Meas, we designed this project for our Bayesian Statistics course taught by [Dr. Alicia Johnson](https://ajohns24.github.io/portfolio/). We'd like to extend our thanks to Alicia for guiding us through Bayes and the capstone experience!

We were initially interested in characterizing New York City’s internal racial dynamics using demography, geographic mobility, community health, and economic outcomes. As this project developed, we found ourselves thinking about the relationships between transportation (in)access and housing inequity. In our project, there are two major sections: **Subway Accessibility** and **Transportation and Structural Inequity**. First, however, let's do a data introduction:


# Data Introduction

All data used in this project are from two major sources: the Tidycensus package and NYC Open Data. 

Tidycensus is an R package interface, developed by Kyle Walker and Matt Herman, that enables easy access to the US Census Bureau’s data APIs and returns Tidyverse-ready data frames from various major US Census Bureau datasets. Our demographic and socioeconomic data are drawn from the 2019 American Community Survey results found in Tidycensus package. A summary of our ACS data variables is below:

- `borough:` NYC Borough
- `total_pop`: Total Population Count by Census Tract
- `mean_income`: Mean Income by Census Tract
- `below_poverty_line_count`: Number of People Living Below the 100% Poverty Line by Census Tract
- `mean_rent`:  Mean Rent by Census
-  `unemployment_count`: Number of People on Unemployment by Census Tract
-  `latinx_count`: Number of Latinx People by Census Tract
- `white_count`: Number of White People by Census Tract
- `black_count`: Number of Black People by Census Tract
- `native_count`: Number of Native People by Census Tract
- `asian_count`: Number of Asian People by Census Tract
- `naturalized_citizen_count`: Number of Naturalized Citizens by Census Tract
- `noncitizen_count`: Number of Foreign Born People by Census Tract
- `uninsured_count`: Number of Uninsured Citizens of any Age by Census Tract
 - `gini_neighborhood` : Income inequality measured by the Gini Index per Census Tract

Note that these predictors are all measured at the census level. To aggregate these estimates at the neighborhood level, we performed two transformations:

- Compute total sums by neighborhood for every count-based measurement.
- Compute mean average estimates by neighborhood for remaining non-count predictors.

Then, for count based demographic predictors, we divide by the total population in each neighborhood to define scaled demographic metrics. They are as follows:

- `asian_perc`: Percentage of Asian People by Neighborhood
- `white_perc`: Percentage of White People by Neighborhood
- `black_perc`: Percentage of Black People by Neighborhood
- `latinx_perc`: Percentage of Latinx People by Neighborhood
- `native_perc`: Percentage of Native People by Neighborhood
- `noncitizen_perc`: Percentage of Foreign Born People by Neighborhood 
- `uninsured_perc`: Percentage of Uninsured Citizens of any Age by Neighborhood 
- `unemployment_perc`: Percentage of People on Unemployment by Neighborhood
- `below_poverty_line_perc`: Percentage of people living below the 100% poverty line.

We used NYC's Open Data portal to collect information on the remaining predictors. In particular, we used geotagged locations of Subway Stops, Bus Stops, Grocery Stores, Schools, and Eviction Sites from the Departments of Transportation, Health, Education, and Housing, respectively, to calculate the following variables: 

- `school_count`: Public school counts by Neighborhood 
- `eviction_count`: Eviction counts by Neighborhood 
- `store_count`: Grocery store and food vendor counts by Neighborhood 
- `sub_count`: Subway station counts by Neighborhood 
- `bus_count`: Bus station counts by Neighborhood
- `perc_covered_by_transit`: Percent of Neighborhood Within Walking Distance (.5 miles) of Any Subway Stop. 
- `transportation_desert_4cat`: Subway Accessibility by Neighborhood (Poor, Limited, Satisfactory, Excellent)

The process involved grouping geotagged locations by the defined neighborhood boundary regions in R’s SF package and ArcGIS We will detail the process of identifying subway deserts in the "Subway Deserts" section, however, for the remaining predictors, the process is largely the same.


# Data Summaries 

Our data has 224 observations of 38 variables. The following table presents a numeric summary and breakdown for a subset of our variables of interests. However, note that we present most demographic count variables using their equivalent percent.

```{r eval = TRUE}
#summary(modeling_data)
library(table1)
table_print <- table1(~ 
                        total_pop + mean_income + mean_rent +
                        gini_neighborhood + 
                        below_poverty_line_perc + 
                        eviction_count + 
                        transportation_desert_4cat + 
                        noncitizen_perc + white_perc + black_perc +
                        latinx_perc + asian_perc
                        | borough, data = nyc_compiled %>% as_tibble())  %>% as_tibble() 

colnames(table_print) <- c("Variable", "Bronx (N=44)", "Brooklyn (N=64)","Manhattan (N=39)", "Queens (N=77)", "Overall (N=224)")

library(kableExtra)
table_print%>%
  filter(Variable!="") %>% kable() %>% kable_styling() 
```

| Variable                     | Bronx (N=44)            | Brooklyn (N=64)         | Manhattan (N=39)          | Queens (N=77)           | Overall (N=224)         |
| ---------------------------- | ----------------------- | ----------------------- | ------------------------- | ----------------------- | ----------------------- |
| total\_pop                   |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 30200 (18700)           | 37800 (24600)           | 38300 (24600)             | 25900 (20900)           | 32300 (22800)           |
|   Median \[Min, Max\]        | 29800 \[0, 69200\]      | 38300 \[0, 97800\]      | 35700 \[0, 95300\]        | 25000 \[0, 87700\]      | 31600 \[0, 97800\]      |
| mean\_income                 |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 43400 (17900)           | 69600 (28700)           | 103000 (49700)            | 74500 (14400)           | 71900 (33900)           |
|   Median \[Min, Max\]        | 38000 \[23100, 94200\]  | 61200 \[27400, 148000\] | 108000 \[33300, 212000\]  | 72600 \[37500, 104000\] | 67200 \[23100, 212000\] |
|   Missing                    | 8 (18.2%)               | 9 (14.1%)               | 6 (15.4%)                 | 19 (24.7%)              | 42 (18.8%)              |
| mean\_rent                   |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 1230 (197)              | 1580 (465)              | 1960 (674)                | 1650 (198)              | 1600 (465)              |
|   Median \[Min, Max\]        | 1240 \[833, 1620\]      | 1450 \[792, 3280\]      | 2070 \[884, 3270\]        | 1630 \[1140, 2250\]     | 1510 \[792, 3280\]      |
|   Missing                    | 8 (18.2%)               | 9 (14.1%)               | 6 (15.4%)                 | 19 (24.7%)              | 42 (18.8%)              |
| gini\_neighborhood           |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.467 (0.0317)          | 0.462 (0.0292)          | 0.516 (0.0379)            | 0.423 (0.0260)          | 0.459 (0.0439)          |
|   Median \[Min, Max\]        | 0.467 \[0.407, 0.519\]  | 0.466 \[0.382, 0.515\]  | 0.525 \[0.436, 0.582\]    | 0.420 \[0.358, 0.484\]  | 0.457 \[0.358, 0.582\]  |
| below\_poverty\_line\_perc   |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.251 (0.122)           | 0.197 (0.143)           | 0.168 (0.129)             | 0.119 (0.0832)          | 0.179 (0.128)           |
|   Median \[Min, Max\]        | 0.251 \[0, 0.444\]      | 0.179 \[0, 1.00\]       | 0.126 \[0, 0.576\]        | 0.103 \[0, 0.615\]      | 0.148 \[0, 1.00\]       |
|   Missing                    | 3 (6.8%)                | 6 (9.4%)                | 3 (7.7%)                  | 15 (19.5%)              | 27 (12.1%)              |
| eviction\_count              |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 438 (340)               | 245 (234)               | 223 (233)                 | 126 (131)               | 238 (255)               |
|   Median \[Min, Max\]        | 406 \[1.00, 1130\]      | 163 \[1.00, 829\]       | 152 \[1.00, 1120\]        | 93.0 \[1.00, 521\]      | 148 \[1.00, 1130\]      |
| transportation\_desert\_4cat |                         |                         |                           |                         |                         |
|   Poor                       | 4 (9.1%)                | 8 (12.5%)               | 2 (5.1%)                  | 40 (51.9%)              | 54 (24.1%)              |
|   Limited                    | 12 (27.3%)              | 14 (21.9%)              | 0 (0%)                    | 18 (23.4%)              | 44 (19.6%)              |
|   Satisfactory               | 8 (18.2%)               | 12 (18.8%)              | 7 (17.9%)                 | 9 (11.7%)               | 36 (16.1%)              |
|   Excellent                  | 20 (45.5%)              | 30 (46.9%)              | 30 (76.9%)                | 10 (13.0%)              | 90 (40.2%)              |
| noncitizen\_perc             |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.174 (0.0936)          | 0.150 (0.132)           | 0.139 (0.0506)            | 0.174 (0.101)           | 0.160 (0.103)           |
|   Median \[Min, Max\]        | 0.169 \[0, 0.581\]      | 0.127 \[0, 1.00\]       | 0.133 \[0, 0.242\]        | 0.143 \[0, 0.471\]      | 0.143 \[0, 1.00\]       |
|   Missing                    | 3 (6.8%)                | 6 (9.4%)                | 3 (7.7%)                  | 15 (19.5%)              | 27 (12.1%)              |
| white\_perc                  |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.114 (0.163)           | 0.381 (0.277)           | 0.454 (0.268)             | 0.287 (0.247)           | 0.309 (0.270)           |
|   Median \[Min, Max\]        | 0.0313 \[0, 0.606\]     | 0.385 \[0, 1.00\]       | 0.496 \[0, 0.829\]        | 0.247 \[0, 1.00\]       | 0.222 \[0, 1.00\]       |
|   Missing                    | 3 (6.8%)                | 6 (9.4%)                | 3 (7.7%)                  | 15 (19.5%)              | 27 (12.1%)              |
| black\_perc                  |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.301 (0.185)           | 0.292 (0.308)           | 0.146 (0.179)             | 0.178 (0.273)           | 0.231 (0.261)           |
|   Median \[Min, Max\]        | 0.274 \[0.0631, 0.833\] | 0.152 \[0, 1.00\]       | 0.0585 \[0.00873, 0.591\] | 0.0406 \[0, 0.899\]     | 0.121 \[0, 1.00\]       |
|   Missing                    | 3 (6.8%)                | 6 (9.4%)                | 3 (7.7%)                  | 15 (19.5%)              | 27 (12.1%)              |
| latinx\_perc                 |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.531 (0.197)           | 0.190 (0.163)           | 0.246 (0.206)             | 0.255 (0.178)           | 0.292 (0.221)           |
|   Median \[Min, Max\]        | 0.618 \[0, 0.784\]      | 0.143 \[0, 0.808\]      | 0.170 \[0.0608, 0.724\]   | 0.217 \[0, 0.856\]      | 0.203 \[0, 0.856\]      |
|   Missing                    | 3 (6.8%)                | 6 (9.4%)                | 3 (7.7%)                  | 15 (19.5%)              | 27 (12.1%)              |
| asian\_perc                  |                         |                         |                           |                         |                         |
|   Mean (SD)                  | 0.0370 (0.0496)         | 0.108 (0.121)           | 0.127 (0.116)             | 0.241 (0.190)           | 0.138 (0.156)           |
|   Median \[Min, Max\]        | 0.0179 \[0, 0.217\]     | 0.0733 \[0, 0.472\]     | 0.113 \[0, 0.661\]        | 0.205 \[0, 0.757\]      | 0.0777 \[0, 0.757\]     |
|   Missing                    | 3 (6.8%)                | 6 (9.4%)                | 3 (7.7%)                  | 15 (19.5%)              | 27 (12.1%)              |

From the numerical summary observe that Manhattan has the largest population counts, highest mean rental prices, highest mean income, highest income inequality (e.g. gini value), most number of neighborhoods with excellent subway access, and the greatest proportion of white citizens. 

Conversely, observe that the Bronx has the lowest mean income and highest eviction counts, while having the highest proportions of people living below the poverty line, and the highest number of people with limited subway access. Importantly, the Bronx also has the highest densities of both Latinx and Black residents of any other borough in New York, meaning that the Black and Latinx residents in New York are experiencing the burden of New York's structural inequities.

We will touch on the connections between demographics, transportation access, and housing more explicitly in the following sections.

# Subway Accessibility

New York City is the most populous city in the US with [more than 8.8 million people](https://en.wikipedia.org/wiki/New_York_City). To support the daily commutes of its residents, NYC also built the New York City Subway, the oldest, longest, and currently busiest subway system in the US, averaging [approximately 5.6 million daily rides on weekdays and a combined 5.7 million rides each weekend](https://en.wikipedia.org/wiki/New_York_City_Subway). Compared to other US cities where automobiles are the most popular mode of transportation (ahem, Minneapolis), NYC only has about 32% of the population who choose to commute by cars, thanks to its efficient and far-reaching transit system, whereas most metro areas in the US have [more than 70% of the population](https://en.wikipedia.org/wiki/Modal_share) who chose to commute by cars.

Despite having the most extensive transit network in the entire US, NYC is still lacking in terms of transit accessibility for some neighborhoods. The general consensus in academia is that residents who walk more than 0.5 miles to get to reliable transit are considered lacking transportation access, or residing in a transportation desert. For our research, we adopted this concept to study these gaps in transportation access. Specifically, we attempt to identify and study "Subway Deserts". 

## Subway Desert Definition 

Extending the USDA's definition of a food desert, we define subway deserts as the percentage of a neighborhood— or any arbitrary geographic area— that is within walking distance of any subway stop. Citing the U.S. Federal Highway Administration, we defined walking distance as [a 0.5 mile radius](https://safety.fhwa.dot.gov/ped_bike/ped_transit/ped_transguide/ch4.cfm) and computed these regions in ArcGIS. We chose subway stations because of the subway's reliable frequency, high connectivity between boroughs, and high ridership per vehicle. Our argument against including the number of bus stops in our calculations of transportation access is that the quantity of bus stops does not accurately imply public transport accessibility due to the variability in bus efficiency, punctuality, and use. A major limitation of our work was the omission of Staten Island because it is not connected to any other borough by subway. Rather, Staten Island users typically drive or train into the city. Further, we felt that the inclusion of Staten Island would mischaracterize the relationship between lacking access and not needing access since Staten Island is an overwhelmingly white, wealthy, borough that has high levels of [car ownership](https://edc.nyc/article/new-yorkers-and-their-cars).

We first geocoded subway stop locations in NYC from the NYC Department of Transportation. Then, using ArcGIS we created a 0.5-mile-radius buffer for each station and calculated what percent of each neighborhood was covered by a buffer region. We display an example below.

<center>
<img src="/media/bayes/plot_1.png" width="50%" height="50%" />
</center>


In the graph, buffer zones are in light pink with overlapping boundaries dissolved between stations, while the dark pink dots indicate the exact geographic locations of the stations. Each neighborhood, then, had a percentage score that defined it's subway accessibility score. 

Upon observation, we categorized the areas served by the subway network into four ordinal categories: Poor, Limited, Satisfactory, and Excellent. These categories are defined at 0-1\%, 1-75\%, 75-90\%, and 90-100\% of area covered by transit, respectively. We defined these cutoffs using the distribution of subway coverage percentages and our own judgment on what constitutes a desert. As such, these cutoffs are specific to New York City and may not be perfectly reproducible. The following plot details the spatial locations of these transportation categories.


```{r, fig.height=5*1.2, fig.width=5*1.2}
nyc_compiled %>%
  ggplot() +
  geom_sf(aes(fill = transportation_desert_4cat), color = "#8f98aa") +
  scale_fill_manual(values=c("#895F32","#E9DBC2","#7D9B8A", "#395645"),
                       guide = guide_legend(title = "Subway Accessibility \nCategory"), na.value="#D6D6D6") +
  theme_minimal() +
  theme(panel.grid.major = element_line("transparent"),
        axis.text = element_blank()) +
  ggtitle("Subway Accessibility by \nNeighborhood in NYC")+ 
    theme(panel.grid.major = element_line("transparent"),
          plot.title = element_text(family="DIN Condensed", size = 35, face = "bold"),
          legend.title = element_text(size = 12),
          legend.text = element_text(size = 12)) + 
    guides(shape = guide_legend(override.aes = list(size = 8)),
           color = guide_legend(override.aes = list(size = 8)))
```
<center>
<img src="/media/bayes/plot_2.png" width="75%" height="75%" />
</center>


In the following two subsections, we describe how we understood and classified transportation deserts using two models.

## Naive Bayes Model

Naive Bayes Model is one of the most popular models for classifying a response variable with 2 or more categories.

We implemented a naive Bayes classifier on subway access because it is both computationally efficient and applicable to Bayesian classification settings where outcomes may have 2+ categories. Specifically, we fit transportation access by taking mean_income, percentage below the poverty line, and the number of grocery stores. Because we are predicting 4 levels of transportation access, we initially fit this model using the `e1071` package to classify subway transit level. 

```{r}
set.seed(454)
naive_model <- naiveBayes(transportation_desert_4cat ~ 
                              mean_income +
                              below_poverty_perc +
                              store_count,
                            data = nyc_naive)
```

```{r}
naive2_prediction <- naive_classification_summary_cv(naive_model, 
                                nyc_naive, 
                                y="transportation_desert_4cat", k=10)$cv
naive2_prediction %>%
  kable(align = "c", caption = "Naive Model - Summary") %>% 
  kable_styling()
```

Under 10-fold cross validation, our Naive Bayes model had an overall cross-validated accuracy of 51.11\%. However, our predictions were most accurate when predicting Poor transportation access (78.12\%) and Excellent transportation access (72.62\%). The following plot describes the cross-validated accuracy breakdown by each observed transportation access category.


```{r}
prediction %>%  
  ggplot(aes(x=transportation_desert_4cat, y=Probability, fill=Predictions)) + 
  geom_bar(position="fill", stat="identity")  +
  scale_y_continuous(labels = seq(0, 100, by = 25)) +
  labs(title="Accessibility Predictions by Observed Category", y="Proportion", x="")+
    theme(panel.grid.major.x = element_line("transparent"),
         # axis.text.y.left = element_blank(),
          axis.text.x.bottom = element_text(size = 12, face = "bold"),
          plot.title = element_text(family="DIN Condensed", size = 20, hjust=.5, face = "bold")) +
   scale_fill_manual(values=c("#895F32","#E9DBC2","#7D9B8A", "#395645"),
                       guide = guide_legend(title = "Subway Accessibility \nCategory"), na.value="#D6D6D6") 

```

<center>
<img src="/media/bayes/plot_3.png" width="100%" height="100%" />
</center>

From the plot, it is clear that our naive Bayes model is sufficient when predicting the extrema of subway (in)access given the overwhelming proportion of true-poor and true-excellent classifications. However, it remains imperfect when considering the inaccuracy for both the limited and satisfactory transportation categories, our data's distributions, and its interpretability.

Importantly, naive Bayes assumes that all quantitative predictors are normally distributed within each Y category and further it assumes (i.e. that predictors are independent within each Y category). Our data do not meet these assumptions, unfortunately. Further, naive Bayes is a blackbox classifier. That is, naive Bayes classification might give us accurate predictions, but it doesn’t give us a sense of where these predictions come from or subway access is related to the predictors.







## Ordinal Model

Having realized the shortcomings of the naive Bayes model, we wanted to see if there are any alternatives. We land on the ordinal regression model.

```{r}
model2 <- stan_polr(transportation_desert_4num ~ 
                      mean_income + below_poverty_line_perc + store_count, 
                    data =data_train, prior_counts = rstanarm::dirichlet(1),
                    prior=R2(0.5), iter=500, seed = 86437, refresh=0, prior_PD=FALSE)


tidy(model2, effects = "fixed", conf.int = TRUE, conf.level = 0.8) %>%
  mutate(term = case_when(
    term == "1|2" ~ "Poor | Limited",
    term == "2|3" ~ "Limited | Satisfactory",
    term == "3|4" ~ "Satisfactory | Excellent",
    TRUE ~ term
    ))%>%
  kable(align = "c", caption = "Ordinal Model - Summary") %>% 
  kable_styling()
```

Then using a function written by [Connie Zhang](https://connie-zhang.github.io/pet-adoption/modelling.html), we describe the accuracy of the ordinal model below.

```{r}
ordinal_accuracy<-function(post_preds,mydata){
  post_preds<-as.data.frame(post_preds)
  results<-c()
  for (j in (1:length(post_preds))){
    results[j]<-as.numeric(tail(names(sort(table(post_preds[,j]))))[4])
    }
  results<-as.data.frame(results)
  compare<-cbind(results,mydata$transportation_desert_4num)
  compare<-compare %>%mutate(results=as.numeric(results))
  compare<-compare %>% mutate(`mydata$transportation_desert_4num`=as.numeric(`mydata$transportation_desert_4num`))
  compare<-compare %>%mutate(accuracy=ifelse(as.numeric(results)==as.numeric(`mydata$transportation_desert_4num`),1,0))
  print(sum(compare$accuracy)/length(post_preds))
}
```




```{r}
nyc_compiled[is.na(nyc_compiled)] = 0

set.seed(86437)

my_prediction2 <- posterior_predict(
  model2, 
  newdata = data_test)

ordinal_accuracy(my_prediction2, data_test)
```

We are then 63.88% accurate when predicting transportation access categories.

```{r}
data_test %>%
  select(nta_id, borough, transportation_desert_4cat) %>%
  rownames_to_column() %>%
  left_join(., prediction_long, by="rowname") %>%
  
  ggplot(aes(x=transportation_desert_4cat, fill=predicted_desert)) + 
  geom_bar(position="fill")+
  scale_y_continuous(labels = seq(0, 100, by = 25)) +
  labs(title="Accessibility Predictions by Observed Category", y="Proportion", x="")+
    theme(panel.grid.major.x = element_line("transparent"),
         # axis.text.y.left = element_blank(),
          axis.text.x.bottom = element_text(size = 12, face = "bold"),
          plot.title = element_text(family="DIN Condensed", size =20, hjust=.5, face = "bold")) +
   scale_fill_manual(values=c("#895F32","#E9DBC2","#7D9B8A", "#395645"),
                       guide = guide_legend(title = "Subway Accessibility \nCategory"), na.value="#D6D6D6")
  
```

<center>
<img src="/media/bayes/plot_4.png" width="100%" height="100%" />
</center>

# Transportation and Structural Inequity

Transportation access is a pervasive structural issue. However, previous research on transportation access has demonstrated that many of these gaps in access also deepen the chasms of racial and class-based inequity. In this analysis, it seemed that low-income and racialized— typically Black and Latinx— communities were most likely to confront [transportation inequities](https://nyc.streetsblog.org/2021/06/18/report-racial-and-economic-inequities-in-transit-affect-accessibility-to-jobs-healthcare/). Additionally, we have observed that predominantly Black and Latinx neighborhoods in the Bronx faced some of the highest rates of eviction, likely indicating that these inequities may overlap.

This next section aims to connect transportation access to housing-inequities that we know also have racial and class dimensions. In particular, we wanted to assess transportation access’s relationship with immigrant community size, rental prices, and eviction counts by neighborhood. We felt that this was an appropriate direction because as you can see from the plots below:


```{r, fig.height=8*2, fig.width=8*2}
ggarrange(desert_map, rent_map, evict, noncit,
          ncol=2, nrow=2)
```

<center>
<img src="/media/bayes/plot_5.png" width="100%" height="100%" />
</center>

Transportation access is typically worst in areas with the highest densities of non-citizen residents, while it is the best in neighborhoods with the highest mean rental prices. Further, observe that eviction counts are highly concentrated in north and south NYC, where some of these neighborhoods have mixed-accessibility to transit. 

Unfortunately, rent, transportation access, and eviction counts may also be associated with respective density of nonwhite communities.

```{r, fig.height=16, fig.width=16+4}
ggarrange(desert_map, white, black, latinx, asian, ncol=3, nrow=2)
```


<center>
<img src="/media/bayes/plot_6.png" width="100%" height="100%" />
</center>

From the plots, neighborhoods with the highest densities of Black and Asian community members also have the poorest scores of subway access, while the converse holds true for neighborhoods with the highest proportion of White residents. 

Further, observe that eviction counts are most common in neighborhoods with the highest densities of Black and Latinx community members. The below faceted visualizations detail the specific relationships between the proportion of Black, Latinx, Asian, and White residents in a neighborhood with eviction counts.

<center>
<img src="/media/bayes/plot_7.png" width="100%" height="100%" />
</center>

Black and Latinx residential proportions by neighborhood are associated with increased eviction counts. However, we must specify that Black resident proportions are uniformly associated with increases in eviction counts across transportation levels, while the relationship between eviction counts and Latinx resident proportion is not. In contrast, both White and Asian proportions are uniformly associated with decreases in eviction counts.

Lastly, let's consider mean neighborhood rental prices. The following visualizations detail the relationships between the proportion of Black, Latinx, Asian, and White residents in a neighborhood with that neighborhood mean rental price.


<center>
<img src="/media/bayes/plot_8.png" width="100%" height="100%" />
</center>

Increases in White-resident proportions were uniformly associated with increases in average neighborhood rental prices across all transportation access categories. The converse is true for the relationship between Black and Latinx neighborhood densities and rental prices. It then seems that despite living in cheaper neighborhoods, both NYC's Black and Latinx communities are carrying the largest burden of eviction. 

Our next section details the statistical models we used to better characterize the clear relationships between housing inequities, urban racism, and transportation access.

## Bayesian Model Specification

In order to understand the respective distributions of immigrant population size, evictions, and mean rental prices in NYC, we fit 3 non-hierarchical Bayesian models with each variable as an outcome. In previous iterations, we fit hierarchical models on these same relationships where neighborhood values were grouped by borough. We initially felt that the internal diversity of boroughs with respect to housing, demographics, and wealth made it more appropriate as a grouping variable. However, we ultimately saw that the non-hierarchical models we fit were largely comparable to their hierarchical counterparts. Further, we considered how even though there is a dramatic internal diversity in boroughs their effects on our outcomes were still sufficiently meaningful to warrant a true statistical adjustment.

We list our non-hierarchical models and their predictors below:

- Non-Citizen Count Model (1)		
  + Predictors: `transportation_desert_4cat`, `total_pop`, `borough`, `gini_neighborhood`, `mean_income`, `mean_rent`, `unemployment_perc`, `black_perc`, `latinx_perc`, `asian_perc`

- Average Rental Price Model (2)
  + Predictors: `transportation_desert_4cat`, `borough`, `gini_neighborhood`, `mean_income`, `black_perc`, `latinx_perc`, `asian_perc`, `bus_count`,`school_count`,`store_count`, `noncitizen_perc`
  
- Eviction Count Model (3)
  + Predictors: `transportation_desert_4cat`, `total_pop`, `borough`, `gini_neighborhood`, `mean_income`, `mean_rent`, `black_perc`, `latinx_perc`, `asian_perc`,
  `bus_count`,`store_count`,`unemployment_perc`, `uninsured_perc`
  

For 1 and 3, we used weakly informative priors and allowed `stan_glm` to estimate initial priors. This decision was ultimately due to our unfamiliarity with NYC's history of evictions and non-citizen population. For Model 2, we specified a normal prior with mean 1600 and standard deviation at 20 using previous experience as renters both in NYC and other comparable major cities. Across all count models, we fit both Poisson and Negative-Binomial prior distributions for the observed eviction and non-citizen counts. However, we observed an inconsistent spread of variance and increased 0 counts in both the eviction and immigrant count data. As such, we felt that a Negative-Binomial distribution was more appropriate for both data.

Our model specifications are detailed in the following subsections

### Model 1: Immigrant/Non-Citizen Count

```{r}
noncit_model <- stan_glm(
  noncitizen_count ~
    transportation_desert_4cat + 
    total_pop + borough + gini_neighborhood + 
    mean_income + mean_rent + unemployment_perc + 
    black_perc + latinx_perc + asian_perc,
  data = modeling_data,
  family = neg_binomial_2,
  chains = 5, iter = 2000*2, seed = 84735, refresh = 0
)

```

`$$
\begin{aligned}
\text{Non-Citizen Count} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu, r) \; \; \; \; \text{where} \log(\mu) = \beta_{0c} + \sum^{14}_{k=1}X_{k}\beta_k \\
\beta_{0c} &\sim N(0,2.5^2)\\				
\beta_{1} &\sim N(0,6.2785^2)\\				
\beta_{2} &\sim N(0,6.7918^2)\\				
\beta_{3} &\sim N(0,5.0879^2)\\				
\beta_{4} &\sim N(0, 0.0001^2)\\				
\beta_{5} &\sim N(0,5.5216^2)\\				
\beta_{6} &\sim N(0,6.5781^2)\\				
\beta_{7} &\sim N(0,5.2519^2)\\				
\beta_{8} &\sim N(0,56.96^2)\\				
\beta_{9} &\sim N(0,0.006^2)\\				
\beta_{10} &\sim N(0,0.3317^2)\\	
\beta_{11} &\sim N(0,7.3986^2)\\				
\beta_{12} &\sim N(0,0.9779^2)\\				
\beta_{13} &\sim N(0,1.0952^2)\\				
\beta_{14} &\sim N(0,1.638^2)\\				
r & \sim Exp(1) \\
\end{aligned}
$$`

### Model 2: Mean Neighborhood Rental Prices

```{r}
rent_model <- stan_glm(
  mean_rent  ~ 
  transportation_desert_4cat + 
    bus_count + school_count + store_count +
    total_pop + borough + 
    gini_neighborhood + mean_income + black_perc + latinx_perc + asian_perc,
  data = modeling_data, 
  family = gaussian,
  prior_intercept = normal(1600 , 20),
  chains = 5, iter = 2000*2, seed = 84735, refresh = 0
)
```

`$$
\begin{aligned}
\text{Mean Rent} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{Normal}(\mu, \sigma) \; \; \; \; \text{where} \mu = \beta_{0c} + \sum^{14}_{k=1}X_{k}\beta_k \\
\beta_ {0c} &\sim N(1600,20^2)\\				
\beta_{1} &\sim N(0,47.315^2)\\				
\beta_{2} &\sim N(0,51.1837^2)\\				
\beta_{3} &\sim N(0,38.3432^2)\\				
\beta_{4} &\sim N(0,0.4958^2)\\				
\beta_{5} &\sim N(0,2.9378^2)\\				
\beta_{6} &\sim N(0,0.4371^2)\\				
\beta_{7} &\sim N(0,0.0008^2)\\				
\beta_{8} &\sim N(0,41.6113^2)\\				
\beta_{9} &\sim N(0,49.5728^2)\\				
\beta_{10} &\sim N(0,39.5783^2)\\
\beta_{11} &\sim N(0,429.2544^2)\\				
\beta_{12} &\sim N(0,0.0454^2)\\				
\beta_{13} &\sim N(0,7.3695^2)\\				
\beta_{14} &\sim N(0,8.2532^2)\\				
\beta_{15} &\sim N(0,12.3441^2)\\
\sigma & \sim Exp(0.13) \\
\end{aligned}
$$`

We chose our prior specifications of mean rental price using Juthi’s experience renting in NYC and a group conversation about typical rental prices we would elect to pay in NYC, Los Angeles, and other major cities we have lived in or around. 


### Model 3: Eviction Count

```{r}
eviction_model <- stan_glm(
  eviction_count  ~ 
    transportation_desert_4cat +
    total_pop + borough + 
    gini_neighborhood + mean_income +  mean_rent + 
    black_perc + latinx_perc + asian_perc,
  data = modeling_data, 
  family = neg_binomial_2,
  chains = 5, iter = 2000*2, seed = 84735, refresh = 0
)
```

`$$
\begin{aligned}
\text{Eviction Count} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu, r) \; \; \; \; \text{where} \log(\mu) = \beta_{0c} + \sum^{14}_{k=1}X_{k}\beta_k \\
\beta_{0c} &\sim N(0,2.5^2)\\				
\beta_{1} &\sim N(0,6.2992^2)\\				
\beta_{2} &\sim N(0,6.7813^2)\\				
\beta_{3} &\sim N(0,4.9972^2)\\				
\beta_{4} &\sim N(0,1e-04^2)\\				
\beta_{5} &\sim N(0,5.4697^2)\\				
\beta_{6} &\sim N(0,6.443^2)\\				
\beta_{7} &\sim N(0,5.3347^2)\\				
\beta_{8} &\sim N(0,56.9002^2)\\				
\beta_{9} &\sim N(0,0.0074^2)\\				
\beta_{10} &\sim N(0,0.5699^2)\\
\beta_{11} &\sim N(0,0.993^2)\\				
\beta_{12} &\sim N(0,1.142^2)\\				
\beta_{13} &\sim N(0,1.6003^2)\\	
r & \sim Exp(1) \\
end{aligned}
$$`



In the following sections, we go through each particular model's outcome and what they tell us about the relationships between transportation and housing.

## Model Performance

### Model 1: Immigrant/Non-Citizen Count

```{r}
pp_check(noncit_model)+ 
  xlab("Non-Citizen Count") +
  labs(title = "Negative Binomial Model of \nImmigrant/Non-Citizen Count")+
  theme(plot.title =  element_text(family="DIN Condensed", face="bold", size=25, hjust=.5)) 
```

<center>
<img src="/media/bayes/plot_9.png" width="100%" height="100%" />
</center>


```{r}
set.seed(84735)

ppc_intervals(list, yrep = predictions_non_citizen,
              prob_outer = 0.8) +
  ggplot2::scale_x_continuous(
    labels = non_citizen_clean$nta_id,
    breaks = 1:nrow(non_citizen_clean)) +
	xaxis_text(angle = 90,  hjust = 1) +
  theme_linedraw()+
  theme(panel.grid.major = element_line("transparent"),
        axis.title.x = element_blank(),
        axis.text.x = element_blank())
        
```

<center>
<img src="/media/bayes/plot_10.png" width="100%" height="100%" />
</center>

```{r}
prediction_summary_cv(model = noncit_model, data=modeling_data, k=10)$cv %>%
  kable(align = "c", caption = "Non-Citizen Model - Cross-Validated Error Metrics") %>% 
  kable_styling()
```

Under 10-fold cross-validation, we can see how our model performs when predicting new, randomly-assorted, non-citizen counts. In this case, it seems that the 10-fold CV median absolute error (MAE) is 2704. (2.38 sd) meaning that the typical difference between the averages of our posterior prediction distributions is 2704 people or 2.37 standard deviations away from the observed number of non-citizen counts. Beyond accuracy metrics, it also seems that only 10.55\% and 51.67\% of the observed non-citizen counts are falling within their 100% and 95% posterior prediction intervals. Together, this indicates our model's performance when predicting non-citizen counts may need further work.

```{r}
library(kableExtra)
tidy(noncit_model, effects = "fixed", conf.int = TRUE, conf.level = 0.8)%>% 
  
  mutate(estimate= ifelse(term == "(Intercept)", exp(estimate), (exp(estimate)-1)*100), 
         conf.low= ifelse(term == "(Intercept)", exp(conf.low), (exp(conf.low)-1)*100), 
         conf.high = ifelse(term == "(Intercept)", exp(conf.high), (exp(conf.high)-1)*100))%>%
  filter(conf.low	> 0 & conf.high > 0 | conf.low	< 0 & conf.high < 0) %>%
  kable(align = "c", caption = "Non-Citizen Model - Summary") %>% 
  kable_styling()
```

- Limited Subway Access: When controlling for all other predictors, a neighborhood with limited transit access is expected to have approximately 21.82% more non-citizens than a neighborhood with poor transit access. There's an 80% probability that this increase could lie anywhere between (7.48%, 38.27%) non citizen residents, indicating that neighborhoods with limited transit access almost certainly have more non citizen residents than neighborhoods with poor access.
- Satisfactory Subway Access: When controlling for all other predictors, a neighborhood with satisfactory transit access is expected to have approximately 30.22% more non-citizens than a neighborhood with poor transit access. There's an 80% probability that this increase could lie anywhere between (13.99%, 49.09%) non citizen residents, indicating that neighborhoods with satisfactory transit access almost certainly have more non citizen residents than neighborhoods with poor access.
- Excellent Subway Access: When controlling for all other predictors, a neighborhood with excellent transit access is expected to have approximately 29.84% more non-citizens than a neighborhood with poor transit access. There's an 80% probability that this increase could lie anywhere between (13.34%, 48.51%) non citizen residents, indicating that neighborhoods with excellent transit access almost certainly have more non citizen residents than neighborhoods with poor access.
- Brooklyn: When controlling for all other predictors, a neighborhood in Brooklyn is expected to have approximately 16.61% more non-citizens than a neighborhood in the Bronx. There's an 80% probability that this increase could lie anywhere between (1.68%, 34.07) non citizen residents, indicating that neighborhoods in Brooklyn almost certainly have more non citizen residents than the Bronx, but the magnitude of this increase may vary. 
- Manhattan: When controlling for all other predictors, a neighborhood in Manhattan is expected to have approximately 20.58% more non-citizens than in the Bronx. There's an 80% probability that this increase could lie anywhere between (2.268, 42.85) non citizen residents, indicating that neighborhoods in Manhattan almost certainly have more non citizen residents than the Bronx. 
- Mean Income: When controlling for all other predictors, a 100 dollar increase in mean neighborhood income is associated with approximately a 0.0793% decrease in non citizen count. However, there is a 80% chance that the decrease in non citizen count may be any value between (0.1102, 0.0476), indicating that there is almost certainly a negative relationship between mean income and non citizen count, but its magnitude may vary.
- Mean Rent: When controlling for all other predictors, a 100 dollar increase in mean neighborhood rent is associated with approximately a 6.29% increase in non citizen counts. However, there is an 80% chance that the increase in non citizen count may be any value between (3.96%, 8.68%), indicating that there is almost certainly a positive relationship between mean rent and non citizen count, but its magnitude may vary.
- Black Percentage: When controlling for all other predictors, a 10% increase in the Black population in a neighborhood is associated with approximately a 5.20% increase in non citizen counts. However, there is an 80% chance that the increase in non citizen count may be any value between (2.99%, 7.45%), indicating that there is almost certainly a positive relationship between Black resident percentage and non citizen count, but its magnitude may vary.
- Latinx Percentage: When controlling for all other predictors, a 10% increase in the Latinx population in a neighborhood is associated with approximately a 13.46% increase in non citizen count. However, there is an 80% chance that the increase in non citizen count may be any value between (10.08%, 17.07%), indicating that there is almost certainly a positive relationship between Latinx resident percentage and non citizen count, but its magnitude may vary.
- Asian Percentage: When controlling for all other predictors, a 10% increase in the Asian population in a neighborhood is associated with approximately a 19.63% increase in non citizen count. However, there is an 80% chance that the increase in non citizen count may be any value between (15.75%, 23.69%), indicating that there is almost certainly a positive relationship between Asian resident percentage and non citizen count, but its magnitude may vary.


### Model 2: Mean Neighborhood Rental Prices



<center>
<img src="/media/bayes/plot_11.png" width="100%" height="100%" />
</center>


```{r}
ppc_intervals(list, 
              yrep = predictions_rent,
              prob_outer = 0.8) +
  scale_x_continuous(
    labels = rent_clean$nta_id,
    breaks = 1:nrow(rent_clean)) +
  
  xaxis_text(angle = 90,  hjust = 1) +
  
  theme_linedraw() +
  
  theme(panel.grid.major = element_line("transparent"),
        axis.title.x = element_blank(),
        axis.text.x = element_blank())
```
<center>
<img src="/media/bayes/plot_12.png" width="100%" height="100%" />
</center>


```{r}

prediction_summary_cv(model = rent_model, data=nyc_compiled, k=10)$cv %>%
  kable(align = "c", caption = "Mean Rent Model - Cross-Validated Error Metrics") %>% 
  kable_styling()
```

Under 10-fold cross-validation, we can see how our model performs when predicting new, randomly-assorted, mean rental prices. In this case, it seems that the 10-fold CV median absolute error (MAE) is 119.256 (0.68 sd) meaning that the typical difference between the averages of our posterior prediction distributions is 119.25 dollars or .68 standard deviations away from the observed mean rental price. Beyond accuracy metrics, it also seems that only 49.55% and 77.23% of the observed mean rental price estimates are falling within their 100% and 95% posterior prediction intervals, respectively. That's great! Together, this indicates our model's performance when predicting mean rental prices by neighborhood is superb.

```{r}
library(kableExtra)
tidy(rent_model, effects = "fixed", conf.int = TRUE, conf.level = 0.8)%>% 
  filter(conf.low	> 0 & conf.high > 0 | conf.low	< 0 & conf.high < 0) %>%
  kable(align = "c", caption = "Rent Model - Summary") %>% 
  kable_styling()
```

For rent model:

- Store Count: When controlling for all other predictors, as grocery store and food vendor counts in a neighborhood increase by 1, rent increases by approximately $1.049 dollars. However, there is a 80% chance that the *increase* associated with store count may be any value between (0.307, 1.799) dollars, indicating that there is almost certainly a positive relationship between store count and rent, but its magnitude may vary slightly.

- Mean Income: When controlling for all other predictors, a *100 dollar* increase in mean neighborhood income is associated with approximately a $1.20 increase in rent. However, there is a 80% chance that the *increase* associated with rental price increases may be any value between (1.129, 1.284) dollars, indicating that there is almost certainly a positive relationship between mean income and mean rental prices, but its magnitude may vary slightly.
  
- Black Percentage: When controlling for all other predictors, a *10%* increase in the black proportion of a neighborhood is associated with approximately a $10.20 *decrease* in rent. However, there is an 80% chance that the *decrease* associated with rent may be any value between (19.36, 0.685) dollars, indicating that there is almost certainly a negative relationship between black resident percentage and rent, but its magnitude may vary.

- Asian Percentage: When controlling for all other predictors, a *10%* increase in the Asian population in a neighborhood is associated with approximately a $20.52 *increase* in rent. However, there is an 80% chance that the *increase* associated with rent may be any value between (5.763, 34.87), indicating that there is almost certainly a positive relationship between Asian resident percentage and rent, but its magnitude may vary greatly.



### Model 3: Eviction Count

```{r}
pp_check(eviction_model)+ 
  xlab("Eviction Count") +
  xlim(0,2000)+
  labs(title = "Negative Binomial Model of \nEviction Count")+
  theme(plot.title =  element_text(family="DIN Condensed", face="bold", size=25, hjust=.5)) 
```

<center>
<img src="/media/bayes/plot_13.png" width="100%" height="100%" />
</center>


```{r}
ppc_intervals(list, 
              yrep = predictions_eviction,
              prob_outer = 0.8) +
  scale_x_continuous(
    labels = eviction_clean$nta_id,
    breaks = 1:nrow(eviction_clean)) +
  
  xaxis_text(angle = 90,  hjust = 1) +
  
  theme_linedraw() +
  
  theme(panel.grid.major = element_line("transparent"),
        axis.title.x = element_blank(),
        axis.text.x = element_blank())
```
<center>
<img src="/media/bayes/plot_14.png" width="100%" height="100%" />
</center>

```{r}
prediction_summary_cv(model = eviction_model, data=nyc_compiled, k=10)$cv %>%
  kable(align = "c", caption = "Eviction Model - Cross-Validated Error Metrics") %>% 
  kable_styling()
```


Under 10-fold cross-validation, we can see how our model performs when predicting new, randomly-assorted, eviction counts. In this case, it seems that the 10-fold CV median absolute error (MAE) is 63.139 (1.56 sd) meaning that the typical difference between the averages of our posterior prediction distributions is 63 counts or 1.56 standard deviations away from the observed number of eviction counts. Beyond accuracy metrics, it also seems that only 25.89% and 58.92% of the observed eviction counts are falling within their 100% and 95% posterior prediction intervals. Together, this indicates our model's performance when predicting eviction counts may need further work.

```{r}
library(kableExtra)
tidy(eviction_model, effects = "fixed", conf.int = TRUE, conf.level = 0.8)%>% 
  
  mutate(estimate= ifelse(term == "(Intercept)", exp(estimate), (exp(estimate)-1)*100), 
         conf.low= ifelse(term == "(Intercept)", exp(conf.low), (exp(conf.low)-1)*100), 
         conf.high = ifelse(term == "(Intercept)", exp(conf.high), (exp(conf.high)-1)*100))%>%
  filter(conf.low	> 0 & conf.high > 0 | conf.low	< 0 & conf.high < 0) %>%
  kable(align = "c", caption = "Eviction Model - Summary") %>% 
  kable_styling()
```


After removing predictors whose 80% credible intervals included the possibility of non-effect when controlling for other covariates, we found that there were 13 remaining predictors of an arbitrary neighborhood's eviction counts. In the following list, categorical predictors’ classes were listed together:

- Subway Accessibility (Limited, Satisfactory, Excellent)
- Total Population
- Borough (Brooklyn, Manhattan, Queens)
- Gini Index
- Mean Income
- Mean Rent
- Black Percentage
- Latinx Percentage
- Asian Percentage

In the following section, we interpret each predictor. However, we included total population to control for eviction count's incidence relative to neighborhood population. As such, we will not interpret its specific `$\beta$` value.


Next, we interpret each predictor:

- Limited Subway Access: When controlling for all other predictors, a neighborhood with limited access to the subway is expected to have approximately 39.81% more eviction counts than a neighborhood with poor subway access. However, there is an 80% chance that the relationship may be any value between (21.10%, 61.52%), indicating that there is almost certainly an increase in white population counts between these two neighborhoods, but its explicit magnitude may vary.
- Satisfactory Subway Access: When controlling for all other predictors, a neighborhood with satisfactory access to the subway is expected to have approximately 41.25% more eviction counts than a neighborhood with poor subway access. However, there is an 80% chance that the relationship may be any value between (21.89%, 64.39%), indicating that there is almost certainly an increase in white population counts between these two neighborhoods, but its explicit magnitude may vary.
- Excellent Subway Access: When controlling for all other predictors, a neighborhood with excellent access to the subway is expected to have approximately 40.58% more eviction counts than a neighborhood with poor subway access. However, there is an 80% chance that the relationship may be any value between (20.67%, 63.63%), indicating that there is almost certainly an increase in white population counts between these two neighborhoods, but its explicit magnitude may vary.
- Gini Index: When controlling for all other predictors, 1 increases in income inequality are associated with approximately 658.52% increases in the number of evictions. However, there is a 80% chance that the relationship may be any value between (73.738%, 3243.93%), indicating that there is almost certainly a strong, positive, relationship between these two variables, but its magnitude may vary.
- Mean Income: When controlling for all other predictors, 100 dollar increases in mean neighborhood income are associated with approximately 0.1182% decreases in the number of evictions. However, there is a 80% chance that the relationship may be any value between (-0.0799%, -0.1553%), indicating that there is almost certainly a weak, negative, relationship between these two variables, but its magnitude may vary slightly.
- Mean Rent: When controlling for all other predictors, 100 dollar increases in mean neighborhood rental prices are associated with approximately 4.55% increases in the number of evictions. However, there is a 80% chance that the relationship may be any value between (1.94%, 7.13%), indicating that there is almost certainly a positive relationship between these two variables, but its magnitude may vary slightly.
- Black Percent: When controlling for all other predictors, 10% increases in the proportion of Black residents in a neighborhood are associated with approximately 17.77% increases in the number of evictions. Further, there is a 80% chance that the relationship may be any value between (15.03%, 20.67%), indicating that there is almost certainly a strong, positive, relationship between these two variables, but its magnitude may vary slightly. 
- Latinx Percent: When controlling for all other predictors, 10% increases in the proportion of Latinx residents in a neighborhood are associated with approximately 6.60% increases in the number of evictions. Further, there is a 80% chance that the relationship may be any value between (2.73%, 10.69%), indicating that there is almost certainly a weak, positive, relationship between these two variables, but its magnitude may vary.
- Asian Percent: When controlling for all other predictors, 10% increases in the proportion of Asian residents in a neighborhood are associated with approximately 4.21% decreases in the number of evictions. Further, there is a 80% chance that the relationship may be any value between (-.2344%, -7.88%), indicating that there is almost certainly a weak, negative, relationship between these two variables, but its magnitude may vary slightly.

# Discussion

We've confirmed that even after adjusting for income, rental prices, income inequality, and borough differences, neighborhoods in NYC with a substantial proportion of Black and Latinx residents are experiencing dramatic increased risks of eviction. Interestingly, when accounting for economic and demographic predictors, the relationships with transportation we expected to see were reversed. That is, increased transportation access was associated with more evictions. This could be related to the increased population sizes in areas with the best access to transportation (e.g. Manhattan) or it could be because the disparities that transportation inaccess reflects (e.g. racial and class-based tensions) have been accounted for. However, our results are almost certainly confounded by the spatial relationships between neighborhoods and the effect of other unmeasured structural predictors in housing equity such as a neighborhood's rent control policies or the reasons behind eviction.




