---
title: "In between a rock and a conditional join"
author: "Adnan Fiaz"
output: 
  html_document:
    keep_md: True
---



Joining two datasets is a common action we perform in our analyses. Almost all languages have a solution for this task: R has the built-in `merge` function or the family of `join` functions in the *dplyr* package, SQL has the `JOIN` operation and Python has the `merge` function from the *pandas* package. And without a doubt these cover a variety of use cases but there's always that one exception, that one use case that isn't covered by the obvious way of doing things.

In my case this is to join two datasets based on a conditional statement. So instead of there being specific columns in both datasets that should be equal to each other I am looking to compare based on something else than equality (e.g. larger than). The following example should hopefully make things clearer. 

```
##   Record SomeValue
## 1      1        10
## 2      2         8
## 3      3        14
## 4      4         6
## 5      5         2
```

The above dataset, _myData_, is the dataset to which I want to add values from the following dataset:

```
##   ValueOfInterest LowerBound UpperBound
## 1               a          1          3
## 2               b          4          5
## 3               c         10         16
```
This second dataset, _linkTable_, is the dataset containing the information to be added to _myData_. You may notice the two dataset have no columns in common. That is because I want to join the data based on the condition that _SomeValue_ is between _LowerBound_ and _UpperBound_. This may seem like an artificial (and perhaps trivial) example but just imagine _SomeValue_ to be a date or zipcode. Then imagine the _LowerBound_ and _UpperBound_ to be bounds on a specific time period or geographical region respectively.

At this point you're probably itching to see what the answer is but you'll have to wait. In the [training courses](https://www.mango-solutions.com/data-science/r-training/courses.html) that Mango also offers one of the most important lessons we teach our participants is that the answer is just as important as *how* you obtain the answer. So i'll try to convey that here too.

## Helping you help yourself

So the first step in finding the answer is to explore R's comprehensive help system and documentation. Since we're talking about joins its only natural to look at the documentation of the `merge` function or the `join` functions from the *dplyr* package. Unfortunately both only have the option to supply columns that are compared to each other based on equality. However the documentation for the `merge` functions mentions that when no columns are given the function performs a cartesian product. That's just a seriously cool way of saying every row from _myData_ is joined with every row from _linkTable_. It might not solve the challenge but it does give me the following idea:


```r
# Attempt #1: Do a cartesian product and then filter the relevant rows
merge(myData, linkTable) %>% 
  filter(SomeValue >= LowerBound, SomeValue <= UpperBound) %>% 
  select(-LowerBound, -UpperBound)
```

```
##   Record SomeValue ValueOfInterest
## 1      5         2               a
## 2      1        10               c
## 3      3        14               c
```
You can do the above in *dplyr* as well but I'll leave that as an exercise. The bigger question is what is wrong with the above answer? You may notice that we're missing records 2 and 4. That's because these didn't satisfy the filtering condition. If we wanted to add them back in we would have to do another join. Something that you won't notice with these small example datasets is that a cartesian product is an expensive operation, combining all the records of two datasets can result in an explosion of values.  

## (Sometimes) a SQL is better than the original

When neither of the built-in functions or functions from packages you know solve the problem, the next step is to expand the search. You can directly resort to your favourite search engine (which will redirect you to Stack Overflow anyway) but it helps to first narrow the search by thinking about any possible clues. For me that clue is that joins are an important part of SQL so I searched for a [SQL solution](https://duckduckgo.com/?q=sql+between+join) that [works in R](https://duckduckgo.com/?q=sql+in+r).    

The above search directs me to the excellent *sqldf* package. This package allows you to write SQL queries and execute them using data.frames instead of tables in a database. I can thus write a SQL JOIN query with a BETWEEN clause and apply it to my two tables.


```r
library(sqldf)
sqldf('SELECT Record, SomeValue, ValueOfInterest 
      FROM myData 
      LEFT JOIN linkTable ON SomeValue BETWEEN LowerBound and UpperBound')
```

```
##   Record SomeValue ValueOfInterest
## 1      1        10               c
## 2      2         8            <NA>
## 3      3        14               c
## 4      4         6            <NA>
## 5      5         2               a
```

Marvelous! That gives me exactly the result I want and with little to no extra effort. The *sqldf* package takes the data.frames and creates corresponding tables in a temporary database (SQLite by default). It then executes the query and returns a data.frame. Even though the package isn't built for performance it handles itself quite well, even with large datasets. The only disadvantage I can think of is that you must know a bit of SQL.    

So now that I have found the answer I can continue with the next step in the analysis. That is the right thing to do but then curiousity gets the better of me and I continue to find a better solution (which obviously doesn't exist). For completeness I have listed some of these solutions below.  

## Fuzzy wuzzy join
If you widen the search for a solution you will (eventually, via various GitHub issues and StackOverflow questions) come across the [*fuzzyjoin*](https://github.com/dgrtwo/fuzzyjoin) package. If you're looking for flexible ways to join two data.frames then look no further. The package has a few ready-to-use solutions for a number of usecases: matching on equality with a tolerance (`difference_inner_join`), string matching (`stringdist_inner_join`), matching on euclidean distance (`distance_inner_join`) and many more. For my usecase I will use the more generic `fuzzy_left_join` which allows for one or more matching functions.


```r
library(fuzzyjoin)
fuzzy_left_join(myData, linkTable, 
                by=c("SomeValue"="LowerBound", "SomeValue"="UpperBound"),
                match_fun=list(`>=`, `<=`)) %>% 
  select(Record, SomeValue, ValueOfInterest)
```

```
##   Record SomeValue ValueOfInterest
## 1      1        10               c
## 2      2         8            <NA>
## 3      3        14               c
## 4      4         6            <NA>
## 5      5         2               a
```

Again, this is exactly what we're looking for. Compared to the SQL alternative it takes a little more time to figure out what is going on but that is a minor disadvantage. On the other hand, now there is no need to go back and forth with a database backend. I haven't checked what the performance differences are, that is a little out of scope for this post.  

## If not dplyr then data.table
I know it can be really annoying when someone answers your question about dplyr by saying it can be done in data.table but it's always good to keep an open mind. Especially when one solves a task the other can't (yet). It doesn't take much effort to convert from a data.frame to a data.table. From there we can use the `foverlaps` function to do a non-equi join (as it is referred to in data.table-speak). 


```r
library(data.table)
myDataDT <- data.table(myData)
myDataDT[, SomeValueHelp := SomeValue]
linkTableDT <- data.table(linkTable)
setkey(linkTableDT, LowerBound, UpperBound)

result <- foverlaps(myDataDT, linkTableDT, by.x=c('SomeValue', 'SomeValueHelp'), 
                    by.y=c('LowerBound', 'UpperBound'))
result[, .(Record, SomeValue, ValueOfInterest)]
```

```
##    Record SomeValue ValueOfInterest
## 1:      1        10               c
## 2:      2         8              NA
## 3:      3        14               c
## 4:      4         6              NA
## 5:      5         2               a
```

Ok so I'm not very well versed in the data.table way of doing things. I'm sure there is a less verbose way but this will do for now. If you know the magical spell please let us know (through the links provided at the end).

## The pythonic way

Now that we're off the tidyverse-reservoir, we might as well go all the way. During my search I also encountered a [Python solution](https://stackoverflow.com/questions/40315997/python-pandas-merge-between-condition) that looked interesting. It involves using pandas and some matrix multiplication and works as follows (yes, you can run Python code in a [RMarkdown](http://rmarkdown.rstudio.com/authoring_knitr_engines.html) document).


```python
import pandas as pd
# create the pandas Data Frames (kind of like R data.frame)
myDataDF = pd.DataFrame({'Record':range(1,6), 'SomeValue':[10, 8, 14, 6, 2]})
linkTable = pd.DataFrame({'ValueOfInterest':['a', 'b', 'c'], 'LowerBound': [1, 4, 10],
                        'UpperBound':[3, 5, 16]})
# set the index of the linkTable (kind of like setting row names)                        
linkTable = linkTable.set_index('ValueOfInterest')
# now apply a function to each row of the linkTable
# this function checks if any of the values in myData are between the upper
# and lower bound of a specific row thus returning 5 values (length of myData)
mask = linkTable.apply(lambda r: myDataDF.SomeValue.between(r['LowerBound'], 
                                r['UpperBound']), axis=1)
# mask is a 3 (length of linkTable) by 5 matrix of True/False values
# by transposing it we get the row names (the ValueOfInterest) as the column names
mask = mask.T
# we can then matrix multiply mask with its column names
myDataDF['ValueOfInterest'] = mask.dot(mask.columns)
print(myDataDF)
```

```
##    Record  SomeValue ValueOfInterest
## 0       1         10               c
## 1       2          8                
## 2       3         14               c
## 3       4          6                
## 4       5          2               a
```
This is a nice way of doing it in python but it's definitely not as readable as the SQL or fuzzyjoin alternatives. I for one had to blink at this a couple of times before I understood its magic. 

## Have no fear, the tidyverse is here










