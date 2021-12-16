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

<script src="//yihui.org/js/math-code.js" defer></script>
<!-- Just one possible MathJax CDN below. You may use others. -->
<script defer
  src="//mathjax.rstudio.com/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

<style>
    table {
        width: 100%;
    }
</style>

# Motivation

In collaboration with [Juthi Dewan](https://juthidewan-portfolio.netlify.app), [Sam Ding](https://sdingx.github.io/portfolio/), and Vichy Meas, we designed this project for our Bayesian Statistics course taught by [Dr. Alicia Johnson](https://ajohns24.github.io/portfolio/). We would like to thank Alicia for guiding us through Bayes and the capstone experience!

A reproducible version of this blog post with all code can be found [here](https://freddybarragan.netlify.app/media/bayes/bayes_final.html).

We were initially interested in characterizing New York City's internal racial dynamics using demography, geographic mobility, community health, and economic outcomes. As this project developed, we found ourselves thinking about the relationships between transportation (in)access and housing inequity. There are two main sections in our project: **Subway Accessibility** and **Transportation and Structural Inequity**. 

In **Subway Accessibility**, we explore transportation deserts, and the significant determinants of subway access in New York City are using two Bayesian classification models. If you have questions or thoughts about this section in particular, please feel free to reach out to [Sam](https://sdingx.github.io/portfolio/) or Vichy by email!

While in **Transportation and Structural Inequity**, we extend our discussion of transportation access to study its relationship to rental prices and evictions using both simple and hierarchical Bayesian multivariate regression. In the [extended document](https://freddybarragan.netlify.app/media/bayes/bayes_final.html), we also fit non-hierarchical spatial models to control for the underlying spatial relationships between neighborhoods. However, we omit major discussion of these models in this blog post as Bayesian spatial regression was beyond the scope of this course. If you have questions about these models, please reach out to either [Juthi](https://juthidewan-portfolio.netlify.app) or me by email!

First, however, let us do a data introduction:




# Data Introduction


All data used in this project are from two primary sources: the Tidycensus package and NYC Open Data. 

[Tidycensus](https://walker-data.com/tidycensus/index.html) is an R package interface, developed by Kyle Walker and Matt Herman, that enables easy access to the U.S. Census Bureau's data APIs and returns Tidyverse-ready data frames from various U.S. Census Bureau datasets. We drew our demographic and socioeconomic data from the 2019 American Community Survey results in the Tidycensus package. A summary of our ACS data variables is below:

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
- `noncitizen_count`: Number of Foreign-Born People by Census Tract
- `uninsured_count`: Number of Uninsured Citizens of any Age by Census Tract
 - `gini_neighborhood`: Income inequality measured by the Gini Index per Census Tract

Note that these predictors are all measured at the census level. To aggregate these estimates at the neighborhood level, we performed two transformations:

- Compute total sums for every count-based measurement.
- Compute mean average estimates for remaining non-count predictors.

Then, we divide by the total population in each neighborhood to define scaled demographic metrics for count-based demographic predictors. They are as follows:

- `asian_perc`: Percentage of Asian People 
- `white_perc`: Percentage of White People 
- `black_perc`: Percentage of Black People 
- `latinx_perc`: Percentage of Latinx People 
- `native_perc`: Percentage of Native People 
- `noncitizen_perc`: Percentage of Foreign-Born People  
- `uninsured_perc`: Percentage of Uninsured Citizens of any Age  
- `unemployment_perc`: Percentage of People on Unemployment 
- `below_poverty_line_perc`: Percentage of people living below the 100% poverty line.

We used [NYC's Open Data](https://opendata.cityofnewyork.us/data/) portal and the [Baruch College GIS Lab](https://www.baruch.cuny.edu/confluence/pages/viewpage.action?pageId=28016896) to collect information on the remaining predictors. In particular, we used geotagged locations of [Subway Stops](https://data.cityofnewyork.us/Transportation/Subway-Stations/arq3-7z49), [Bus Stops](https://www.baruch.cuny.edu/confluence/pages/viewpage.action?pageId=28016896), [Grocery Stores](https://data.ny.gov/Economic-Development/Retail-Food-Stores/9a8c-vfzj), [Schools](https://data.cityofnewyork.us/Education/School-Point-Locations/jfju-ynrr), and [Evictions](https://data.cityofnewyork.us/City-Government/Evictions/6z8x-wfk4) from the Departments of Transportation, Health, Education, and Housing, respectively, to calculate the following variables: 

- `school_count`: Public school counts  
- `eviction_count`: Eviction counts  
- `store_count`: Grocery store and food vendor counts  
- `sub_count`: Subway station counts  
- `bus_count`: Bus station counts 
- `perc_covered_by_transit`: Percent of Neighborhood Within Walking Distance (.5 miles) of Any Subway Stop. 
- `transportation_desert_3cat`: Subway Accessibility  (Poor, Typical, Excellent)

The process involved grouping geotagged locations by the defined neighborhood boundary regions in R's S.F. package and ArcGIS. We will detail the process of identifying subway deserts in the "Subway Deserts" section. 

# Data Summaries 

We present a [numeric summary](https://freddybarragan.netlify.app/media/bayes/bayes_final.html#Data_Summaries) on our data with 224 observations of 38 variables. However, note that we use percent equivalents for most demographic count variables.

We found that Manhattan had the highest population counts, highest mean rental prices, highest mean income, highest income inequality (e.g., Gini value), the most neighborhoods with excellent subway access, and the largest proportion of white citizens. 

Conversely, the Bronx has the lowest mean income and highest eviction counts while having the highest proportions of people living below the poverty line and the highest number of people with limited subway access. Notably, the Bronx also had the highest densities of Latinx and Black residents of any other borough in New York, meaning that the Black and Latinx residents in New York are experiencing the burden of New York's structural inequities.

We will explicitly touch on the connections between demographics, transportation access, and housing in the following sections.

# Subway Accessibility

New York City is the most populous city in the U.S. with [more than 8.8 million people](https://en.wikipedia.org/wiki/New_York_City). To support the daily commutes of its residents, NYC also built the New York City Subway, the oldest, longest, and currently busiest subway system in the U.S., averaging [approximately 5.6 million daily rides on weekdays and a combined 5.7 million rides each weekend](https://en.wikipedia.org/wiki/New_York_City_Subway). 

Compared to other U.S. cities where automobiles are the most popular mode of transportation (ahem, Minneapolis), only 32% of NYC's population chooses to commute by car. NYC's far-reaching transit system is unique, given that [more than 70% of the population](https://en.wikipedia.org/wiki/Modal_share) commutes by car in other metropolitan areas.

Despite having the most extensive transit network in the entire U.S., NYC lacks transit accessibility for some neighborhoods. The consensus in academia is that residents who walk more than 0.5 miles to get to reliable transit lack transportation access or reside in a transportation desert. We adopted this concept to study these gaps in transportation access for our project. Specifically, we attempt to identify and study "Subway Deserts." 

## Subway Desert Definition 

Extending the USDA's definition of a food desert, we define Subway Deserts as the percentage of a neighborhood— or any arbitrary geographic area— within walking distance of any subway stop. Citing the U.S. Federal Highway Administration, we defined walking distance as [a 0.5-mile radius](https://safety.fhwa.dot.gov/ped_bike/ped_transit/ped_transguide/ch4.cfm) and computed these regions in ArcGIS. We chose subway stations because of the subway's reliable frequency, high connectivity between boroughs, and high ridership per vehicle. Our argument against including the number of bus stops in our calculations of transportation access is that the quantity of bus stops does not accurately imply public transport accessibility due to the variability in bus efficiency, punctuality, and use. A significant limitation of our work was the omission of Staten Island because it is not connected to any other borough by subway. Rather, Staten Island users typically drive or train into the city. Further, we felt that the inclusion of Staten Island would mischaracterize the relationship between lacking access and not needing access since Staten Island is an overwhelmingly white, wealthy borough that has high levels of [car ownership](https://edc.nyc/article/new-yorkers-and-their-cars).

We first geocoded subway stop locations in NYC from the NYC Department of Transportation. Then, using ArcGIS, we created a 0.5-mile-radius buffer for each station and calculated what percent of each neighborhood was covered by a buffer region. We display an example below.

<center>
<img src="/media/bayes/plot_1.png" width="50%" height="50%" />
</center>


In the graph, buffer zones are in light pink with overlapping boundaries dissolved between stations, while the dark pink dots indicate the exact geographic locations of the stations. Each neighborhood, then, had a percentage score that defined its subway accessibility score. 

Upon observation, we categorized the areas served by the subway network into four ordinal categories: Poor, Typical, and Excellent. We defined these categories as 0-25\%, 25-75\%, and 75-100\% of the area within walking distance to some subway stop, respectively. We defined these cutoffs using the distribution of subway coverage percentages, plotted below. 

<center>
<img src="/media/bayes/ordinal_cuts.png" width="50%" height="50%" />
</center>

We found that the original data has a bimodal distribution with observations heavily concentrated in 0-25% and 75-100%. This distribution is likely due to Manhattan's over-saturated transit coverage and the lack of subway access in the suburban neighborhoods of Queens. 

There was an unequal distribution of observations within different access levels where the Poor and Typical categories have fewer observations combined than the Excellent category.

The following plot details the spatial locations of these transportation categories.

<center>
<img src="/media/bayes/plot_2.png" width="75%" height="75%" />
</center>


We will discuss this plot further in the "Transportation and Inequity" section. However, observe that transportation access is best in Manhattan (top-left) and Brooklyn (bottom-left), while the worst in Queens (right) and the Bronx (center-top).

We describe how we understood and classified transportation deserts using two models in the following two subsections.

## Naive Bayes Model

The naive Bayes Model is one of the most popular models for classifying a response variable with two or more categories.

We implemented a Naive Bayes classifier on subway access because it is computationally efficient and applicable to Bayesian classification settings where outcomes may have 2+ categories. We fit transportation access categories with the predictors mean income, percentage below the poverty line, and the number of grocery stores. Because we are predicting three levels of transportation access, we initially fit this model using the `e1071` package to classify subway transit levels.  We fit our model below.

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
<center>

| transportation\_desert\_4cat | Poor        | Limited     | Satisfactory | Excellent   |
| ---------------------------- | ----------- | ----------- | ------------ | ----------- |
| Poor                         | 78.12% (25) | 12.50% (4)  | 0.00% (0)    | 9.38% (3)   |
| Limited                      | 31.43% (11) | 11.43% (4)  | 20.00% (7)   | 37.14% (13) |
| Satisfactory                 | 20.69% (6)  | 17.24% (5)  | 6.90% (2)    | 55.17% (16) |
| Excellent                    | 9.52% (8)   | 15.48% (13) | 2.38% (2)    | 72.62% (61) |

</center>


Under 10-fold cross-validation, our Naive Bayes model had an overall cross-validated accuracy of 51.11\%. However, our predictions were most accurate when predicting Poor transportation access (78.12\%) and Excellent transportation access (72.62\%). The following plot describes the cross-validated accuracy breakdown by each observed transportation access category.


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

From the plot, it is clear that our naive Bayes model is sufficient when predicting the extrema of subway accessibility given the overwhelming proportion of true-poor and true-excellent classifications. However, it remains imperfect when considering the inaccuracy for both the limited and satisfactory transportation categories, our data distributions, and interpretability.

Importantly, naive Bayes assumes that all quantitative predictors are normally distributed within each Y category, and it assumes that predictors are independent within each Y category. Below we verify whether this is an appropriate assumption for our data.

<center>
<img src="/media/bayes/naive_assumption.png" width="100%" height="100%" />
</center>

Naive Bayes's assumption of normality within categories does not hold, unfortunately. These stratified distributions imply that although this is a "good" classifier, it is not appropriate. Further, naive Bayes is a black box classifier. So, although our classifications might be accurate, we will never understand where these predictions come from or how subway access is related to these predictors. Our next section details another Bayesian classification model that can serve as an alternative to Naive Bayes.



## Ordinal Model

An ordinal or ordered logistic regression model predicts the outcome of an ordinal variable, a categorical variable whose classes exist on an arbitrary scale where only the relative ordering between different values is significant. In this case, our subway desert category is an ordinal variable with categories ranging from the least covered to the most covered (e.g. `$[1,2,3]$`) by NYC's subway stops. Once again, we defined these categories by splitting the percentage covered using two cut-points, `$\zeta_1$` and `$\zeta_2$`, to create three ordered categories— Poor, Typical, and Satisfactory. Our `$\ zeta$` are listed below.

`$$
\begin{align*}
\zeta_1 = 0.25 \\
\zeta_2 = 0.75 \\
\end{align*}
$$`

Next, we introduce a latent variable `$y^*$`, as the linear combination of the `$k$` predictor variables. Here, we selected mean neighborhood income (`$X_1$`), percentage below the poverty line in a neighborhood (`$X_2$`), the number of grocery stores and food vendors in a neighborhood (`$X_3$`), and three dummy variables for the borough of our neighborhood (`$X_4, X_5, X_6$`) as our predictors.
Then we predict the transportation desert status `$Y$` for the `$i$`th neighborhood with the following model:
 
`$$
\begin{align*}
Y_i| \zeta_1, \zeta_2,\zeta_3,\beta_1,\dots,\beta_6 &=
\begin{cases}
1 & y^{*} < \zeta_1 \\
2 & \zeta_1 \leq y^{*} < \zeta_2 \\
3 &  \zeta_2 \leq  y^{*} \\
\end{cases} 
\\
\\
y_i^{*} & = \sum_{k=1}^6 \beta_kX_i \\
\end{align*} \\
$$`

Where

`$$\begin{align*}
\zeta_1 = 0.25 \\
\zeta_2 = 0.75 \\
\end{align*}$$`

`$$
\beta_k \sim N(m_{k}, s_{k}^{2}) \\
$$`

It is important to note that there is no intercept in `$y_i^*$`, which is an omission by the construction of `stan_polr`'s model and the multi-class outcomes for this ordinal model. 

Since we do not have prior information about the relationship between observed transportation access and these specific predictors, we will be using the default prior in `stan_polr`. Specifically, we establish a prior for the location of the proportion of the outcome variance that is attributable to the predictors. This proportion is more commonly known as the `$R^2$` metric used in frequentist linear regressions and could be located at any value between (0,1). It is also worth noting that we could specify uniform Dirichlet count priors (e.g., `$\text{Dirichlet}(1,1,1)$`) on the categories of transportation access in `stan_polr`. 

Since we are using a uniform prior on the location of `$R^2$`, we say the ratio of our `$R^2$`'s proportion is around 0.5 or that a perfect explanation of variance and non-explanation of variance in our outcome categories is equally plausible. So, we expect that 50% of the variability in transportation access cannot be explained by the mean neighborhood income, percentage below the poverty line in a neighborhood, the number of grocery stores and food vendors in a neighborhood, and the borough. Visit the $R^2$ section in `rstanarm` 's documentation [here](https://rdrr.io/cran/rstanarm/man/priors.html) for a more technical overview. 

One technical limitation is the paucity of cross-validated error metrics for `stan_polr` 's ordinal regression model. To test the model's fitness on new data, we used a manual train-test split where our training data would include 70% of the original observations, while our test data would include the remaining 30%.


```{r}
ordinal_model <- stan_polr(transportation_desert_4num ~ mean_income + below_poverty_line_perc + store_count, 
                    data =data_train, prior_counts = dirichlet(1),
                    prior=R2(0.5), iter=500, seed = 86437, refresh=0, prior_PD=FALSE)
```

Here is an interpretation of the coefficients in the tidy table:


**Mean Income**: When controlling all other factors, for each additional dollar a given neighborhood's mean income has, the model estimates that the latent variable `$y^*$` increases by 0.0000197. However, there is an 80% chance that the increase in the latent variable $y^*$ may be any value between (0.0000010, 0.0000383), indicating that there is almost certainly a positive relationship between mean income and $y^*$, but its magnitude may vary.

**Percent Living Below the Poverty Line**: When controlling all other factors, for each additional % people below poverty a given neighborhood has, the model estimates that the latent variable $y^*$ increases by 0.0977415. However, there is an 80% chance that the increase in the latent variable $y^*$ may be any value between (0.0422522, 0.1576466), indicating that there is almost certainly a positive relationship between below poverty percentage and $y^*$, but its magnitude may vary.

`Grocery Store Count`: When controlling all other factors, for each additional food and grocery store a given neighborhood has, the model estimates that the latent variable `$y^*$` increases by 0.0224638. However, there is an 80% chance that the increase in the latent variable $y^*$ may be any value between (0.0138214, 0.0316698), indicating that there is almost certainly a positive relationship between the number of stores and $y^*$, but its magnitude may vary.

`Brooklyn`: When controlling all other factors, if a neighborhood is in Brooklyn, the model estimates that the latent variable `$y^*$` increases by 0.1084268 compared to that neighborhood in the Bronx. However, there is an 80% chance that the increase in the latent variable $y^*$ may be any value between (-0.6968746, 0.8669477), indicating the relationship between a neighborhood being in Brooklyn and $y^*$ could be insignificant.

`Manhattan`: When controlling all other factors, if a neighborhood is in Manhattan, the model estimates that the latent variable `$y^*$` increases by 2.5965602 compared to that neighborhood in the Bronx. However, there is an 80% chance that the increase in the latent variable $y^*$ may be any value between (1.2840073, 4.3013850), indicating that we are confident that the relationship between a neighborhood being in Manhattan and $y^*$ exists, but its magnitude may vary.

`Queens`: When controlling all other factors, if a neighborhood is in Queens, the model estimates that the latent variable $y^*$ increases by -0.4148264 compared to that neighborhood in the Bronx. However, there is an 80% chance that the increase in the latent variable $y^*$ may be any value between (-1.1315662, 0.3349786), indicating that we are not confident with the relationship between a neighborhood being in Queens and $y^*$.


Then, editing a function written by [Connie Zhang](https://connie-zhang.github.io/pet-adoption/modelling.html) into a tidy format, we compute the accuracy of the ordinal model below. Specifically, we take the most common category predicted across all of our model's simulations, then use that modal category as our final prediction.


```{r}
tidy_ordinal_accuracy <- function(testing_data, predictions){
  
  # write mode function since it does not exist in R
  mode <- function(x) {
                t <- table(x)
                names(t)[which.max(t)]
  }
  
  #write mode function since it does not exist in R
  mydata <- testing_data  %>%
    dplyr::select(`transportation_desert_3num`) %>% 
    rownames_to_column("observation") 
  
  # extract predictions
  # transpose so each row is a case from the training data, 
  # then compute rowwise modes of classifications to find the most common prediction
  raw_table <- predictions%>%
    t() %>%
    as_tibble()%>% 
    mutate_if(is.character, as.numeric) %>%
    rownames_to_column("observation") %>% 
    rowwise(id="observation") %>%
    mutate_if(is.numeric, as.factor) %>%
    summarize(modal_prediction = mode(c_across(where(is.factor))))  %>%
    right_join(mydata, ., by = "observation")

  # compute transportation category accuracies
  group_specific <- raw_table %>%
    mutate(Accurate = ifelse(modal_prediction == transportation_desert_3num, 1,0))%>%
    group_by(transportation_desert_3num) %>%
    summarise(Accuracy = mean(Accurate)) %>%
    mutate(transportation_desert_3num = case_when(
    transportation_desert_3num == 1 ~ "Poor",
    transportation_desert_3num == 2 ~ "Typical",
    TRUE ~ "Excellent"))  
 
   # compute overall accuracy
  overall <- raw_table %>%
    mutate(Accurate = ifelse(modal_prediction ==transportation_desert_3num, 1,0))%>%
    summarise(Accuracy = mean(Accurate)) %>%
    mutate(transportation_desert_3num = "Overall", .before=1)
  
  # bind into table
  rbind(group_specific, overall) %>%
    dplyr::rename("Access Level" = transportation_desert_3num) %>%
    kable(align = "c", caption = "Ordinal Model - Accuracy") %>% 
    kable_styling()
  
}
```


```{r}
set.seed(86437)

my_prediction2 <- posterior_predict(
  ordinal_model, 
  newdata = data_test)

tidy_ordinal_accuracy(data_test, my_prediction2)
```


| Access Level | Accuracy  |
| ------------ | --------- |
| Poor         | 0.571 |
| Limited      | 0.555 |
| Satisfactory | 0.000 |
| Excellent    | 1.000 |
| Overall      | 0.722|

It seems that our model is incredibly accurate when classifying neighborhoods as having excellent and poor subway access (90%; 71.4%), while almost uniformly incorrect when classifying a neighborhood as having typical subway access (9.09%). Our model is more likely to categorize a neighborhood with typical subway access as having excellent or poor access. Note that we could not accurately predict this "Typical" category in both the Ordinal and the naive model. 

In addition to accuracy within each group, we also consider the compositions of the actual vs. predictions. Below we visualize the same error metrics for the ordinal model.


<center>
<img src="/media/bayes/plot_4.png" width="100%" height="100%" />
</center>

Here is a graph of a detailed breakdown of proportions within each observed category. Once again, observe that our predictions are always incorrect for the Typical category. Likely, this must be a byproduct of our interval criteria for each respective level of transportation inaccessibility— that is, these categories may not be sufficiently different to find differences. Additionally, the unequal distribution between the categories may be another potential determinant behind the inaccuracy of the Typical category classifications.

In further refinements of the project, we would like to tweak the cutoffs to strike a balance between the equal distribution and the model accuracy (i.e., we want to make sure all three categories can be predicted with a similar accuracy).

If you would like to check out what happens as we vary our predictors in this ordinal model, we created a shiny app that allows users to vary the predictors manually. Click this [link](https://vichym.shinyapps.io/nyc_subway_desert_prediction/) to check it out!

The following section explores how transportation accessibility affects urban housing inequities.


# Transportation and Structural Inequity

Transportation access is a pervasive structural issue. However, previous research on transportation access has demonstrated that many of these gaps in access also deepen the chasms of racial and class-based inequity. In this analysis, it seemed that low-income and racialized— typically Black and Latinx— communities were most likely to confront [transportation inequities](https://nyc.streetsblog.org/2021/06/18/report-racial-and-economic-inequities-in-transit-affect-accessibility-to-jobs-healthcare/). Additionally, we have observed that predominantly Black and Latinx neighborhoods in the Bronx faced some of the highest rates of eviction, likely indicating that these inequities may overlap.

This next section aims to connect transportation access to housing-inequities that we know also have racial and class dimensions. In particular, we wanted to assess transportation access's relationship with immigrant community size, rental prices, and eviction counts by neighborhood. 

<center>
<img src="/media/bayes/plot_5.png" width="100%" height="100%" />
</center>

Transportation access is typically worst in areas with the highest densities of non-citizen residents, while it is the best in neighborhoods with the highest mean rental prices. Further, observe that eviction counts are highly concentrated in north and south NYC, where some neighborhoods have mixed accessibility to transit. 

Unfortunately, rent, transportation access, and eviction counts may also be associated with the respective density of nonwhite communities.

```{r, fig.height=16, fig.width=16+4}
ggarrange(desert_map, white, black, latinx, asian, ncol=3, nrow=2)
```

<center>
<img src="/media/bayes/plot_6.png" width="100%" height="100%" />
</center>

From the plots, neighborhoods with the highest densities of Black and Asian community members also have the poorest scores of subway access. In contrast, the converse holds for neighborhoods with the highest proportion of White residents. 

Further, observe that eviction counts are most common in neighborhoods with the highest densities of Black and Latinx community members. The below visualizations detail the specific relationships between the proportion of Black, Latinx, Asian, and White residents in a neighborhood with eviction counts.

<center>
<img src="/media/bayes/plot_7.png" width="100%" height="100%" />
</center>

Neighborhood Black and Latinx residential proportions seem to be associated with increased eviction counts. However, we must specify that Black resident proportions are uniformly associated with increases in eviction counts across transportation levels, while the relationship between eviction counts and Latinx resident proportion is not. In contrast, both White and Asian proportions are uniformly associated with decreases in eviction counts.

Lastly, let us consider mean neighborhood rental prices. The following visualizations detail the relationships between the proportion of Black, Latinx, Asian, and White residents in a neighborhood with the mean rental price.


<center>
<img src="/media/bayes/plot_8.png" width="100%" height="100%" />
</center>

Increases in White-resident proportions were uniformly associated with increases in average neighborhood rental prices across all transportation access categories. The converse is valid for the relationship between Black and Latinx neighborhood densities and rental prices. It then seems that despite living in cheaper neighborhoods, both NYC's Black and Latinx communities are carrying the most significant burden of eviction. 

Our next section details the statistical models we used to better characterize the precise relationships between housing inequities, urban racism, and transportation access.

## Non-Hierarchical Models



In order to understand the respective distributions of immigrant population size, evictions, and mean rental prices in NYC, we fit three non-hierarchical Bayesian models with each variable as an outcome. 

We list our non-hierarchical models and their predictors below:


- Non-Citizen Count Model (1)		
  + Predictors: `transportation_desert_4cat`, `borough`, `total_pop`, `gini_neighborhood`, `mean_income`, `mean_rent`, `unemployment_perc`, `black_perc`, `latinx_perc`, `asian_perc`
  + Symbolic Equivalents: `$x_1, x_2, ..., x_{14}$`

- Average Rental Price Model (2)
  + Predictors: `transportation_desert_4cat`, `borough`, `gini_neighborhood`, `mean_income`, `black_perc`, `latinx_perc`, `asian_perc`, `below_poverty_line_perc`,`school_count`,`store_count`
  + Symbolic Equivalents: `$x_1, x_2, ..., x_{14}$`
  
  
- Eviction Count Model (3)
  + Predictors: `transportation_desert_4cat`, `borough`, `total_pop`,
  `below_poverty_line_perc`, `gini_neighborhood`, `mean_income`, `mean_rent`, 
  `black_perc`, `latinx_perc`, `asian_perc`
  + Symbolic Equivalents: `$x_1, x_2, ..., x_{14}$`
  

Because there are four levels of both `transportation_desert_3cat` and `borough`, `stan_glm` defines  `$x_1, \dots, x_3$` and `$x_5, \dots, x_7$` as dummy variables of each respective predictor's categories, with one common reference category for our intercept.
  
Across all models, we specified weakly-informative normal priors for the `$\beta_{k} $`'s associated with each predictor `$X_{k}$`. However, there are differences in terms of model specifications that we outline below:

For 1 and 3, we used weakly informative normal priors on all predictors and the intercept, then allowed `stan_glm` to estimate initial priors. This decision was ultimately due to our unfamiliarity with NYC's history of evictions, non-citizen population, and their respective relationships to our predictors.

We fit parallels of 1 and 3 that both had a Poisson likelihood instead of a Negative Binomial likelihood in previous iterations of this report. However, we observed an inconsistent spread of variance and increased 0 counts in both the eviction and immigrant count data. Because `stan_glm` does not have a zero-inflated Poisson distribution, we ultimately performed two Negative-Binomial regressions. We removed further discussions of our Poisson regressions from this project.

For 2, we also used weakly informative normal priors on all predictors. However, we specified a normal prior with `$\mu = 600$` and `$\sigma = 20$` for the scaled intercept of mean rental prices. 

Our model specifications are detailed in the following subsections.

### Model 1: Immigrant/Non-Citizen Count

`$$
\begin{split}
\text{Non-Citizen Count}_{i} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu_i, r) \; \; \; \text{where } \log(\mu_i) = \beta_{0c} + \sum^{14}_{k=1} \beta_{k}X_{ik} \\
\beta_{0c} &\sim N(0,2.5^2)\\	
r &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

where

`$$
\begin{align}
s_k \in \{
&5.5004,				
7.9328,				
4.9972,				
5.4697,				
6.443,				
5.3347,				
.0001,	\\			
&56.9002,				
0.0074,				
0.5699,
39.9937,				
0.993,				
1.142,				
1.6003
\}
\end{align}
$$`


We fit our model below using `stan_glm`.

```{r}
noncit_model <- stan_glm(
  noncitizen_count ~
    transportation_desert_4cat + 
    borough + total_pop + gini_neighborhood +mean_income + mean_rent + unemployment_perc + 
    black_perc + latinx_perc + asian_perc,
  data = modeling_data,
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

Next, we assess its distributional fit to our data.

<center>
<img src="/media/bayes/plot_9.png" width="100%" height="100%" />
</center>

It seems that our simulations (light green) of non-citizen count distributions were broadly consistent across our iterations. However, our simulations were strongly biased from the observed non-citizen counts, where we would typically predict smaller non-citizen counts more frequently. Additionally, it seemed that variance between simulations increased as non-citizen counts increased. Acknowledging these simulation trends, our negative-binomial model seems to be a pretty good distributional fit but could undoubtedly be improved upon! 


### Model 2: Mean Neighborhood Rental Prices


`$$
\begin{split}
\text{Mean Rental Price}_{i} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim N(\mu_i, \sigma^2) \; \; \; \; \text{where } \mu_i = \beta_{0c} + \sum^{14}_{k=1} \beta_{k}X_{ik} \\
\beta_{0c} &\sim N(1600,20^2)\\	
\sigma &\sim \text{Exp}(0.23)\\
\beta_{k} &\sim N(0,s_k^2)
\end{split}
$$`

where

`$$
\begin{align}
s_k \in \{
&24.1301				
34.8009				
21.9225				
23.9953				
28.2651				
23.4030				
249.6185		\\			
&		
0.0325	
4.3562				
5.0098	
7.0204				
110.1265				
1.7160				
0.2803	
\}
\end{align}
$$`

Once again, note that we are specifying the parameters of our scaled intercept prior, `$\beta_{0c}$`, so that the typical mean neighborhood rental price `$\sim $N(1600,20^2)$`. In contrast, our priors for the `$\beta_k$` are weakly-informed negative priors. We chose our prior intercept specifications of the mean rental price (`$\beta_{0c}$`) using Juthi's experience renting in NYC and a group conversation about typical rental prices we would elect to pay in NYC, Los Angeles, and other major cities. However, we decided to continue using weakly-informative normal priors for the predictors because we were unsure about their relationship— if any— to rental prices.

We fit our model below.

```{r}
rent_model <- stan_glm(
  mean_rent  ~ 
  transportation_desert_4cat + borough + gini_neighborhood +
    mean_income + black_perc + latinx_perc + asian_perc+
    below_poverty_line_perc + school_count + store_count,
  data = modeling_data, 
  family = gaussian,
  prior_intercept = normal(1600 , 20),
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

Next, we assess its distributional fit to our data.

<center>
<img src="/media/bayes/plot_10.png" width="100%" height="100%" />
</center>

Our simulations are relatively consistent. However, these simulated distributions are much more variable than the simulated distributions we saw in our non-citizen rent model. Importantly, it also seems our simulated normal posterior distributions had higher variance than the observed data. That is likely because the mean rental prices themselves are not perfectly normally distributed. Theoretically, we could change our likelihood model to adjust for the skew, but for now, this is good!


### Model 3: Eviction Count


`$$
\begin{split}
\text{Eviction Count}_{i} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu_i, r) \; \; \; \text{where } \log(\mu_i) = \beta_{0c} + \sum^{14}_{k=1} \beta_{k}X_{ik} \\
\beta_{0c} &\sim N(0,2.5^2)\\	
r &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

where

`$$
\begin{align}
s_k \in \{
&5.5004,				
7.9328,				
4.9972,				
5.4697,				
6.443,				
5.3347,				
.0001, \\			
&25.1032,				
56.9002,				
0.0074,				
0.5699,				
0.993,				
1.142,				
1.6003
\}
\end{align}
$$`

We fit our model below.

```{r}
eviction_model <- stan_glm(
  eviction_count  ~ 
    transportation_desert_4cat + borough + total_pop +
    below_poverty_line_perc + gini_neighborhood + mean_income + mean_rent+ 
    black_perc + latinx_perc + asian_perc,
  data = modeling_data, 
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

Next, we assess its distributional fit to our data.

<center>
<img src="/media/bayes/plot_11.png" width="100%" height="100%" />
</center>


Our simulations of eviction count distributions are dramatically consistent across the iterations. Further, our simulations were relatively consistent with the observed eviction counts. So, our negative-binomial model seems to be a pretty good distributional fit but could be improved upon! 


## Hierarchical Models

Given the potential differences in demographic characteristics, housing trends, and transportation by borough and the clear hierarchy from borough to the neighborhood, we fit three hierarchical parallels of the non-hierarchical models above. Now, however, we let borough be a grouping variable in our data. Importantly, our priors for the `$\beta_k$`s have stayed the same. 

We list our hierarchical models and their predictors below:

- Non-Citizen Count Hierarchical Model (4)		
  + Predictors: `transportation_desert_4cat`, `total_pop`, `gini_neighborhood`, `mean_income`, `mean_rent`, `unemployment_perc`, `black_perc`, `latinx_perc`, `asian_perc`
  + Grouping: `borough`

- Average Rental Price Hierarchical Model (5)
  + Predictors: `transportation_desert_4cat`, `gini_neighborhood`, `mean_income`, `black_perc`, `latinx_perc`, `asian_perc`, `bus_count`,`school_count`,`store_count`, `noncitizen_perc`
  + Grouping: `borough`

- Eviction CountHierarchical Model (6)
  + Predictors: `transportation_desert_4cat`, `total_pop`, `gini_neighborhood`, `mean_income`, `mean_rent`, `black_perc`, `latinx_perc`, `asian_perc`,
  `bus_count`,`store_count`,`unemployment_perc`, `uninsured_perc`
  + Grouping: `borough`

Again for 4 and 6, we used weakly informative priors and allowed `stan_glm` to estimate initial priors. While for 5, we specified the same prior intercept as a normal distribution with a mean of 1600 and standard deviation at 20. 


### Model 4: Immigrant/Non-Citizen Count



`$$
\begin{split}
\text{Non-Citizen Count}_{ij} \mid  \beta_{0}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu_{ij}, r) \; \; \; \text{where } \log(\mu_{ij}) = \beta_{0j} + \sum^{11}_{k=1} \beta_{k}X_{ijk} \\
\beta_{0j}\mid \beta_{0}, \sigma_0 & \stackrel{ind}{\sim} N(\beta_0, \sigma_0^2)\\	
\beta_{0c} &\sim N(0, 2.5^2) \\
r &\sim \text{Exp}(1)\\
\sigma_0 &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

where

`$$
\begin{align}
s_k \in \{
&5.5004,				
7.9328,				
4.9972,				
.0004,				
56.9002,\\			
&	0.0074,				
0.5699,				
39.9937,				
0.993,				
1.142,
1.6003\}
\end{align}
$$` 

We fit our model using `stan_glmer` below.

```{r}
hi_noncit_model <- stan_glmer(
  noncitizen_count ~
    transportation_desert_4cat + 
    total_pop + gini_neighborhood + 
    mean_income + mean_rent +
    unemployment_perc + 
    black_perc + latinx_perc + asian_perc + (1|borough),
  data = modeling_data,
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

Next, we check its distributional fit to our data.

<center>
<img src="/media/bayes/plot_15.png" width="100%" height="100%" />
</center>

It seems that our simulations of non-citizen count distributions in the hierarchical model were consistent across the iterations. Like the non-hierarchical model, our simulations are biased away from the observed non-citizen counts as our model predicts a higher number of non-citizens more frequently. 


### Model 5: Mean Neighborhood Rental Prices


`$$
\begin{split}
\text{Mean Rental Price}_{ij} \mid  \beta_{0j}, \beta_1, \dots, \beta_k,\sigma_y & \sim N(\mu_{ij}, \sigma_y^2) \; \; \; \text{where } \mu_{ij} = \beta_{0j} + \sum^{11}_{k=1} \beta_{k}X_{ijk} \\
\beta_{0j} & \stackrel{ind}{\sim} N(\beta_0, \sigma_0^2)\\	
\beta_{0c} &\sim N(1600, 20^2) \\
\sigma_y  &\sim \text{Exp}(1)\\
\sigma_0 &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

where 

`$$
\begin{align}
s_k \in \{
&24.1301				
34.8009				
21.9225				
249.6185				
0.0325
4.3562		\\			
& 5.0098				
7.0204				
110.1265				
1.7160	
0.2803
\}
\end{align}
$$`


Once again, we chose the same prior specifications of the mean rental price intercept `$\beta_{0c}$` as our previous models.

We fit our model using `stan_glmer` below.

```{r}
hi_rent_model <- stan_glmer(
  mean_rent  ~ 
    transportation_desert_4cat + gini_neighborhood +
    mean_income + black_perc + latinx_perc + asian_perc+
    below_poverty_line_perc + school_count + store_count + (1 | borough),
  data = modeling_data, 
  family = gaussian,
  prior_intercept = normal(1600, 20, autoscale = TRUE),
  prior = normal(0, 2.5, autoscale = TRUE), 
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

Next, we check its distributional fit to our data.

<center>
<img src="/media/bayes/plot_16.png" width="100%" height="100%" />
</center>

Our simulations are relatively consistent. However, these simulated distributions are much more variable than the simulated distributions we saw in our non-citizen rent model. Importantly, it also seems our simulated normal posterior distributions had higher variance than the observed data, just like the non-hierarchical model.


### Model 6: Eviction Count

`$$
\begin{split}
\text{Eviction Count}_{ij} \mid  \beta_{0}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu_{ij}, r) \; \; \; \text{where } \log(\mu_{ij}) = \beta_{0j} + \sum^{11}_{k=1} \beta_{k}X_{ijk} \\
\beta_{0j}\mid \beta_{0}, \sigma_0 & \stackrel{ind}{\sim} N(\beta_0, \sigma_0^2)\\	
\beta_{0c} &\sim N(0, 2.5^2) \\
r &\sim \text{Exp}(1)\\
\sigma_0 &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

where

`$$
\begin{align}
s_k \in \{
&5.5004,				
7.9328,				
4.9972,				
0.0001,				
25.1032,	56.9002,		\\			
&	0.0074,				
0.5699,				
0.9930,				
1.1420,
1.6003
\}
\end{align}
$$`

We fit our model using `stan_glmer` below.

```{r}
hi_eviction_model <- stan_glmer(
  eviction_count  ~ 
    transportation_desert_4cat +
    total_pop + below_poverty_line_perc+
    gini_neighborhood + mean_income +  mean_rent + 
    black_perc + latinx_perc + asian_perc + (1|borough),
  data = modeling_data, 
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

Next, we check its distributional fit to our data.

<center>
<img src="/media/bayes/plot_17.png" width="100%" height="100%" />
</center>


Our simulations of eviction count distributions are dramatically consistent across the iterations. Further, our simulations were relatively consistent with the observed eviction counts. Our negative-binomial model seems to be a pretty good distributional fit but could definitely be improved upon! 

In the following sections, we go through each model's outcome and what they tell us about the relationships between transportation and housing.


## Model Performance

### Immigrant/Non-Citizen Count

<center>

| model            | mae      | mae\_scaled | within\_50 | within\_95 |
| ---------------- | -------- | ----------- | ---------- | ---------- |
| Non-Hierarchical | 1026.946 | 0.5670020   | 0.5666667  | 0.9666667  |
| Hierarchical     | 1036.095 | 0.5555193   | 0.5555556  | 0.9666667  |

</center>


Using in-sample scaled MAE, it is evident that the differences between our two negative binomial models of non-citizen counts were minute. We would typically recommend using cross-validated error metrics to compare these models in other settings. However, because we intend to compare a hierarchical model to a non-hierarchical one, cross-validation works slightly differently. In the former case, cross-validation via the `prediction_summary_cv` function in `bayesrules` makes data permutations to represent a "new borough" rather than a set of arbitrary models. To circumvent these differences, we use the expected-log predictive density (ELPD) of each respective model to compare the performance of each model. The details of this approach can be found [here](https://www.bayesrulesbook.com/chapter-10.html#chapter-10-train-test).


<center>

| model             | elpd\_diff  | se\_diff  | elpd\_loo  | se\_elpd\_loo | p\_loo   | se\_p\_loo | looic    | se\_looic |
| ----------------- | ----------- | --------- | ---------- | ------------- | -------- | ---------- | -------- | --------- |
| hi\_noncit\_model | 0.0000000   | 0.0000000 | \-1607.625 | 11.46815      | 13.50260 | 2.306438   | 3215.250 | 22.93629  |
| noncit\_model     | \-0.9888152 | 0.7546653 | \-1608.614 | 11.61547      | 14.85341 | 2.514718   | 3217.227 | 23.23093  |

</center>



From the above ELPD rankings, it seems that the non-hierarchical model of non-citizen counts performed better than its hierarchical parallel. Explicitly, there was a decrease of .536 standard deviations when comparing the non-hierarchical to its hierarchical parallel.


<center>
<img src="/media/bayes/residual_scatter_noncit.png" width="100%" height="100%" />
</center>

Our residuals are not perfect, given the slight heteroskedasticity as non-citizen count increased, but it seems that the distributions were almost perfectly comparable between the two models!


<center>
<img src="/media/bayes/plot_21.png" width="100%" height="100%" />
</center>


The residuals on our hierarchical model are not perfectly randomly distributed across all neighborhoods in NYC, but the scale of these residuals is much smaller than the non-hierarchical model. One could also make the case that there are slight over predictions (darker blue) of non-citizen counts in Brooklyn (bottom left) for the non-hierarchical model. However, these differences are mostly negligible.

<center>

| Model            | WAIC     | SE       |
| ---------------- | -------- | -------- |
| Non-Hierarchical | 3208.463 | 23.15916 |
| Hierarchical     | 3206.880 | 23.22938 |

</center>


Our last error metric of non-citizen counts is the Watanabe–Akaike information criterion (WAIC). When comparing two models with the WAIC, the better predicting model would have a smaller WAIC value. Here, it seems that non-hierarchical model of non-citizen counts has the lowest WAIC, but the difference is marginal. Altogether, we choose the hierarchical model of non-citizen counts given its lower ELPD, WAIC, and spatial residual patterning.

## Mean Neighborhood Rental Prices

<center>

| model            | mae       | mae\_scaled | within\_50 | within\_95 |
| ---------------- | --------- | ----------- | ---------- | ---------- |
| Non-Hierarchical | 0.8377195 | 0.5259208   | 0.6055556  | 0.9500000  |
| Hierarchical     | 0.8024302 | 0.5087840   | 0.5888889  | 0.9444444  |

</center>

Once again, when using in-sample scaled MAE, the differences between our two models of mean rental prices were essentially minimal. Our predictions of the typical neighborhood rental price in both models were about 80 dollars away from their observed rental prices. However, the hierarchical model's scaled mean absolute error metrics indicate that across simulations, the observed value is about 0.3 standard deviations away from the medians of the posterior predictive distributions of rental prices. This is a nontrivial improvement from the .5 standard deviation in the case of the non-hierarchical distribution. 

<center>

| model           | elpd\_diff | se\_diff  | elpd\_loo  | se\_elpd\_loo | p\_loo   | se\_p\_loo | looic    | se\_looic |
| --------------- | ---------- | --------- | ---------- | ------------- | -------- | ---------- | -------- | --------- |
| hi\_rent\_model | 0.000000   | 0.0000000 | \-341.3198 | 15.17132      | 17.47628 | 3.915789   | 682.6395 | 30.34264  |
| rent\_model     | \-1.542781 | 0.6447002 | \-342.8626 | 14.91792      | 18.60953 | 3.799422   | 685.7251 | 29.83584  |

</center>

From the above ELPD rankings, it seems that the hierarchical model of mean neighborhood rental prices performed better than its non-hierarchical parallel.


<center>
<img src="/media/bayes/residual_scatter_rent.png" width="100%" height="100%" />
</center>

Our residuals look pretty good, but there is some increasing trend that we are underpredicting the rental prices as they increase. However, it seems that the residual distributions are almost perfectly comparable between the two models!


<center>
<img src="/media/bayes/plot_21.png" width="100%" height="100%" />
</center>


Regarding the non-hierarchical model, the distribution of our residuals indicates that our model's mispredictions are randomly distributed. Further, our residuals in the non-hierarchical model are relatively small compared to the hierarchical model. In the hierarchical model, there is a slight indication of systematic mispredictions that appear distributed by borough. Crucially, it seems that we are systematically overpredicting rental prices in both Queens (bottom right) and Brooklyn (bottom left), but with more over-predictions occurring for Brooklyn. 

<center>

| Model            | WAIC     | SE       |
| ---------------- | -------- | -------- |
| Non-Hierarchical | 684.9805 | 29.67027 |
| Hierarchical     | 680.4569 | 29.84038 |

</center>

The hierarchical model of rental prices has the lowest WAIC, but the difference is negligible. The hierarchical model's residual patterning, ELPD metric, in-sample residual errors, and WAIC indicate that the hierarchical version may be more appropriate.


## Eviction Count

<center>

| model            | mae      | mae\_scaled | within\_50 | within\_95 |
| ---------------- | -------- | ----------- | ---------- | ---------- |
| Non-Hierarchical | 42.28500 | 0.5461398   | 0.5777778  | 0.9555556  |
| Hierarchical     | 43.08787 | 0.5467191   | 0.5777778  | 0.955555   |

</center>

Once again, when using in-sample scaled MAE, the differences between our two negative-binomial models of evictions count were minute. Our predictions of eviction counts in both models were about 40 cases away from their observed counts and with very similar scaled MAE metrics. 

<center>

| model               | elpd\_diff | se\_diff  | elpd\_loo  | se\_elpd\_loo | p\_loo   | se\_p\_loo | looic    | se\_looic |
| ------------------- | ---------- | --------- | ---------- | ------------- | -------- | ---------- | -------- | --------- |
| eviction\_model     | 0.000000   | 0.0000000 | \-1054.382 | 13.52879      | 16.69892 | 2.592070   | 2108.764 | 27.05759  |
| hi\_eviction\_model | \-0.038482 | 0.5198728 | \-1054.420 | 13.54050      | 16.34022 | 2.549446   | 2108.841 | 27.08100  |

</center>


From the above ELPD rankings, it also seems that the hierarchical model of eviction counts performed better than its non-hierarchical parallel. Next, we will compare how these two models' residuals are spatially distributed.

<center>
<img src="/media/bayes/residual_scatter_evictions.png" width="100%" height="100%" />
</center>

Our residuals look fantastic! There is a slight heteroskedastic trend in our predictions, but for the most part, it seems our residuals on eviction count predictions for both models are normally distributed around 0. Once again, it seems that the residual distributions are comparable. However, in our hierarchical model (red), we tend to underpredict eviction counts slightly more in extreme cases. Given that the hierarchical model will naturally shrink predictions of extreme cases, we expected this.


<center>
<img src="/media/bayes/plot_22.png" width="100%" height="100%" />
</center>

Our residuals are randomly distributed across all neighborhoods, indicating that our model's bias is consistent across observations— this is good! One could make the case that slight under predictions (darker red) of eviction counts occur more in the Bronx (top) for both models. However, these patterns are relatively minute. 

<center>

| Model            | WAIC     | SE       |
| ---------------- | -------- | -------- |
| Non-Hierarchical | 2103.173 | 27.53465 |
| Hierarchical     | 2103.144 | 27.38933 |

</center>

It seems that the hierarchical model of eviction counts has the lowest WAIC, but only by .03 units. However, we select the hierarchical model of eviction counts from the hierarchical model's improved residual patterning, ELPD metric, in-sample error metrics, and WAIC.

# Results \& Discussion

## Results

Our previous section consistently demonstrated that all 3 of our hierarchical models performed better than our non-hierarchical models with respect to absolute error metrics, residual distributions, expected-log predictive densities, and WAICs. In this section, we detail our findings using the hierarchical models. In this section, we emphasize the broader conclusions of our models and only report the nature of the associations between our outcomes and predictors (e.g., positive or negative). For individualized interpretations of each predictor for each model, please see the ["Full Interpretations" section in the Appendix](https://freddybarragan.netlify.app/media/bayes/bayes_final.html#Appendix)!

### Model 4: Immigrant/Non-Citizen Count

After removing predictors whose 80% credible intervals included the possibility of non-effect when controlling for other covariates, there were seven significant predictors of an arbitrary neighborhood's non-citizen counts. When considering the random-effects of borough and controlling for relevant covariates, non-citizen counts were positively associated with better subway access (Typical and Excellent), total population counts, mean rental prices, and the percentages of Black, Latinx, \& Asian people. At the same time, mean neighborhood income was negatively associated with non-citizen counts. The following table lists the specific $\beta$ values (labeled as "estimate") for each predictor, ordered by their association.


| term                                     | estimate    | std.error | conf.low    | conf.high    |
| ---------------------------------------- | ----------- | --------- | ----------- | ------------ |
| (Intercept)                              | 655.1704959 | 0.4859082 | 355.3384761 | 1259.5178318 |
| transportation\_desert\_4catLimited      | 23.8215261  | 0.0871758 | 10.9223819  | 39.3037996   |
| transportation\_desert\_4catSatisfactory | 28.7781449  | 0.1154065 | 11.0213880  | 48.5799874   |
| transportation\_desert\_4catExcellent    | 31.4830994  | 0.0989881 | 15.4929332  | 49.2745095   |
| total\_pop                               | 0.0025815   | 0.0000017 | 0.0023574   | 0.0027864    |
| mean\_income                             | \-0.0801747 | 0.0002335 | \-0.1097123 | \-0.0498783  |
| mean\_rent                               | 6.5520771   | 0.0166215 | 4.1387631   | 8.9194434    |
| black\_perc                              | 5.2653616   | 0.0170943 | 3.1311412   | 7.5478209    |
| latinx\_perc                             | 12.5891526  | 0.0224721 | 9.4244537   | 15.8286429   |
| asian\_perc                              | 19.9003911  | 0.0262381 | 15.9533952  | 23.8771062   |

Ultimately, this model demonstrates that immigrant hot-spots in New York City tend to form in economically disadvantaged neighborhoods with better subway access and larger nonwhite populations.



### Model 5: Mean Neighborhood Rental Prices

We observed that when considering the random-effects of borough and controlling for relevant covariates, mean rental prices were associated with five predictors. Specifically, we observed meaningful increases in mean rental prices when comparing neighborhoods with excellent subway access to neighborhoods with poor subway access. We also saw that neighborhood rental prices were positively associated with mean neighborhood income and the proportion of Asian residents in a neighborhood. Additionally, we found that mean rental prices were negatively associated with the proportion of a neighborhood's Black community and the number of schools. The following table details the specific `$\beta$` values (labeled as estimate) for each predictor, ordered by their association.


```{r}
tidy(hi_rent_model, effects = "fixed", conf.int = TRUE, conf.level = 0.8)%>% 
  filter(conf.low	> 0 & conf.high > 0 | conf.low	< 0 & conf.high < 0) %>%
  kable(align = "c", caption = "Hierarchical Rent Model - Summary") %>% 
  kable_styling()
```

| term          | estimate    | std.error | conf.low    | conf.high   |
| ------------- | ----------- | --------- | ----------- | ----------- |
| (Intercept)   | 7.5780558   | 2.0393369 | 5.0030700   | 10.1066203  |
| mean\_income  | 0.0122831   | 0.0007915 | 0.0112633   | 0.0132664   |
| asian\_perc   | 0.2416356   | 0.1144681 | 0.0976769   | 0.3856322   |
| school\_count | \-0.0398565 | 0.0267881 | \-0.0739144 | \-0.0056472 |

Our results indicate that economically privileged neighborhoods with the best subway access tend to have the highest mean rental prices and lower proportions of Black residents. This conclusion is intuitively true, given that housing prices are typically determined in response to the economic composition of renters. There is a cyclical relationship between rental prices and economic wealth where wealthy renters will select neighborhoods, and rental prices in wealthy neighborhoods will increase. Moreover, given NYC's longstanding [history of redlining](https://storymaps.arcgis.com/stories/4f06e78ca6964a04b968d9b5781499ae) and the dramatic inequities between Black and White people in the United States, it is, of course, true that these high rental prices areas will have lower proportions of Black residents.

We were surprised to find that the proportion of Asian residents in a neighborhood was positively associated with mean rental prices. This relationship indicates that Asian residents in NYC may opt to live in neighborhoods with higher rental prices or that Asian residents may be relegated to areas with very high rental prices. Both possibilities can be true given the vast diversity of Asian residents' economic privilege and the dramatic differences between Asian ethnic communities in NYC. [For example, Chinese-Americans in NYC are the second largest ethnic group and have established multiple rooting neighborhoods— "Chinatowns"— throughout NYC since the 1880s; the first of the nine was the Manhattan Chinatown](https://bt.barnard.edu/nycgildedages/neighborhoods/chinatown/). After a century, these neighborhoods have established internal business networks and accumulated sufficient economic wealth that the experience of Chinese-American neighborhoods in NYC may not be comparable to other Asian ethnic enclaves that have been newly formed. In contrast, [prior work](https://www.aafederation.org/wp-content/uploads/2019/08/WorkingButPoor.pdf) has shown that Asian-American poverty rates in NYC are some of the highest in the country. Together it is then likely that our specific posterior of the relationship between rental prices and Asian resident density has been biased by both the omission of specific Asian-ethnic data and the aggregated nature of our economic data.

Lastly, we found that mean rental prices were negatively associated with the number of schools in NYC, which is likely a byproduct of how excessively urban, well-connected areas in NYC do not typically have families with children living in them.



### Model 6: Eviction Count

We determined that many racial inequities associated with rental prices were replicated in the number of eviction counts by neighborhood. 

Specifically, we found that even when considering the random-effects of borough and controlling for our covariates, neighborhood eviction counts were positively associated with income inequality, better subway access (Typical and Excellent), the percentages of Black \& Latinx residents, mean rental prices, and total neighborhood population count. In contrast, our models suggest that eviction counts were negatively associated with mean income and the percent of Asian residents.



```{r}
tidy(hi_eviction_model, effects = "fixed", conf.int = TRUE, conf.level = 0.8)%>% 
  
  mutate(estimate= ifelse(term == "(Intercept)", exp(estimate), (exp(estimate)-1)*100), 
         conf.low= ifelse(term == "(Intercept)", exp(conf.low), (exp(conf.low)-1)*100), 
         conf.high = ifelse(term == "(Intercept)", exp(conf.high), (exp(conf.high)-1)*100))%>%
  filter(conf.low	> 0 & conf.high > 0 | conf.low	< 0 & conf.high < 0) %>%
  kable(align = "c", caption = "Eviction Model - Summary") %>% 
  kable_styling()
```

| (Intercept)                              | 13.8092973   | 0.6874845 | 5.6333236   | 33.3451782    |
| ---------------------------------------- | ------------ | --------- | ----------- | ------------- |
| transportation\_desert\_4catLimited      | 40.7106141   | 0.1040043 | 23.8780090  | 62.4039496    |
| transportation\_desert\_4catSatisfactory | 43.2081471   | 0.1274771 | 20.8712188  | 69.7214014    |
| transportation\_desert\_4catExcellent    | 41.6204049   | 0.1220271 | 21.8224202  | 64.8944937    |
| total\_pop                               | 0.0022122    | 0.0000018 | 0.0019792   | 0.0024542     |
| gini\_neighborhood                       | 1754.6596346 | 1.3691202 | 230.8933884 | 10463.7578459 |
| mean\_income                             | \-0.1278395  | 0.0003281 | \-0.1689573 | \-0.0867512   |
| mean\_rent                               | 4.5296754    | 0.0192151 | 1.9621581   | 7.1989234     |
| black\_perc                              | 18.3931979   | 0.0196683 | 15.4720498  | 21.3857255    |
| latinx\_perc                             | 8.6003316    | 0.0315746 | 4.1163943   | 13.0608650    |
| asian\_perc                              | \-4.0231070  | 0.0305908 | \-7.6692201 | \-0.0645505   |


We have confirmed that even after adjusting for the putative factors of economic instability and evictions (e.g., income, rental prices, and population size), neighborhoods with a substantial proportion of Black and Latinx residents in NYC are experiencing higher counts of eviction, despite tending to live in cheaper neighborhoods. Specifically, 10% increases in the percent of Black and Latinx residents are associated with approximately 17% and 8% increases from the typical eviction count across NYC. Once again, given the enforced precarity and exploitation of Black and Latinx communities in the United States, it is no surprise that eviction counts are associated with a neighborhood's racial composition.

We were also surprised to find that the proportion of Asian residents in a neighborhood was negatively associated with eviction counts. This trend may be another byproduct of the economic stability and developed housing markets in specific Asian ethnic communities. Our results also affirm that within-neighborhood income inequalities can be critical factors in defining eviction counts, suggesting that racist housing policies alone do not sufficiently explain the housing inequities observed in NYC.

Interestingly, when accounting for the above economic and demographic predictors, we were surprised to see that eviction counts were lower in neighborhoods with poor subway access when compared to neighborhoods with typical and excellent access. We felt that this could be a consequence of gentrification or the austere renting policies set by landlords in neighborhoods with intense competition. However, [one analysis](https://www.nlihc.org/sites/default/files/HPD_Transit_Displacement_Eviction.pdf) aimed to explore transit-induced displacement hypothesis using eviction rates suggested that there did not exist a statistically significant relationship between eviction rates and newly-constructed transit neighborhoods. However, our results suggest that there *is* a connection between increased subway access and eviction counts. It is plausible that this difference may be attributable to the historical preconditions of [redlining in NYC](https://storymaps.arcgis.com/stories/4f06e78ca6964a04b968d9b5781499ae); the fact that many of NYC's neighborhoods have historically been connected to the subway system, so they are not "newly transit-accessible" like those studied in Delmelle et al.'s analysis; or that our current model specifications may be inappropriate for our data. 



## Discussion

Across our models, it becomes clear that many of the structural housing and demographic issues present in NYC need to be more rigorously addressed by both policy-makers and its citizens, regardless of these particular models' performance. Health begins at home. Moreover, if NYC's Black and Latinx residents are being crushed under the fist of inequity and consequently experiencing increased risks of eviction or tenuous rental prices, then it becomes a health imperative to critically and revolutionarily address NYC's housing system. 

Although much work went into designing models with causal blocking and confounding in mind, our models remain imperfect. Some significant limitations involved the encoding of transportation deserts and the presence of unmeasured confounders. We cannot be entirely confident about our models ' conclusions because we may not have accounted for all structural predictors in housing equity, such as a neighborhood's rent control policies, nor have we adjusted for the specific reasons behind each observed eviction.

Importantly, we found statistically significant spatial clustering of non-citizen counts, eviction counts, and mean rental prices using Moran's I from the `spdep` package, which indicates that the spatial relationships between neighborhoods may have also occurred biased our results. 



```{r}
library(spdep)
modeling_data <- modeling_data %>%
  st_as_sf()

col_sp <- as(modeling_data, "Spatial")
col_nb <- poly2nb(col_sp) # queen neighborhood
col_listw <- nb2listw(col_nb, style = "B") # listw version of the neighborhood
W <- nb2mat(col_nb, style = "B") # binary structure

evict_moran <- tidy(moran.mc(col_sp$eviction_count, listw = col_listw, nsim = 999, alternative = "greater")) %>%
  mutate(variable = "Eviction", .before=1)   

rent_moran <- tidy(moran.mc(col_sp$mean_rent, listw = col_listw, nsim = 999, alternative = "greater")) %>%
  mutate(variable = "Mean Rent", .before=1)   
  
rbind(rent_moran, evict_moran) %>%
  kable() %>%
  kable_styling()
  
```
<center>

| Term      | statistic | p.value | parameter | method                            | alternative |
| --------- | --------- | ------- | --------- | --------------------------------- | ----------- |
| Mean Rent | 0.6560430 | 0.001   | 1000      | Monte-Carlo simulation of Moran I | greater     |
| Eviction  | 0.5794612 | 0.001   | 1000      | Monte-Carlo simulation of Moran I | greater     |

</center>

As such, we could extend this work to the spatial domain using methods from the `CARBayes` or `INLA` packages. Following work by [Katie Jolly and Raven McKnight](https://www.ravenmcknight.com/post/carbayes-tutorial/), we have outlined a spatial workflow for both the eviction and rental price models using `CARBayes` in the [Appendix](file:///Users/freddy/Documents/GitHub/454/checkpoint/CH5.html#Appendix). However, because spatial models were beyond the scope of this project and class, we'd like to emphasize that in a more detailed analysis the models we outline in the Appendix would likely be adjusted to use different data distributions, spatial effect priors (e.g., BYM, Intrinsic CAR, etcetera), and different predictors' coefficient priors. Further, in a more detailed analysis, we would explicitly describe the mathematical construction of the models.