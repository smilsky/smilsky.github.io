---
title: "R Project: Moving to Dallas vs. Austin"
date: 2019-03-08
tags: [data analytics]
header:
  image: "/images/dallas-austin/dallas-austin.jpg"
excerpt: "Social Analytics, Dallas vs. Austin"
---
### Objective
The aim of this project was comparing Dallas and Austin as the cities where to move for students after graduating from college. This was done by analyzing relative reddits. Exactly the same analysis was done for the both cities, therefore, the code below is given only for one of them, unlike the images.

### Imports
```r
    library(tm)
    library(wordcloud2)
    library(syuzhet)
    library(data.table)
    library(ggplot2)
    library(RedditExtractoR)
    library(igraph)
    library(tidytext)
```
### Extracting Data
```r
    url1 = "https://www.reddit.com/r/Austin/comments/3dwb1i/to_those_that_have_moved_to_austin_from_another/"
    urldata1 = reddit_content(url1)
    austin1=urldata1["comment"]

    url2 = "https://www.reddit.com/r/Austin/comments/8ru2co/thinking_of_moving_to_austin/"
    urldata2=reddit_content(url2)
    austin2=urldata2["comment"]

    url3="https://www.reddit.com/r/Austin/comments/8qm22j/why_everyone_is_still_moving_to_austin/"
    urldata3=reddit_content(url3)
    austin3=urldata3["comment"]

    url4="https://www.reddit.com/r/Austin/comments/4712sr/tell_me_why_we_shouldnt_move_to_austin_from_nyc/"
    urldata4=reddit_content(url4)
    austin4=urldata4["comment"]

    austinDF=rbind(austin1, austin2, austin3, austin4) # creating one data frame
```
Analysis was done by using text mining libraries that are available in R. I created a corpus and a TDM and drew the conclusions from there.

### Pre-processing
```r
    austinCorp=tm_map(austinCorp, content_transformer(tolower)) # creating the corpus

    for(j in seq(austinCorp)){austinCorp[[j]]<-gsub("\n", " ", austinCorp[[j]])} # replacing empty lines with spaces
    for(j in seq(austinCorp)){austinCorp[[j]]<-gsub("\\/\\w*", " ", austinCorp[[j]])} # replacing "/" with space

    austinCorp=tm_map(austinCorp, removeNumbers) # removing numbers
    austinCorp=tm_map(austinCorp, removeWords, stopwords("english")) # removing stopwords
    austinCorp=tm_map(austinCorp, removePunctuation)
    austinCorp=tm_map(austinCorp, removeWords, c("reddit", "subreddit", "lol", "amp", "deleted", "http", "https", "like", "just", "can", "lot", "will", "get", "many", "really", "city", "one")) # removing words that do not have relevant meaning and might skew the results

    SearchReplace=content_transformer(function(x,pattern1, pattern2) gsub(pattern1,pattern2,x)) # creating a replace function
    austinCorp=tm_map(austinCorp,SearchReplace,'moved','move') # replacing same-root words with the main word
    austinCorp=tm_map(austinCorp,SearchReplace,'moving','move')

    austinTDM=removeSparseTerms(TermDocumentMatrix(austinCorp),.99) # creating a TDM with 99% frequency
```
### Most Frequent Terms
```r
    findFreqTerms(austinTDM, 1)
    austinTerms=rowSums(as.matrix(austinTDM)) # creating a data frame for the terms
    termsDF=data.frame(austinTerms)
    termsDF<- cbind(Word = rownames(termsDF), termsDF) # creating column from the index
    rownames(termsDF) <- 1:nrow(termsDF) # resetting the index
    colnames(termsDF)[colnames(termsDF) == 'austinTerms'] <- 'Occurrences' # renaming the column
    termsDF<-termsDF[order(termsDF$Occurrences, decreasing = TRUE),] # sorting in descending order  
    rownames(termsDF) <- NULL # resetting index (just for better visualization)
```
### Most Frequent Terms Wordcloud
```r
    austinSum=rowSums(as.matrix(TermDocumentMatrix(austinCorp, control=list(stopwords=T))))
    austinSum=as.data.frame(cbind(row.names(as.matrix(austinSum)),as.matrix(austinSum)))
    colnames(austinSum)=c('word','freq')
    austinSum$freq=as.numeric(as.character(austinSum$freq))
    austinSum=as.list(austinSum)

    wordcloud2(austinSum,
               size = 1,
               rotateRatio = 0,
               color= ifelse(austinSum$freq>50,'brown','orange'))
```
<img src="/images/dallas-austin/1.JPG" alt="">

### Sentiment Analysis
Next, I conducted sentiment analysis to see which sentiments are associated with moving to both cities. I used syuzhet library.
```r
    # Some prepocessing

    austinDF2=data.frame(text=sapply(austinCorp, as.character),stringsAsFactors = FALSE)
    austinDF2$text[austinDF2$text==""] <- NA
    austinDF2<- na.omit(austinDF2)
    colnames(austinDF2)<-c("Post")

    austinDF2=cbind(austinDF2,get_sentiment(austinDF2$Post)) # adding a sentiment column
    colnames(austinDF2)[colnames(austinDF2) == 'get_sentiment(austinDF2$Post)'] <- 'Sentiment' # changing column name
```
Syuzhet library gives sentiment scores to posts, but I wanted to just classify them as positive, neutral, or negative, which I did in the following way:
```r
    austinDF2$Sentiment[austinDF2$Sentiment<0]<--1
    austinDF2$Sentiment[austinDF2$Sentiment>0]<-1
    austinDF2$Sentiment[austinDF2$Sentiment==1]<-"Positive"
    austinDF2$Sentiment[austinDF2$Sentiment==-1]<-"Negative"
    austinDF2$Sentiment[austinDF2$Sentiment==0]<-"Neutral"

    austinDT=data.table(austinDF2)

    counts=austinDT[, .(number = .N), by = Sentiment]

    bargraph<-ggplot(counts, aes(x = Sentiment, y = number)) +
      geom_bar(stat = "identity", fill=c("#FC0202", "#FCE502", "#1DFC02")) + geom_text(aes(label=number), position = position_dodge(width = 1),
                                              vjust = -0.5, size = 3)
    bargraph<-bargraph + labs(x="Sentiment Types",y="Number",title="Number of Posts with Different Sentiments")
    bargraph<-bargraph + theme(axis.text.x = element_text(size  = 9,angle = 45,hjust = 1,vjust = 1))
    bargraph
```
And that gave the following results:
<img src="/images/dallas-austin/2.JPG" alt="">

The full sentiment was received by doing next (for both cities):
```r
    fullSentiment=colSums(get_nrc_sentiment(austinDF2$Post))

    fullsent<-barplot(fullSentiment,col=rainbow(20),
            xlab = "Emotion",
            ylab = "Count",
            main = "NRC Sentiments for Austin Comments")
    text(fullsent, fullSentiment + 2*sign(fullSentiment), labels=fullSentiment, pos=1, offset=-1, xpd=TRUE)
```
The given bargraph is shown below:
<img src="/images/dallas-austin/3.JPG" alt="">

### Associations
    Here I tried to find associations between the terms.
    ```r
    term = 'austin' # most frequent term
    ausnet1=as.data.frame(findAssocs(austinTDM, term, 0.40))
    ausnet1=cbind(term, rownames(ausnet1), ausnet1)
    colnames(ausnet1)=c('word1','word2','freq')
    rownames(ausnet1)=NULL

    ausnet2=ausnet1

    term = 'move'
    ausnet1=as.data.frame(findAssocs(austinTDM, term, 0.40))
    ausnet1=cbind(term, rownames(ausnet1), ausnet1)
    colnames(ausnet1)=c('word1','word2','freq')
    rownames(ausnet1)=NULL

    ausnet2=as.data.frame(rbind(ausnet1,ausnet2))

    term = 'people'
    ausnet1=as.data.frame(findAssocs(austinTDM, term, 0.40))
    ausnet1=cbind(term, rownames(ausnet1), ausnet1)
    colnames(ausnet1)=c('word1','word2','freq')
    rownames(ausnet1)=NULL

    ausnet2=as.data.frame(rbind(ausnet1,ausnet2))

    austinGraph=graph_from_data_frame(ausnet2, directed = F)
    austinGraph=simplify(austinGraph)

    mc = multilevel.community(austinGraph) # creating a multivel community for the associations

    plot(mc, austinGraph, vertex.shape = 'sphere', layout = layout.davidson.harel, vertex.size=20, vertex.label.color = 'black')
```
<img src="/images/dallas-austin/4.JPG" alt="">

### Most Frequent Word Pairs
I also tried to look at the most frequent word pairs:
```r
    bigrams <- austinDT %>%
      unnest_tokens(bigram, Post, token = "ngrams", n = 2)

    bigramDT=data.table(bigrams[,2])

    bicounts=bigramDT[, .(number = .N), by= bigram] # most frequent pairs
    bicounts=bicounts[order(-number)]

    bplot<-ggplot(data=bicounts[1:20,], aes(x = reorder(bigram, number), y=number, fill=number)) +
      geom_bar(stat="identity") + labs(x="Word Pair", y="Frequency") + coord_flip() + scale_fill_gradient(low="#E80404", high="#FFB602")
    bplot
```
<img src="/images/dallas-austin/5.JPG" alt="">

### Conclusions
Based on the above analysis, the conclusions for Austin are the following:
* More Traffic
* More Outdoorsy
* Many Californian Transplants
* Weird
* Higher Cost of Living
For Dallas the following conclusions were drawn:
* Growing Urban Core
* Less Expensive
* DFW airport
* Many comments about moving there
