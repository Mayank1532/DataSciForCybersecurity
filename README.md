# Data Science for Cybersecurity
_Open source code and resources arising from the ATI-funded Data Science for Cybersecurity project carried out at the University of Cambridge_

In order to use these scripts you need to obtain the CrimeBB database of hacking-related web forum posts.
Further information is available on the [Cambridge Cybercrime Centre website](https://www.cambridgecybercrime.uk/process.html).

### Contents
* CrimeBBprocessR: library of R scripts for CrimeBB analysis and automated labelling;
* Social_Network_Analysis: python and R scripts for Social Network Analysis and analysis of interests


#### CrimeBBprocessR

The models and heuristics which constitute the prediction functions in this library are described in our [Crime Science](https://crimesciencejournal.biomedcentral.com/articles/10.1186/s40163-018-0094-4) paper. 

It is assumed that you have a data frame of HackForums posts, obtained from CrimeBB. Other forums in the database have not been tested. Please contact Andrew (apc38 at cam dot ac dot uk) about adaptation of the tools to other datasets.

Firstly we need to install the library, which you can do from the command line with the supplied installation script (from the top directory):
```
$ Rscript installRlibrary.R
```

Then we need to collect some HackForums posts, ideally working with a whole set (or sets) of posts for a given thread ID (or IDs):

```
if (!require("pacman")) install.packages("pacman")
suppressMessages(library(pacman))
pacman::p_load(dbplyr, dplyr)
con <- src_postgres(dbname='crimebb')
post <- tbl(con, 'Post')
threadID <- 1238274  # for example
post.sub <- post %>%
  filter(Thread==threadID) %>%
  select(IdPost:Content, AuthorName) %>%
  dplyr::collect() %>% as.data.frame()
colnames(post.sub) <- c('postID', 'authorID', 'threadID', 'timestamp', 'post', 'author')
```

Then match up the posts with some info about the thread and bulletin board:
```
pacman::p_load(CrimeBBprocessR, stringr, text2vec, tidytext, nnet, tm, LiblineaR)
thread <- tbl(con, 'Thread')
thread.sub <- thread %>%
  filter(Site==0,IdThread==threadID) %>%  # HF site 0
  select(IdThread, Author:Heading) %>%
  dplyr::collect() %>% as.data.frame()
threadTitle <- ""
bboardID <- NA
if (nrow(thread.sub)>0) {
  threadTitle <- thread.sub$Heading[1]
  threadOP <- thread.sub$AuthorName[1]
  threadOPid <- thread.sub$Author[1]
  bboardID <- thread.sub$Forum[1]
} else {  # if thread ID missing from thread table, find earliest, longest post
  earliest <- post.sub[which(post.sub$timestamp==min(post.sub$timestamp)),]
  longest <- earliest[which.is.max(nchar(earliest$post)),]  # requires nnet
  threadOP <- longest$author[1]
  threadOPid <- longest$authorID[1]
}
bboardTitle <- ""
if (!is.null(bboardID)) {
  bboard <- tbl(con, 'Forum')
  bboard.sub <- bboard %>%
    filter(Site==0,IdForum==bboardID) %>%
    select(IdForum, Title) %>%
    dplyr::collect() %>% as.data.frame()
  bboardTitle <- bboard.sub$Title[1]
}
post.sub$threadTitle <- threadTitle
post.sub$threadOP <- threadOP
post.sub$threadOPid <- threadOPid
post.sub$bboardID <- bboardID
post.sub$bboardTitle <- bboardTitle
```

Your data frame is ready for processing. You should by now have the following columns: `postID, authorID, 
threadID, timestamp, post, author, threadTitle, threadOP, threadOPid, bboardID, bboardTitle`.
```
df1 <- initial_processing(post.sub)
```

Next up: sentiment analysis --
```
df2 <- sentiment_scoring(df1)
```

Predict post addressee, post type and author intent:
```
df3 <- predict_addressee(df2)
df4 <- predict_posttype(df3)
df5 <- predict_intent(df4)
```

Finally, there's a call to the function which matches reputation votes to posts. This is computationally expensive (AC intends to resolve this) and so is best called with
the 'execute' flag set to `FALSE`. This way it returns the data frame with new empty columns relating to reputation votes.
```
memb.df <- data.frame()
rep.df <- data.frame()
final.df <- repvote_matching(df5, memb.df, rep.df, execute=F)
```


_Paula Buttery, Andrew Caines, Alice Hutchings, Sergio Pastrana; March 2019_
