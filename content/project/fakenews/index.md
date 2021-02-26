---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "So Fake it's Californian"
summary: "Applying Random Forests & Sentiment Analysis to the Detection of Fake News"
authors: []
tags: 
- Art
- Data Science
categories: []
date: 2019-11-02T14:47:00-05:00

# Optional external URL for project (replaces project detail page).
external_link: ""

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: true

# Custom links (optional).
#   Uncomment and edit lines below to show custom links.
links:
- name: ""
  url: https://twitter.com/freddybarragan_
  icon_pack: fab
  icon: twitter
- name: ""
  url: https://instagram.com/mossydraw
  icon_pack: fab
  icon: instagram
  

url_code: ""
url_pdf: ""
url_slides: ""
url_video: ""

# Slides (optional).
#   Associate this project with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides = "example-slides"` references `content/slides/example-slides.md`.
#   Otherwise, set `slides = ""`.
slides: ""
---

# Part 1: Process the data

## Setup
```{r}
library(tidyverse)
library(ggrepel)
library(sentimentr)
library(ngram)
library(syuzhet)
library(data.table)
library(stringr)
library(gridExtra)
library(caret)    # for machine learning algorithms
library(rpart)        # for building trees
library(rpart.plot)  # for plotting nice trees
library(randomForest)
library(kableExtra)
library(class)
```

```{r}
fakenews <- read.csv("https://www.macalester.edu/~ajohns24/data/buzzfeed.csv")
```

## Variable Creation

```{r master, cache=TRUE}
# Capitalization Counts

fakenews <- fakenews %>%
  select(-url, -source) %>%
  mutate(title_cap_count = str_count(title, "[A-Z]")) %>%
  mutate(text_cap_count = str_count(text, "[A-Z]"))


# Sentiment Analysis

  ## title sentiment
  title_sent <- sentiment(fakenews$title) 
  ## text sentiment
  text_sent <- sentiment(fakenews$text) 

  title_sent_clean <- title_sent %>%
    group_by(element_id) %>%
  ## title word count
    summarise(word_count =sum(word_count), mean_sent= mean(sentiment)) %>%
    rename(title_count = word_count) %>% 
    rename(title_sent = mean_sent) %>%
    select(-element_id)
  

  text_sent_clean <- text_sent %>%
    group_by(element_id)%>%
  ## text word count
    summarise(word_count =sum(word_count),mean_sent= mean(sentiment)) %>%
    rename(text_count = word_count) %>%
    rename(text_sent = mean_sent)  %>%
    select(-element_id)
  
  
fakenews_master <- cbind(fakenews, title_sent_clean, text_sent_clean)

  
  ## emotional word counts in text
  text_emotions <- data.frame(get_nrc_sentiment(fakenews_master$text)) %>%
    rename(anger_text = anger, anticipation_text = anticipation, disgust_text = disgust, fear_text = fear, joy_text = joy, sadness_text = sadness, surprise_text = surprise, trust_text = trust, negative_text = negative, positive_text = positive)

  ## emotional word counts in title
  title_emotions <- data.frame(get_nrc_sentiment(fakenews_master$title)) %>%
    rename(anger_title = anger, anticipation_title = anticipation, disgust_title = disgust, fear_title = fear, joy_title = joy, sadness_title = sadness, surprise_title = surprise, trust_title = trust, negative_title = negative, positive_title = positive)

  full_emotions <- cbind(text_emotions,title_emotions)

  
  ## text & title polarity 
  fakenews_master <- data.frame(cbind(fakenews_master,full_emotions)) %>%
    mutate(text_polarity = abs(text_sent)) %>%
    mutate(title_polarity = abs(title_sent))
```

```{r author}
# Author Counts


  fakenews_master <- fakenews_master %>%
    mutate(authors = gsub("View All Posts","",fakenews_master$authors))
  fakenews_master <- fakenews_master %>%
    mutate(authors = gsub("Fed Up","",fakenews_master$authors))
    fakenews_master <- fakenews_master %>%
    mutate(authors = gsub("Abc News","",fakenews_master$authors))%>%
    mutate(authors = gsub("Featured Commentator","",fakenews_master$authors))

  ## delete commas left behind by View All Posts in the front of an entry
  fakenews_master$authors<- ifelse(startsWith(fakenews_master$authors, ","), 
    gsub(",","",fakenews_master$authors), fakenews_master$authors
    )
  
  ## delete commas left behind by View All Posts at the end of an entry
  fakenews_master$authors<- ifelse(endsWith(fakenews_master$authors, ","),
    gsub(",","",fakenews_master$authors),  fakenews_master$authors)

  ## author count
  fakenews_master <- fakenews_master %>%
    mutate(author_count =  sapply(strsplit(authors, ","), length)) 
```


```{r exclamation}
# Grammar Counts

  ## exclamation points in title & text
  fakenews_master <- fakenews_master %>%
    mutate(title_exclaim_count = str_count(title, "!")) %>%
    mutate(text_exclaim_count = str_count(text, "!"))


  ## counts number of commas in a title
  fakenews_master <- fakenews_master %>%
    mutate(title_grammar =  (str_count(title, ",")/title_count))


  ## proportion of capslocked words in a title
  fakenews_master <- fakenews_master %>%
    mutate(title_caplock_prop = (str_count(title, "\\b[A-Z]{3,}\\b")/title_count))
 
  ## proportion of capslocked words in text
  fakenews_master <- fakenews_master %>%
    mutate(text_caplock_prop = (str_count(text, "\\b[A-Z]{3,}\\b")/text_count))
  
  ## number of capslocked words in a title
    fakenews_master <- fakenews_master %>%
      mutate(title_caplock = (str_count(title, "\\b[A-Z]{3,}\\b")))
  ## number of capslocked words in text
    fakenews_master <- fakenews_master %>%
      mutate(text_caplock = (str_count(text, "\\b[A-Z]{3,}\\b")))
```

```{r emails}
# Buzzwords

  ## mention of emails
  fakenews_master <- fakenews_master %>%
    mutate(emails = str_count(text, "emails"))

  ## mention of SHARES in all caps
  fakenews_master <- fakenews_master %>%
    mutate(share = str_count(text, "SHARES"))
```

```{r proportion emotion}
# Get proportion of emotion variables
fakenews_master <- fakenews_master %>%
  filter(!is.na(text_count)) %>%
  filter(!is.na(title_count)) %>%
  mutate_at(vars(contains('_text')), funs(./text_count)) %>% #renaming
  mutate_at(vars(contains('_title')), funs(./title_count))
```

```{r final}
# cleaned data w/o text
fakenews_final <- fakenews_master %>%
  select(-c(text,title,authors))%>%
  mutate_if(is.character, ~as.factor(.)) 
```

> In the above code chunks we have defined 39 variables using sentiment analysis packages (`sentimentr` & `syuzhet`) and string counts on both the titles and the texts of each article. Below is a table of each variable and their definition:

```{r}
Variable <- c(colnames(fakenews_final))

Definition <- c("Whether an article is real or fake.", "Number of capitalized characters in an article's title.", "Number of capitalized characters in an article's text.", "Number of words in an article's title.", "Mean sentiment in the article's title (negative values = negative feelings).",  "Number of words in an article's text.", "Mean sentiment in the article's text (negative values = negative feelings).", "Proportion of words in the article's text that are associated with anger.", "Proportion of words in the article's text that are associated with anticipation.", "Proportion of words in the article's text that are associated with disgust.", "Proportion of words in the article's text that are associated with fear.", "Proportion of words in the article's text that are associated with joy.", "Proportion of words in the article's text that are associated with sadness.", "Proportion of words in the article's text that are associated with surprise.", "Proportion of words in the article's text that are associated with trust.", "Proportion of words in the article's text that are associated with negative emotions.", "Proportion of words in the article's text that are associated with positive emotions.", "Proportion of words in the article's title that are associated with anger.", "Proportion of words in the article's title that are associated with anticipation.", "Proportion of words in the article's title that are associated with disgust.", "Proportion of words in the article's title that are associated with fear.", "Proportion of words in the article's title that are associated with joy.", "Proportion of words in the article's title that are associated with sadness.", "Proportion of words in the article's title that are associated with surprise.", "Proportion of words in the article's title that are associated with trust.", "Proportion of words in the article's title that are associated with negative emotions.", "Proportion of words in the article's title that are associated with positive emotions.", "Mean absolute sentiment of an article's text to reflect its polarity.", "Mean absolute sentiment of an article's title to reflect its polarity.", "Number of exclamation points in the article's title.", "Number of exclamation points in the article's text.", "Number of listed authors.", "Number of commas in an article's title.", "Number of capslocked words in an article's title.", "Number of capslocked words in an article's text.",  "Number of capitalized characters in an article's title.", "Number of capitalized characters in an article's text.", "Number of times the word 'emails' is mentioned in an article's text.", "Number of times the word 'SHARES' is mentioned in an article's text")

meta_table <- data.frame(cbind(Variable, Definition))

kable(meta_table, align = "c") %>%
  kable_styling()%>%
  scroll_box(height="400px")
```

> There's so much info! So, let's connect these defined predictors to two real articles and see how these variables play out in the wild. Below is a table of full observations for two articles.

```{r}
kable(fakenews_master[c(63, 83),] %>%
        select(-c(text, authors)), aalign = "c") %>%
  kable_styling()%>%
  scroll_box(width="100%")
```

> There's still so much information! In particular, it seems that capitalization counts, title word counts, title polarity, title & text exclamation, and sentiment values are higher in fake news, but the number of authors and word counts in text are lower. Between real and fake news, the proportion of words associated with the various emotions do not change drastically, but in this instance our first news article's title and text were slightly more emotive across the board. Among these predictors it seemed like variables regarding word, capitalization, sentiment, and author counts were the most variable by type.

\
\
\
\

## Visualizations {.tabset}

> In this section we will be looking at a select number of predictors whose expressions were different by an articles type or are nominally important identifiers of real and fake news 

### Sentiment 

```{r,fig.width=10}
p <- ggplot(fakenews_final, aes(x=text_sent, fill=type)) +
  geom_density(alpha=.3) + 
  labs(guide=NULL)+
  theme_linedraw()

p2 <- ggplot(fakenews_final, aes(x=title_sent, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()


p3 <- ggplot(fakenews_final, aes(x=text_polarity, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()


p4 <- ggplot(fakenews_final, aes(x=title_polarity, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()

grid.arrange(p,p2,p3, p4, ncol=2) 
```

> Above are four density plots describing text (right) and title (left) sentiment and polarity colored by an article's validity. Consistent with our example, it doesn't seem like there are sufficient differences between real and fake news articles' density distributions to use these as classifiers.  

\
\
\
\

### Word Count

```{r,  fig.width=10}
p <- ggplot(fakenews_final, aes(x=title_count, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()

p2 <- ggplot(fakenews_final, aes(x=text_count, fill=type)) +
  geom_density(alpha=.3) +
  scale_x_log10()+ 
  theme_linedraw()

grid.arrange(p,p2, ncol=2) 
```

> We noted previously that word counts between real and fake news varied. Using these density plots, we've gotten confirmation that there is some notable split in the number of characters in a title and more slight split in the number of characters in a text. 

\
\
\
\

### Capitalization
```{r,  fig.width=10}
p <- ggplot(fakenews_final, aes(x=title_caplock, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()

p2 <- ggplot(fakenews_final, aes(x=title_cap_count, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()

grid.arrange(p,p2, ncol=2)
```

> The density plot on the left summarizes the number of words that were written in capslock. On the right is the raw count of capitalized characters in the title. Using these density plots, we've gotten confirmation that there is an ostensible split in the number of capitalized characters and words in a title by type.

\
\
\
\

### Authors
```{r}
ggplot(fakenews_final, aes(x=author_count, fill=type)) +
  geom_density(alpha=.3) + theme_linedraw()
```

> Above is a distribution of author counts. The distribution is multimodal, but easily separated between an article's type, meaning that it will likely be a good predictor of an article's validity. Generally, we know that among articles without any listed authors, the majority are fake, but as you increase the number of authors, the majority of observations are real. This will likely be a key detail in our model's classification methods.


\
\
\
\

## Limitations

> Though our methods are robust, using sentiment analysis and string counting to predict a news article's validity will not produce the perfect results without more details that are not at our disposal. In most cases, fake news circulates through social media channels (e.g. Twitter, Reddit, Facebook, etc.) and is primarily communicated in videos, photos, and in slang. So our approach would not be well-equipped handling the "real world" of fake news, given the linguistic complexity of slang, the inability to parse visual media, and the explicit articles we use in our dataset. Even though they are commonly tracked and encoded in real spam and fake news detection software, we haven't taken reader, poster, and sharer demographics into account— that's a big shortcoming of this modeling approach. Lastly, we are making a very particular assumption about English fluency with text analysis which fails when we consider the millions of people who speak a dialect of English or English as second language. How can this text analysis correctly classify the melding of dialects, languages, and slangs to produce equitable classifications of fake news on a broader scale?



\
\
\
\
\
\



# Part 2: Analysis
##  Models {.tabset}

> In typical classification settings, its critical to strike a balance between interpretability, computational efficiency, and curses of dimensionality. Using sentiment analysis and text & title statistics, we've produced a high-dimension dataset of highly correlated variables (e.g. disgust, anger, and negativity) which likely have nonparametric relationships with a news article's validity. So, I am testing three concurrent classification methods by first identify meaningful predictors and then simplifying our model, while maximizing our accuracy: Random Forests, KNN Classification, and Simple Logistic Regression.


### 1. Random Forest

> I am using a random forest to identify significant predictors which will be fed into our KNN and logistic classifiers. The primary reason I'm using this for variable selection is because it can identify parametric and non-parametric predictors, making it more successful than a LASSO regression, while still providing an excellent predictive framework, if our subsequent models don't perform well.

```{r}
set.seed(253)

forest_model <- train(
  type ~ .,
  data = fakenews_final,
  method = "rf",
  tuneGrid = data.frame(mtry = c(2, 6, 20, 39)),
  trControl = trainControl(method = "oob"),
  metric = "Accuracy",
  na.action = na.omit
)
```

```{r}
plot(forest_model)
```

>  This is a visualization of our random forest's accuracy metrics as we increased the number of predictors in our trees Ultimately, it means that our most accurate model had a relatively small number of predictors (6)! That's awesome for our analysis and justifies other classification methods which are typically hindered by data in higher dimensions (e.g. KNN Classifiers).


```{r}
variable_importance <- data.frame(randomForest::importance(forest_model$finalModel)) %>% 
  mutate(predictor = rownames(.))

# Arrange predictors by importance (most to least)
kable(variable_importance %>% 
  arrange(desc(MeanDecreaseGini)) %>% 
  head())%>%
  kable_styling()
```

> We've identified the 6 most influential predictors: `title_cap_count`, `author_count`, `title_count`, `disgust_text`, `text_sent`, `title_caplock_prop.` They are ranked by their `MeanDecreaseGini` metric, which is essentially their ability to "purify" the classifications at each split in a tree (i.e. reduce the number of misclassifications). One important detail is that `title_cap_count`'s mean decrease in Gini was substantially higher than other predictors. Given the distributions we saw in the previous section, it makes sense that these predictors were chosen— each had some identifiable split between real and fake news— but now lets see how accurate of a forest we've made. 


```{r}
# Define forest_plot
forest_plot <- function(x1, x2, y, lab_1, lab_2){
  y <- as.factor(y)
  model <- randomForest(y ~ x1 + x2, ntree = 500)
  x1s <- seq(min(x1), max(x1), len = 100)
  x2s <- seq(min(x2), max(x2), len = 100) 
  testdata <- expand.grid(x1s,x2s) %>% 
    mutate(class = predict(model, newdata = data.frame(x1 = Var1, x2 = Var2), type = "class"))
  
  ggplot(testdata, aes(x = Var1, y = Var2, color = class)) + 
    geom_point() + 
    labs(x = paste(lab_1), y = paste(lab_2), title = "Random Forest Classification Boundaries") + 
    theme(legend.position = "bottom")+
    theme_linedraw()
}

forest_plot(x1 = fakenews_final$author_count, x2 = fakenews_final$title_cap_count, y = fakenews_final$type, lab_1 = "author_count", lab_2 = "title_cap_count")
```

> Above is a visualization of this random forest's classification boundaries only using two of the 6 predictors. These boundaries indicate that as the number of capitalized characters increase in a title, an article will likely be classified as "fake". This could mean that a classifier with a smaller number of predictors could be appropriate. Additionally because they have slightly erratic placement, it may be nice to use another method so that we can get more localized classifications of our model.


```{r}
forest_model$finalModel
```

> Our forest's OOB Accuracy when predicting the difference between real and fake news is $\approx `r 100 -25.14` \%$ when using 6 predictors— that's really accurate! One thing to note is that our model had a higher specificity ($`r (round((1-0.1333333), 4) *100)` \%$) than sensitivity ($`r (round((1-0.3764706), 4) *100)` \%$). Given that "fake news" can have more disastrous implications on broad American politics, its nice that our specificity is higher. However, the goal of future sections is to improve our classification system's accuracy, specificity, and sensitivity, while simplifying our model.


\
\
\
\


### 2. 10-Fold CV KNN

> KNN Classification is most inappropriate when considering data in high-dimensions, but it is substantially more interpretable and simpler than a random forest. So, we will use the two most significant predictors previously identified: `title_cap_count` and `author_count` to classify news. I am using just 2 predictors, as opposed to the full 6, because the difference between these two predictors and the other predictors by their `MeanDecreaseGini` is substantially larger meaning that we can likely get good predictions in just 2 dimensions. Additionally, because a majority of the variables selected were related to the number of characters 

```{r}
# Set the seed
set.seed(253)

# Build a KNN model
knn_model <- train(
  type ~ author_count + title_cap_count,
  data = fakenews_final,
  preProcess = c("center","scale"),
  method = "knn",
  tuneGrid = data.frame(k = c(1:19, seq(20, 150, by = 5), 175)),
  trControl = trainControl(method = "cv", number = 10, selectionFunction = "best"),
  metric = "Accuracy",
  na.action = na.omit
)
```

```{r}
plot(knn_model)
```

> These are our KNN accuracy metrics as we increase the number of neighbors in each neighborhood. It seems like we get our best classification when K is around 10 That isn't ideal for variance, but because KNN is a lazy learner and our ultimate goal is classification ability, it's expected. Below we've zoomed in on the accuracy metrics for the most accurate model.

```{r}
knn_model$results %>% 
  filter(k == knn_model$bestTune$k)
```

> We were right about KNN's potential! Using KNN-classifiers we can correctly classify news $`r round((knn_model$results %>% filter(k == knn_model$bestTune$k))$Accuracy,4)*100` \%$ of the time when $K=12$. So by using just 2 predictors, we've increased our accuracy by $3.44$ percentage points and simplified our model substantially. However, let's look at this model's sensitivity and specificity to verify improvement.

```{r}
knn_class <- predict(knn_model, newdata = fakenews_final, type = "raw")

kable(table(fakenews_final$type, knn_class), align = "c") %>%
  kable_styling()
```

> We've improved our accuracy using a KNN classifier, but we've also *substantially* improved our specificity from the random forest. Now, our sensitivity ($`r round((56/85), 4)*100` \%$) to real news has increased (difference = $3.52941$) and now we can correctly classify fake news $`r round((85/90), 4)*100` \%$ of the time. That's excellent given the circumstances and the simplicity of this classifier.
>
Now let's examine these classification boundaries.

```{r}
# Define KNN plot
knnplot <- function(x1, x2, y, k, lab_1, lab_2){
  x1s <- seq(min(x1), max(x1), len = 100)
  x2s <- seq(min(x2), max(x2), len = 100) 
  testdata <- expand.grid(x1s,x2s)
  knnmod <- knn(train = data.frame(x1,x2), test = testdata, cl = y, k = k, prob = TRUE)
  testdata <- testdata %>% mutate(class = knnmod)
  
  ggplot(testdata, aes(x = Var1, y = Var2, color = class)) + 
    geom_point() + 
    labs(x = paste(lab_1), y = paste(lab_2), title = "KNN Classification Boundaries") + 
    theme(legend.position = "bottom")+
    theme_linedraw()
}

knnplot(x1 = fakenews_final$author_count, x2 = fakenews_final$title_cap_count, y = fakenews_final$type, lab_1 = "author_count", lab_2 = "title_cap_count", k = knn_model$bestTune$k)

```

> These classification boundaries are similarly located to the ones in our random forest, but they are substantially more granulated. That is a byproduct of using KNN, as opposed to a forest, but they do warrant some celebration, given their accuracy and simplicity. Still these boundaries indicate that as you increase `title_cap_count`, an article will be classified as fake news, barring those exceptions when `title_cap_count` is between 15 and 25.

\
\
\
\


### 3. 10-Fold CV Logistic Model

> Using a random forest for variable selection was justified given that it can identify good predictors of an article's validity without assuming a linear relationship to the outcome. In this section, we're validating the necessity of a nonparametric approach by using a 10-Fold CV logistic regression. This model is one of the simplest classification methods, but it assumes variables like `author_count` will have a parametric relationship with the outcome which is likely not the case given its probability distribution.

```{r}
set.seed(253)

# Perform logistic regression
logistic_model <- train(
    type ~ title_cap_count + author_count,
    data = fakenews_final,
    method = "glm",
    family = "binomial",
    trControl = trainControl(method = "cv", number = 10),
    metric = "Accuracy",
    na.action = na.omit
)
```


```{r}
summary(logistic_model)

# Model summary table
kable(summary(logistic_model) %>%
  coef() %>%
  exp(), align="c") %>%
  kable_styling()
```

> On average, the odds of an article being real decrease by a multiplicative factor of $0.8413711$, as the number of capitalized letters increase in a title, when controlling for the number of listed authors. Conversely, the odds of an article being real increase by a multiplicative factor of $1.1569697$, as the number of listed authors increase, when controlling for the number of capitalized letters in a title.

> It's also important to notice that `author_count` was not classified as statistically significant under parametric assumptions. That conclusion makes sense given its multi-modal density distributions, but this is the most immediate indication that a parametric approach may not be useful.


```{r}
predict_logistic <- na.omit(logistic_model$trainingData)

classifications <- predict(logistic_model, newdata = predict_logistic, type = "raw")

confusionMatrix(data = classifications, 
  reference = predict_logistic$.outcome, 
  positive = "real")
```

```{r}
kable(logistic_model$results, align = "c") %>%
  kable_styling()
```

> Our accuracy metrics are *roughly* similar across all the modeling methods we've used. However, this cross-validated logistic classifier had the lowest cross-validated accuracy ($`r round((0.7486),4)*100` \%$) and specificity ($`r round((0.7667),4)*100` \%$) of the three. This is likely because these predictors may not fulfill the linear assumptions underpinning a logistic regression. And although we have slightly improved our sensitivity, it would be inappropriate to use this method for our final classification system given the parametric relationships we are ignoring and the potentially severe implications of misclassifying fake news.


\
\
\
\


# Part 3: Summarize

> Fake news has disastrous effects. In the U.S., it has resulted in dangerous [anti-mask misinformation]("https://www.boomlive.in/world/anti-mask-posts-use-fake-who-document-to-spread-misinformation-9424") and an increased risk of COVID-19 transmission; [child-trafficking allegations on the 2016 democratic ballot]("https://www.politifact.com/article/2016/dec/05/how-pizzagate-went-fake-news-real-problem-dc-busin/") which crippled their electoral success; and [smear campaigns on antifascist organizers, during the George Floyd protests]("https://www.reuters.com/article/uk-factcheck-antifa-twitter-fake/fact-check-antifa-twitter-account-that-called-for-violence-was-fake-idUSKBN23B2TY"). And given its omnipresence in digital space, it's critical that we are able to identify, remove, and prevent fake news as stringently as possible (i.e. with high specificity).
>
Using random forests and density distributions, we've identified a good subset of predictors: `title_cap_count`, `author_count`, `title_count`, `disgust_text`, `text_sent`, and `title_caplock_prop.` However, because most of these predictors were related to redundant counts of title's capitalization and had a low mean decrease in Gini, we simplified our following models to 2 predictors: `title_cap_count` and `author_count`. Below is a visualization of article validity by both predictors.

```{r,  fig.width=10}
p <- ggplot(fakenews_final, aes(author_count, title_cap_count, color=type))+
  geom_point() +
  labs(title="An Initial Scatterplot")+
  scale_x_continuous(breaks= seq(0,10, by=1))+
  theme_linedraw()
  
  
p2 <- ggplot(fakenews_final, aes(author_count, title_cap_count, color=type))+
  geom_jitter() +
  labs(title="A More Useful Scatterplot")+
  scale_x_continuous(breaks= seq(0,10, by=1))+
  theme_linedraw()

grid.arrange(p,p2, ncol=2) 
```

> From both plots, we know that a large majority of fake news articles have a high number of capitalized characters and few number of authors. Likely, this distribution is a product of the title formatting typically seen in articles whose main goal is to virally spread sensationalist information (e.g. "SHARE!", "LOOK", "HOMOSEXUALITY KILLS"). While increasing the number of authors typically implies an added peer-review process between writers— a good indication of being real— the stunning omission of author names in these data is likely in part due to viral resharing and intentional anonymity. 
>
Next, we will identify the "best" model and what it means for the classification of fake news. So, let's compare accuracy metrics. Below is a table of each model's classification statistics, arranged by their accuracy.

```{r}
set.seed(253)

log <- logistic_model$results %>%
  mutate(parameter = "NA") %>%
  rename("Tuning Parameter" = parameter) %>%
  mutate(Model = "Logistic Regression") 

knnmod <- knn_model$results %>% 
  filter(k == knn_model$bestTune$k) %>%
  mutate(k = "K = 5") %>%
  rename("Tuning Parameter" = k) %>%
  mutate(Model = "KNN")

forest <- forest_model$results %>% 
  filter(mtry == forest_model$bestTune$mtry) %>%    
  mutate(mtry = "mtry = 6") %>%
  rename("Tuning Parameter" = mtry) %>%
  mutate(Model = "Random Forest")
  
model_accuracies <- bind_rows(forest, knnmod, log) %>%
  select(Model, Accuracy, Kappa)

Specificity <- c(round(((1-0.1333333)*100),2), round((85/90)*100, 2), round((0.7667*100),2))

model_accuracies <- cbind(model_accuracies, Specificity)
         
```

```{r}
kable(arrange(model_accuracies, desc(Accuracy)), align = "c") %>%
  kable_styling()
```

> KNN was the most accurate and specific model. Using our KNN classifier, we improved classification accuracy by $`r round(0.0115779, 4)*100`$ percentage points from the random forest and by $`r round(0.0343137, 4)*100`$ from the logistic regression. The KNN model also had the highest specificity of every classification method ($`r (round((53/90), 4)*100)` \%$). Typically, KNN is difficult to interpret and even more difficult to implement when using data with a large number of potential predictors. However, the benefits of a random forest for variable selection have allowed us to create a very simple model which correctly accounts for the nonparametric relationship between validity and its predictors (e.g. `author_count.`). Most importantly, this model allows to correctly flag and identify fake news articles with a very low false-negative rate, thus minimizing the spread of potentially dangerous information. 

