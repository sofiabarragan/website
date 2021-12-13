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

# Motivation

In collaboration with Juthi Dewan, Sam Ding, and Vichy Meas, we designed this project for our Bayesian Statistics course taught by [Dr. Alicia Johnson](https://ajohns24.github.io/portfolio/). We'd like to extend our thanks to Alicia for guiding us through Bayes and the capstone experience!

A reproducible version of this blog post with all code can be found [here](https://freddybarragan.netlify.app/media/bayes/CH5.html).

We were initially interested in characterizing New York City’s internal racial dynamics using demography, geographic mobility, community health, and economic outcomes. As this project developed, we found ourselves thinking about the relationships between transportation (in)access and housing inequity. In our project, there are two major sections: **Subway Accessibility** and **Transportation and Structural Inequity**.  In **Subway Accessibility** we explore transportation deserts and what are the major determinants of Subway Inaccess in New York City using two Bayesian classification models. While in **Transportation and Structural Inequity**, we extend our discussion of transportation access to study its relationship to both rental prices and evictions using both simple and hierarchical Bayesian multivariate regression. In the [extended document](https://freddybarragan.netlify.app/media/bayes/CH5.html), we also fit non-hierarchical spatial models to control for the underlying spatial relationships between neighborhoods, but omit major discussion in this blog post as Bayesian spatial regression was beyond the scope of this course.

First, however, let's do a data introduction:




# Data Introduction


All data used in this project are from two major sources: the Tidycensus package and NYC Open Data. 

[Tidycensus](https://walker-data.com/tidycensus/index.html) is an R package interface, developed by Kyle Walker and Matt Herman, that enables easy access to the US Census Bureau’s data APIs and returns Tidyverse-ready data frames from various major US Census Bureau datasets. Our demographic and socioeconomic data are drawn from the 2019 American Community Survey results found in Tidycensus package. A summary of our ACS data variables is below:

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

- Compute total sums  for every count-based measurement.
- Compute mean average estimates  for remaining non-count predictors.

Then, for count based demographic predictors, we divide by the total population in each neighborhood to define scaled demographic metrics. They are as follows:

- `asian_perc`: Percentage of Asian People 
- `white_perc`: Percentage of White People 
- `black_perc`: Percentage of Black People 
- `latinx_perc`: Percentage of Latinx People 
- `native_perc`: Percentage of Native People 
- `noncitizen_perc`: Percentage of Foreign Born People  
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
- `transportation_desert_4cat`: Subway Accessibility  (Poor, Limited, Satisfactory, Excellent)

The process involved grouping geotagged locations by the defined neighborhood boundary regions in R’s `sf` package and ArcGIS. We will detail the process of identifying subway deserts in the "Subway Deserts" section. 

# Data Summaries 

We present a [numeric summary](https://freddybarragan.netlify.app/media/ch4.html#Data_Summaries) on our data with 224 observations of 38 variables. However, note that we use percent equivalents for most demographic count variables.

We found that Manhattan had the largest population counts, highest mean rental prices, highest mean income, highest income inequality (e.g. gini value), most number of neighborhoods with excellent subway access, and the greatest proportion of white citizens. 

Conversely, we observed that the Bronx has the lowest mean income and highest eviction counts, while having the highest proportions of people living below the poverty line, and the highest number of people with limited subway access. Importantly, the Bronx also had the highest densities of both Latinx and Black residents of any other borough in New York, meaning that the Black and Latinx residents in New York are experiencing the burden of New York's structural inequities.

We will touch on the connections between demographics, transportation access, and housing more explicitly in the following sections.

# Subway Accessibility

New York City is the most populous city in the US with [more than 8.8 million people](https://en.wikipedia.org/wiki/New_York_City). To support the daily commutes of its residents, NYC also built the New York City Subway, the oldest, longest, and currently busiest subway system in the US, averaging [approximately 5.6 million daily rides on weekdays and a combined 5.7 million rides each weekend](https://en.wikipedia.org/wiki/New_York_City_Subway). 

Compared to other US cities where automobiles are the most popular mode of transportation (ahem, Minneapolis), only 32% of NYC's population chooses to commute by cars. NYC's far-reaching transit system is then unique, given that [more than 70% of the population](https://en.wikipedia.org/wiki/Modal_share) commute by cars in other metropolitan areas.

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

| transportation\_desert\_4cat | Poor        | Limited     | Satisfactory | Excellent   |
| ---------------------------- | ----------- | ----------- | ------------ | ----------- |
| Poor                         | 78.12% (25) | 12.50% (4)  | 0.00% (0)    | 9.38% (3)   |
| Limited                      | 31.43% (11) | 11.43% (4)  | 20.00% (7)   | 37.14% (13) |
| Satisfactory                 | 20.69% (6)  | 17.24% (5)  | 6.90% (2)    | 55.17% (16) |
| Excellent                    | 9.52% (8)   | 15.48% (13) | 2.38% (2)    | 72.62% (61) |

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

| term                       | estimate   | std.error | conf.low   | conf.high  |
| -------------------------- | ---------- | --------- | ---------- | ---------- |
| mean\_income               | 0.0000460  | 0.0000104 | 0.0000339  | 0.0000596  |
| below\_poverty\_line\_perc | 17.0648982 | 3.4532734 | 12.8319749 | 22.3269332 |
| store\_count               | 0.0189996  | 0.0052394 | 0.0126227  | 0.0254789  |
| Poor | Limited             | 5.4113154  | 1.2620341 | 3.9385520  | 7.1398347  |
| Limited | Satisfactory     | 6.7324319  | 1.3225479 | 5.2493699  | 8.5003721  |
| Satisfactory | Excellent   | 7.7039327  | 1.3435146 | 6.1613497  | 9.5624926  |

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

This next section aims to connect transportation access to housing-inequities that we know also have racial and class dimensions. In particular, we wanted to assess transportation access’s relationship with immigrant community size, rental prices, and eviction counts by neighborhood. 

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

Neighborhood Black and Latinx residential proportions by seem to be associated with increased eviction counts. However, we must specify that Black resident proportions are uniformly associated with increases in eviction counts across transportation levels, while the relationship between eviction counts and Latinx resident proportion is not. In contrast, both White and Asian proportions are uniformly associated with decreases in eviction counts.

Lastly, let's consider mean neighborhood rental prices. The following visualizations detail the relationships between the proportion of Black, Latinx, Asian, and White residents in a neighborhood with that neighborhood mean rental price.


<center>
<img src="/media/bayes/plot_8.png" width="100%" height="100%" />
</center>

Increases in White-resident proportions were uniformly associated with increases in average neighborhood rental prices across all transportation access categories. The converse is true for the relationship between Black and Latinx neighborhood densities and rental prices. It then seems that despite living in cheaper neighborhoods, both NYC's Black and Latinx communities are carrying the largest burden of eviction. 

Our next section details the statistical models we used to better characterize the clear relationships between housing inequities, urban racism, and transportation access.

## Non-Hierarchical Models

In order to understand the respective distributions of immigrant population size, evictions, and mean rental prices in NYC, we fit 3 non-hierarchical Bayesian models with each variable as an outcome. In the following section, we fit hierarchical regression models with similar likelihood and prior structure. However, in those models neighborhood values are grouped by borough. 

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
  

Because there are four levels of both `transportation_desert_4cat` and `borough`, `stan_glm` defines  `$x_1, \dots, x_3$` and `$x_5, \dots, x_7$` as dummy variables of each respective predictor's categories, with one common reference category for our intercept.
  
Across all models we specified weakly-informative normal priors for the `$\beta_{k}$`s associated with each predictor $X_{k}$. However, there are differences in terms of model specifications that we outline below:

For 1 and 3, we used weakly informative normal priors on all predictors and the intercept, then allowed `stan_glm` to estimate initial priors. This decision was ultimately due to our unfamiliarity with NYC's history of evictions and non-citizen population and what their relationships to our predictors may be.

We fit parallels of 1 and 3 that both had a Poisson likelihood, as opposed to a Negative Binomial likelihood in previous iterations of this report. However, we observed an inconsistent spread of variance and increased 0 counts in both the eviction and immigrant count data. Because `stan_glm` does not have a zero-inflated poisson distribution, we ultimately performed two Negative-Binomial regressions. Further discussions of our Poisson regressions have been removed from this project.

For 2, we also used weakly informative normal priors on all predictors. However, we specified a normal prior with `$\mu = 600$` and `$\sigma = 20$` for the scaled intercept of mean rental prices. 

Our model specifications are detailed in the following subsections

### Model 1: Immigrant/Non-Citizen Count

```{r}
noncit_model <- stan_glm(
  noncitizen_count ~
    transportation_desert_4cat + 
    borough + total_pop + gini_neighborhood + 
    mean_income + mean_rent + unemployment_perc + 
    black_perc + latinx_perc + asian_perc,
  data = modeling_data,
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

`$$
\begin{split}
\text{Non-Citizen Count}_{i} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu_i, r) \; \; \; \text{where } \log(\mu_i) = \beta_{0c} + \sum^{14}_{k=1} \beta_{k}X_{ik} \\
\beta_{0c} &\sim N(0,2.5^2)\\	
r &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

`$\text{and where}$`

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

<center>
<img src="/media/bayes/plot_9.png" width="100%" height="100%" />
</center>

It seems that our simulations (light green) of non-citizen count distributions were largely consistent across our iterations. However, our simulations were strongly biased from the observed non-citizen counts where we would typically predict smaller non-citizen counts more frequently. Additionally, it seemed that variance between simulations increased as non-citizen counts increased. Acknowledging these trends, our negative-binomial model seems to be a pretty good distributional fit, but could certainly be improved upon! 


### Model 2: Mean Neighborhood Rental Prices

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

`$$
\begin{split}
\text{Mean Rental Price}_{i} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim N(\mu_i, \sigma^2) \; \; \; \; \text{where } \mu_i = \beta_{0c} + \sum^{14}_{k=1} \beta_{k}X_{ik} \\
\beta_{0c} &\sim N(1600,20^2)\\	
\sigma &\sim \text{Exp}(0.23)\\
\beta_{k} &\sim N(0,s_k^2)
\end{split}
$$`

`$\text{and where}$`

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

Once again note that we are specifying the parameters of our scaled intercept prior, `$\beta_{0c}$`, so that the the typical mean neighborhood rental price `$\sim $N(1600,20^2)$`, while our priors for the $\beta_k$ are weakly-informed negative priors. We chose our prior intercept specifications of mean rental price (`$\beta_{0c}$`) using Juthi’s experience renting in NYC and a group conversation about typical rental prices we would elect to pay in NYC, Los Angeles, and other major cities we have lived in or around. However, we decided to continue using weakly-informative normal priors for the predictors because we were unsure about their relationship— if any— to rental prices.

<center>
<img src="/media/bayes/plot_10.png" width="100%" height="100%" />
</center>

From the above posterior prediction check, it seems that our simulations are relatively consistent. However, these simulated distributions are much more variable than the simulated distributions we saw in our non-citizen rent model. Importantly, it also seems our simulated normal posterior distributions had higher variance than what was observed and that is likely because the mean rental prices themselves are not perfectly normally distributed. Theoretically, we could change our likelihood model to adjust for the skew, but for now this is good!


### Model 3: Eviction Count

```{r}
eviction_model <- stan_glm(
  eviction_count  ~ 
    transportation_desert_4cat + borough + total_pop +
    below_poverty_line_perc + gini_neighborhood + mean_income + mean_rent+ 
    black_perc + latinx_perc + asian_perc,
  data = modeling_data, 
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735,
)
```

`$$
\begin{split}
\text{Eviction Count}_{i} \mid  \beta_{0c}, \beta_1, ..., \beta_k, r & \sim \text{NegBin}(\mu_i, r) \; \; \; \text{where } \log(\mu_i) = \beta_{0c} + \sum^{14}_{k=1} \beta_{k}X_{ik} \\
\beta_{0c} &\sim N(0,2.5^2)\\	
r &\sim \text{Exp}(1)\\
\beta_{k} &\sim N(0,s_k^2) 
\end{split}
$$`

`$\text{and where}$`

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

<center>
<img src="/media/bayes/plot_11.png" width="100%" height="100%" />
</center>


Our simulations of eviction count distributions are dramatically consistent across the iterations. Further, our simulations were relatively consistent with the observed eviction counts. Our negative-binomial model seems to be a pretty good distributional fit, but could certainly be improved upon! 



## Hierarchical Models

Given the potential differences in demographic characteristics, housing trends, and transportation by borough and the clear hierarchy from borough to neighborhood, we fit 3 hierarchical parallels of the  non-hierarchical models above. Now, however, we let borough be a grouping variable in our data. Importantly, our priors for the $\beta_k$s have stayed the same. 

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

Again for 4 and 6, we used weakly informative priors and allowed `stan_glm` to estimate initial priors. While for 5, we specified the same prior intercept as a normal distribution with mean 1600 and standard deviation at 20. 

### Model 4: Immigrant/Non-Citizen Count

```{r}
hi_noncit_model <- stan_glmer(
  noncitizen_count ~
    transportation_desert_4cat + 
    total_pop + gini_neighborhood + 
    mean_income + mean_rent +
    unemployment_perc + 
    black_perc + latinx_perc + asian_perc + (1 | borough),
  data = modeling_data,
  family = neg_binomial_2,
  chains = 4, iter = 1000*2, seed = 84735, refresh = 0
)
```

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

`$\text{and where}$`

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

<center>
<img src="/media/bayes/plot_15.png" width="100%" height="100%" />
</center>

It seems that our simulations of non-citizen count distributions in the hierarchical model were largely consistent across the iterations. Similar to the non-hierarchical model, our simulations are biased away from the observed non-citizen counts as our model predicts a higher number of non-citizens more frequently. 


### Model 5: Mean Neighborhood Rental Prices

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

`$\text{and where}$`

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

<center>
<img src="/media/bayes/plot_16.png" width="100%" height="100%" />
</center>

Our simulations are relatively consistent. However, these simulated distributions are much more variable than the simulated distributions we saw in our non-citizen rent model. Importantly, it also seems our simulated normal posterior distributions had higher variance than what was observed and that is likely because the mean rental prices themselves are not perfectly normally distributed. Theoretically, we could change our likelihood model to adjust for the skew, but for now this is good!


### Model 6: Eviction Count

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

`$\text{and where}$`

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

<center>
<img src="/media/bayes/plot_17.png" width="100%" height="100%" />
</center>


Our simulations of eviction count distributions are dramatically consistent across the iterations. Further, our simulations were relatively consistent with the observed eviction counts. Our negative-binomial model seems to be a pretty good distributional fit, but could certainly be improved upon! 

In the following sections, we go through each particular model's performance before selecting our final set of models to interpret.


## Model Performance

### Immigrant/Non-Citizen Count


| model            | mae      | mae\_scaled | within\_50 | within\_95 |
| ---------------- | -------- | ----------- | ---------- | ---------- |
| Non-Hierarchical | 1026.946 | 0.5670020   | 0.5666667  | 0.9666667  |
| Hierarchical     | 1036.095 | 0.5555193   | 0.5555556  | 0.9666667  |

Using in-sample scaled MAE, it’s evident that the differences between our two negative binomial models of non-citizen counts were minute In other settings, we would typically recommend to use cross-validated error metrics to truly compare these models. However, because we are comparing a hierarchical model to a non-hierarchical model, cross-validation works slightly differently. In the former case, cross-validation via the `prediction_summary_cv` function in `bayesrules` makes data permutations to represent a "new borough", rather than a set of arbitrary models. To circumvent these differences, we use the expected-log predictive density (ELPD) of each respective model to compare the performance of each model. The details of this approach can be found [here](https://www.bayesrulesbook.com/chapter-10.html#chapter-10-train-test).



| model            | elpd\_diff        | se\_diff    | elpd\_loo | se\_elpd\_loo | p\_loo   | se\_p\_loo | looic    | se\_looic |
| ----------------- | ----------- | --------- | ------------- | -------- | ---------- | -------- | --------- |
| hi\_noncit\_model | 0.0000000   | 0.0000000 | \-1603.686    | 11.69514 | 13.74279   | 2.572952 | 3207.373  | 23.39029 |
| noncit\_model     | \-0.7008571 | 0.6682885 | \-1604.387    | 11.59500 | 14.55443   | 2.396960 | 3208.775  | 23.19000 |


From the above ELPD rankings, it seems that the non-hierarchical model of non-citizen counts performed better than its hierarchical parallel. Explicitly, there was a decrease of .536 standard deviations when comparing the non-hiearchical to its hiearchical parallel.


<center>
<img src="/media/bayes/plot_21.png" width="100%" height="100%" />
</center>


The residuals on our hierarchical model are not perfectly randomly distributed across all neighborhoods in NYC. Specifically, one could also make the case that there are slight over predictions (darker blue) of non-citizen counts in the Brooklyn (bottom left) for the non-hierarchical model. However, these differences and concentrations seem mostly negligible upon visual inspection.


| Model            | WAIC     | SE       |
| ---------------- | -------- | -------- |
| Non-Hierarchical | 3208.463 | 23.15916 |
| Hierarchical     | 3206.880 | 23.22938 |

Our last error metric of non-citizen counts is the Watanabe–Akaike information criterion (WAIC). When comparing two models with the WAIC, the better predicting model would be the model with the smaller WAIC value. Here, it seems that non-hierarchical model of non-citizen counts has the lowest WAIC, but the difference is largely minimal. Altogether, we choose the hierarchical model of non-citizen counts given its lower ELPD, WAIC, and spatial residual patterning.

## Mean Neighborhood Rental Prices

| model            | mae       | mae\_scaled | within\_50 | within\_95 |
| ---------------- | --------- | ----------- | ---------- | ---------- |
| Non-Hierarchical | 0.8377195 | 0.5259208   | 0.6055556  | 0.9500000  |
| Hierarchical     | 0.8024302 | 0.5087840   | 0.5888889  | 0.9444444  |

Once again when using in-sample scaled MAE, the differences between our two models of mean rental prices were largely minimal. Our predictions of rental price in both models were about 80 dollars away from their observed rental prices. However, the scaled mean absolute error metrics in the hierarchical model indicate that across simulations the observed value is about 0.3 standard deviations away from their the medians of the posterior predictive distributions of rental prices. This is a nontrivial improvement from the .5 standard deviation in the case of the non-hierarchical distribution. 

| model            | elpd\_diff      | se\_diff   | elpd\_loo | se\_elpd\_loo | p\_loo   | se\_p\_loo | looic    | se\_looic |
| --------------- | ---------- | --------- | ------------- | -------- | ---------- | -------- | --------- |
| hi\_rent\_model | 0.000000   | 0.0000000 | \-340.4415    | 14.97455 | 16.67414   | 3.558889 | 680.8830  | 29.94909 |
| rent\_model     | \-2.194559 | 0.6438043 | \-342.6361    | 14.82936 | 18.33416   | 3.689465 | 685.2721  | 29.65871 |

From the above ELPD rankings, it seems that the hierarchical model of mean neighborhood rental prices performed better than its non-hierarchical parallel.

<center>
<img src="/media/bayes/plot_21.png" width="100%" height="100%" />
</center>


Regarding the non-hierarchical model, the distribution of our residuals indicate that our model's mispredictions are randomly distributed. Further, our residuals in the non-hierarchical model are relatively small when compared to the hierarchical model. In the hierarchical model there is a slight indication of systematic mispredictions that appear distributed by borough. Crucially, it seems that we are systematically overpredicting rental prices in both Queens (bottom right) and Brooklyn (bottom left), but with greater overprediction occurring for Brooklyn. 

| Model            | WAIC     | SE       |
| ---------------- | -------- | -------- |
| Non-Hierarchical | 684.9805 | 29.67027 |
| Hierarchical     | 680.4569 | 29.84038 |

It seems that hierarchical model of rental prices has the lowest WAIC, but the difference is largely minimal. The hierarchical model’s residual patterning, ELPD metric, in-sample residual errors, and WAIC all indicate that the hierarchical version may be more appropriate.


## Eviction Count

| model            | mae      | mae\_scaled | within\_50 | within\_95 |
| ---------------- | -------- | ----------- | ---------- | ---------- |
| Non-Hierarchical | 42.28500 | 0.5461398   | 0.5777778  | 0.9555556  |
| Hierarchical     | 43.08787 | 0.5467191   | 0.5777778  | 0.955555   |

Using in-sample scaled MAE, the differences between our two negative-binomial models of evictions count were largely minimal. Our predictions of eviction counts in both models were about 40 cases away from their observed counts and with very similar scaled MAE metrics. Typically, we'd suggest the use of the non-hierarchical model given that the performance is so similar and the computational burden of a hierarchical model is no longer warranted. 

| model            |  elpd\_diff          | se\_diff    | elpd\_loo | se\_elpd\_loo | p\_loo   | se\_p\_loo | looic    | se\_looic |
| ------------------- | ----------- | --------- | ------------- | -------- | ---------- | -------- | --------- |
| hi\_eviction\_model | 0.0000000   | 0.0000000 | \-1051.930    | 13.74662 | 16.95747   | 2.781855 | 2103.86   | 27.49324 |
| eviction\_model     | \-0.0950349 | 0.5368605 | \-1052.025    | 13.84190 | 17.51510   | 2.923380 | 2104.05   | 27.68380 |

From the above ELPD rankings, it also seems that the hierarchical model of eviction counts performed better than its non-hierarchical parallel. Next, we will compare how these 2 models' residuals are spatially distributed.

<center>
<img src="/media/bayes/plot_22.png" width="100%" height="100%" />
</center>

Our residuals are randomly distributed across all neighborhoods indicating that our model’s bias is consistent across observations— this is really good! One could make the case that there are slight under predictions (darker red) of eviction counts in the Bronx (top) for both models. However, these patterns are relatively minute.

| Model            | WAIC     | SE       |
| ---------------- | -------- | -------- |
| Non-Hierarchical | 2103.173 | 27.53465 |
| Hierarchical     | 2103.144 | 27.38933 |

It seems that hierarchical model of rental prices has the lowest WAIC, but only by .03 units. But given the hierarchical model’s residual patterning, ELPD metric, in-sample residual patterns, and WAIC we select the hierarchical model of eviction counts.



# Discussion \& Extensions

We’ve confirmed that even after adjusting for income, rental prices, income inequality, and borough differences, neighborhoods in NYC with a substantial proportion of Black and Latinx residents are experiencing dramatic increased risks of eviction. Interestingly, when accounting for economic and demographic predictors, the relationships with transportation we expected to see were reversed. That is, increased transportation access was associated with more evictions. This could be related to the increased population sizes in areas with the best access to transportation (e.g. Manhattan) or it could be because the disparities that transportation inaccess reflects (e.g. racial and class-based tensions) have been accounted for.

Importantly, we found that there was statistically significant spatial clustering of both eviction counts and mean rental prices using Moran's I from the `spdep` package. 

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

| Term      | statistic | p.value | parameter | method                            | alternative |
| --------- | --------- | ------- | --------- | --------------------------------- | ----------- |
| Mean Rent | 0.6560430 | 0.001   | 1000      | Monte-Carlo simulation of Moran I | greater     |
| Eviction  | 0.5794612 | 0.001   | 1000      | Monte-Carlo simulation of Moran I | greater     |


Our results are then likely biased by the spatial relationships between neighborhoods. As such, we could extend this work to the spatial domain using methods from the `CARBayes` or `INLA` packages. Following work by [Katie Jolly and Raven McKnight](https://www.ravenmcknight.com/post/carbayes-tutorial/), we have outlined a spatial workflow for both the eviction and rental price models using `CARBayes` in the appendix. However, because spatial models were beyond the scope of this project and class, we'd like to emphasize that in a more detailed analysis the models we outline in the appendix would likely be adjusted in `STAN` or `CARBayes` to use different likelihoods, spatial effect priors (e.g. BYM, Intrinsic CAR, etc), and different priors for the predictors' coefficients. Further, in a more detailed analysis, we would explicitly describe the mathematical construction of the models.

Although a lot of work went into designing models with causal blocking and confounding in mind, our models remain imperfect. Some major limitations involved the encoding of transportation deserts and the presence of unmeasured confounders.Because we may not have not accounted for all structural predictors in housing equity such as a neighborhood’s rent control policies nor have we adjusted for the reasons behind eviction, we cannot be entirely confident about our models' conclusions.

Across our models, it becomes clear that many of the structural housing and demographic issues present in NYC need to be more rigorously addressed by both policy-makers and its citizens, regardless of these particular models' performance. Health begins at home. And if NYC's Black and Latinx residents are being crushed under the fist of inequity and consequently experiencing increased risks of eviction or tenuous rental prices, then it becomes a health imperative to critically and revolutionarily address NYC's housing system. 
