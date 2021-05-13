---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "So Fake it's Californian"
summary: "Applying Random Forests, KNN Regression, & Sentiment Analysis to the Detection of Fake News"
authors: [Freddy Barragan]
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

date: 2019-11-02T14:47:00-05:00
---

Fake news has disastrous effects. In the U.S., it has resulted in dangerous [anti-mask misinformation]("https://www.boomlive.in/world/anti-mask-posts-use-fake-who-document-to-spread-misinformation-9424") and an increased risk of COVID-19 transmission; [unfounded, child-trafficking allegations on the 2016 democratic ballot]("https://www.politifact.com/article/2016/dec/05/how-pizzagate-went-fake-news-real-problem-dc-busin/"); and [smear campaigns on antifascist organizers, during the George Floyd protests]("https://www.reuters.com/article/uk-factcheck-antifa-twitter-fake/fact-check-antifa-twitter-account-that-called-for-violence-was-fake-idUSKBN23B2TY"). And given its omnipresence in digital space, it's critical that we are able to identify, remove, and prevent fake news as stringently as possible.

Leveraging sentiment analysis of real news articles & random forests for variable selection, I built a powerful, low-dimensional, fake news-classifier with KNN regression (K=5, 78\% 10-fold CV accuracy). Typically, KNN is difficult to interpret and even more difficult to implement when using data with a large number of potential predictors. However, the benefits of a random forest for variable selection have allowed us to create a very simple model which correctly accounts for the nonparametric relationship between validity and its predictors. Most importantly, this model allows to correctly flag and identify fake news articles with a very low false-negative rate, thus minimizing the spread of potentially dangerous information. Below are classification boundaries of our KNN-classifier, only using 2 predictors:

<center>
![](/media/fake_knnplot.png)
</center>

Though these methods are robust, using sentiment analysis and string counting to predict a news article's validity will not produce the perfect results without more details that were not at our disposal. In most cases, fake news circulates through social media channels (e.g. Twitter, Reddit, Facebook, etc.) and is primarily communicated in videos, photos, and in slang. By doing so, we are making a very particular assumption about English fluency with text analysis which fails when we consider the millions of people who speak a dialect of English or English as second language. How can this text analysis correctly classify the melding of dialects, languages, and slangs to produce equitable classifications of fake news on a broader scale?

 Data preparation & exploratory analysis was a collaborative effort between my friends \& classmates Juthi Dewan, Franco Salinas Meza, Hannah Staats, Kyaw Za Zaw, and Danny Frank-Siegel.