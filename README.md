
## A brief exploration of clustering and classification on Twitter data

The goal of this project was to apply clustering and classification to text data with multiple authors. For the text data I chose to gather recent tweets from 15 data science influencers on Twitter, as identified in a Hacker Noon post I have seen referenced. They are (with their Twitter handles): Kirk Borne (KirkDBorne), Ronald van Loon (Ronald_vanLoon), Craig Brown (craigbrownphd), Bob Hayes (bobehayes), Bernard Marr (BernardMarr), Lillian Pierson (BigDataGal), Andrew Ng (AndrewYNg), Monica Rogati (mrogati), Carla Gentry (data_nerd), Gregory Piatetsky (kdnuggets), Vincent Granville (analyticsbridge), Naval Ravikant (naval), Tamara Dull (tamaradull), Hilary Mason (hmason) and Evan Sinar (Evan Sinar). Since all 15 would be expected to tweet about data science topics, the classification is not expected to be overly easy.

Following is a summary of the procedures and results. The full code and results are available in twitter.ipynb.

## Clustering

After fetching the most recent 500 tweets from each author that were not retweets, by filtering the most recent 800, I kept the full texts of the tweets as well as the hashtag and mention entities. I first attempted to cluster based on the entities, which provide a distinctive and limited set of terms to cluster on. It turned out, however, that the number of common entities, used by at least two authors, differed tremendously from author to author, with two of the authors having none, three others having less than 10 and three having more than 1,000 (the highest being more than 4,000). This invalidated the approach, and I moved on to try clustering on the text tweets with tf-idf vectorization. Tweets are compact and rely on impact words more than grammatical constructions, so analyzing on vocabulary seems more appropriate than analysis involving parts of speech and phraseology.

Since I wanted to cluster the authors, I combined the tweet texts into one document (list) for each author. Neither mean shift nor spectral clustering produced promising results, while k-means offered the advantage of being able to look at the cluster centers as points in the feature space, which gives an idea of the topics they most relate to, while the vectorization also allowed comparison of the scores of the words for each author.

The initial attempt produced 7 clusters, with thematic similarity among the members of several of them, although two of the clusters contained only one author each and one contained the four authors who used few or no hashtags and mentions. The best clustering was found by first using 1,000 runs that looped through a range of cluster sizes to determine which produced the best (lowest) silhouette score. The optimal number of clusters was taken as the size that produced the best score the greatest number of times. Another 1,000 trials were then run using the best feature size to find the clustering with the best silhouette score. The Jupyter notebook contains a plot of these clusters against two LSA components.

Besides looking at the 10 highest-scoring terms for each author, a more programmatic attempt at surfacing similarity consisted of looping through the highest-ranked terms for each cluster center and finding those that had high tf-idf scores (above an experimentally determined threshold) for each member of that cluster. Additionally, I looked at the highest-ranking words for each cluster (whether or not members had them in common or as high-scoring words) to identify the main topics. These “rankings” essentially are how far the cluster center is along the axis representing each term in the feature space, which comprises all the terms in the tf-idf vectorization.

A second clustering attempt added in bigrams (using the n-gram functionality of sklearn’s tf-idf vectorizer. This time the optimal number of clusters was reduced to 6, which were identical to the initial 7 except that one of the formerly lone authors was assigned put into one of the existing clusters.

The third and final attempt tried to compensate for the highly divergent use of hashtags by matching where possible the hashtags in each author’s combined tweet texts with concatenated bigrams that emerged as a tf-idf feature. For example, #machinelearning would match “machine learning.” For each match, including multiple matches for the same hashtag, the corresponding bigram was appended to the combined texts. The idea was to augment similarity scores between authors that used the same key two-word phrases whether as hashtags or as separate words. The optimal number of clusters went down to 5. The one author who had remained solitary on their own in the second clustering attempt was assigned to a pre-existing cluster, but a pre-existing 2-person cluster disappeared, with one member assigned to another group but the other now alone. These clusters were again plotted against two LSA components:

![‘clustering plot’](cluster_plot.png)
> *In the final clustering, none of the clusters overlap even though the two components being graphed explain less than 20% of the total variance. This low explained variance also provides reasonable cause for the two points closest together in the graph, near (0.3, -0.4) being assigned to different clusters.*

The main topics for the clusters changed little between the first and third attempts, although it became more difficult to find terms common to all cluster members as clusters increased in size. Following is a diagram showing the clusters and key terms close to the cluster centers. Lines connect clusters that share a key term:

![‘diagram showing final clusters’](clusters.png)

## Classification  
For classification purposes, the tweet texts were first divided into stratified training and test sets, following the requested 60/40 split, that maintained class balance. The basic tf-idf vectorizer was used to fit and transform the training set, and then to transform the test set. It was not allowed to fit the test set to avoid data leakage. The vectorized training data was fed to a number of algorithms: random forest, gradient boosting, logistic regression and k-nearest neighbors. Additionally, ADA boosting was tried with the random forest and logistic regression classifiers. Initially, performance was compared using 4-fold cross-evaluation scoring on the stock estimators. The Matthews correlation coefficient was used for scoring, because it incorporates all of the confusion matrix information.

Gradient boosting and logistic regression, without boosting, were the best performers, with Matthews scores in the .62–.65 range. Using cluster labels instead of individuals did not improve the logistic regression results, despite reducing the number of classes by half, although it did produce about a 10% improvement for gradient boosting. A run on the test set showed that neither estimator was overfitting, so both were selected for further development.

The next iteration involved lemmatizing the tweet texts using spaCy to condense the vocabulary and maybe boost similarity. This had no significant effect on the scores, although gradient boosting proved less consistent. In light of this, logistic regression performing better on the 15-class predictions and gradient boosting consuming much more compute resources, taking orders of magnitude longer, I went ahead with logistic regression.

For the final feature combination, I used bigrams as well as single terms. I did not take the additional step of adding bigrams that matched hashtags, as I did with clustering, since hashtags vs. bigrams is a useful difference for classification purposes. Again, there was no significant gain in performance. So I went on to optimize the logistic regression algorithm on the basic tf-idf vectorization data, using a grid search over the parameter space.

As a last step, the optimization was run for different train/test splits. The model was not overly sensitive to these, although both 65/35 (best score) and 70/30 performed slightly better than 60/40. The decrease in the C parameter on the 70/30 split, however, suggested that the model might have been starting to overfit, using larger regression coefficients. The optimized model achieved a Matthews score of .69 on the holdout test set for the 15-class predictions. Using a more familiar metric, the mean F1 score for the individual classes ranged from .77 to .98, with 8 of the 15 classes scoring above .90 and only 1 scoring below .82. The ROC curves for each class were plotted together, and the ROC-AUC scores displayed:

!['ROC curves'](ROC.png)

> *The ROC curves and AUC-ROC scores shows that the optimal model performs reasonably well on classifying the tweets by individual author, considering that there are 15 classes. More than half of the classes have ROC-AUC scores of .85 or above, and only one is below .75.*

### Clustering vs classification

As part of the project idea was to compare classification vs clustering on text data, I ran a basic comparison using the adjusted Rand score to see how close clustering would come to the classification performance. I used a basic logistic regression estimator instead of the optimized one to start with, and a basic k-means clustering algorithm with the number of clusters set to 15. Running both on the basic vectorized tweet texts, the logistic regression classification achieved an adjusted Rand score of .81, while k-means clustering achieved only .10. I stopped there.

## Further steps

- For clustering authors, hashtags could be completely replaced with bigrams and mentions removed to try and cluster just on content.
- For both clustering authors and classifying tweets by author, additional preprocessing of the tweet text data, such as customizing stop words.
- Additional classification features could be engineered based on analysis of the results for the classes with lower scores.
- More generally, features based on POS tagging and other semantic approaches could be used in the classification.

## Takeaways

- Even basic NLP methods can be very effective at classification tasks given a reasonably large collection of text.
- Conversely, more complicated methods do not necessarily give better results.
- A vocabulary-based approach using tf-idf vectorization can work particularly well for Twitter data.
