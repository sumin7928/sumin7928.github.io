---
layout: post
title: "Alpaca Beam"
header: "Using NLP to detect Spam and Ham using PySpark"
date: 2019-03-18
category: machine-learning
tags: python spark
---

In this particular project, I will test NLP on PySpark's MLlib libraries by building a spam filter. 

We'll use a dataset used for SMS spam detection - UCI Repository SMS Spam Detection: [https://archive.ics.uci.edu/ml/datasets/SMS+Spam+Collection](https://archive.ics.uci.edu/ml/datasets/SMS+Spam+Collection)

_Btw ham isn't spam._

## Import the data into a Spark RDD

Let's import the data and rename the columns.
```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName('nlp').getOrCreate()
data = spark.read.csv("smsspamcollection/SMSSpamCollection",inferSchema=True,sep='\t')
data = data.withColumnRenamed('_c0','class').withColumnRenamed('_c1','text')
```

Check the data contents:
```python
data.show()
```

    +-----+--------------------+
    |class|                text|
    +-----+--------------------+
    |  ham|Go until jurong p...|
    |  ham|Ok lar... Joking ...|
    | spam|Free entry in 2 a...|
    |  ham|U dun say so earl...|
    |  ham|Nah I don't think...|
    | spam|FreeMsg Hey there...|
    |  ham|Even my brother i...|
    |  ham|As per your reque...|
    | spam|WINNER!! As a val...|
    | spam|Had your mobile 1...|
    |  ham|I'm gonna be home...|
    | spam|SIX chances to wi...|
    | spam|URGENT! You have ...|
    |  ham|I've been searchi...|
    |  ham|I HAVE A DATE ON ...|
    | spam|NewMobileMovieClu...|
    |  ham|Oh k...i'm watchi...|
    |  ham|Eh u remember how...|
    |  ham|Fine if thats th...|
    | spam|England v Macedon...|
    +-----+--------------------+
    only showing top 20 rows
    

## Clean and Prepare the Data
With the current data we can create features from the text. First we can add a length feature.

```python
from pyspark.sql.functions import length

data = data.withColumn('length',length(data['text']))
data.show()
```

    +-----+--------------------+------+
    |class|                text|length|
    +-----+--------------------+------+
    |  ham|Go until jurong p...|   111|
    |  ham|Ok lar... Joking ...|    29|
    | spam|Free entry in 2 a...|   155|
    |  ham|U dun say so earl...|    49|
    |  ham|Nah I don't think...|    61|
    | spam|FreeMsg Hey there...|   147|
    |  ham|Even my brother i...|    77|
    |  ham|As per your reque...|   160|
    | spam|WINNER!! As a val...|   157|
    | spam|Had your mobile 1...|   154|
    |  ham|I'm gonna be home...|   109|
    | spam|SIX chances to wi...|   136|
    | spam|URGENT! You have ...|   155|
    |  ham|I've been searchi...|   196|
    |  ham|I HAVE A DATE ON ...|    35|
    | spam|NewMobileMovieClu...|   149|
    |  ham|Oh k...i'm watchi...|    26|
    |  ham|Eh u remember how...|    81|
    |  ham|Fine if thats th...|    56|
    | spam|England v Macedon...|   155|
    +-----+--------------------+------+
    only showing top 20 rows
    
    
What happens when we work out the average length of spam and ham text?
```python
data.groupby('class').mean().show()
```

    +-----+-----------------+
    |class|      avg(length)|
    +-----+-----------------+
    |  ham|71.48663212435233|
    | spam|138.6706827309237|
    +-----+-----------------+
    
_Hmm.. quite a difference between spam and ham just by observing the average lengths._

## Feature Transformations

We will need to import some methods to tokenize the text and remove Stop words (words that appear frequently but don't carry much meaning).

I'll use the IDF method to generate the TF-IDF ([Term frequency-inverse document frequency](https://en.wikipedia.org/wiki/Tf%E2%80%93idf)) which shows how important the word is in the document.
A pipeline can be used to do the transformations.

```python
from pyspark.ml.feature import Tokenizer, StopWordsRemover, CountVectorizer, IDF, StringIndexer

tokenizer = Tokenizer(inputCol="text", outputCol="token_text")
stopremove = StopWordsRemover(inputCol='token_text',outputCol='stop_tokens')
count_vec = CountVectorizer(inputCol='stop_tokens',outputCol='c_vec')
idf = IDF(inputCol="c_vec", outputCol="tf_idf")
ham_spam_to_num = StringIndexer(inputCol='class',outputCol='label')
```

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.linalg import Vector
from pyspark.ml import Pipeline

clean_up = VectorAssembler(inputCols=['tf_idf','length'],outputCol='features')
data_prep_pipe = Pipeline(stages=[ham_spam_to_num,tokenizer,stopremove,count_vec,idf,clean_up])
```

Put the data in the pipeline to clean the data.
```python
cleaner = data_prep_pipe.fit(data)
clean_data = cleaner.transform(data)
```

## The Model

I'll use Naive Bayes, but one can choose another model if desired.


```python
from pyspark.ml.classification import NaiveBayes

# Use defaults
nb = NaiveBayes()
```

## Training and Evaluation!

Lets restrict our data to the labels and features before fitting it in our Naive Bayes model.
```python
clean_data = clean_data.select(['label','features'])
clean_data.show()
```

    +-----+--------------------+
    |label|            features|
    +-----+--------------------+
    |  0.0|(13461,[8,12,33,6...|
    |  0.0|(13461,[0,26,308,...|
    |  1.0|(13461,[2,14,20,3...|
    |  0.0|(13461,[0,73,84,1...|
    |  0.0|(13461,[36,39,140...|
    |  1.0|(13461,[11,57,62,...|
    |  0.0|(13461,[11,55,108...|
    |  0.0|(13461,[133,195,4...|
    |  1.0|(13461,[1,50,124,...|
    |  1.0|(13461,[0,1,14,29...|
    |  0.0|(13461,[5,19,36,4...|
    |  1.0|(13461,[9,18,40,9...|
    |  1.0|(13461,[14,32,50,...|
    |  0.0|(13461,[42,99,101...|
    |  0.0|(13461,[567,1744,...|
    |  1.0|(13461,[32,113,11...|
    |  0.0|(13461,[86,224,37...|
    |  0.0|(13461,[0,2,52,13...|
    |  0.0|(13461,[0,77,107,...|
    |  1.0|(13461,[4,32,35,6...|
    +-----+--------------------+
    only showing top 20 rows
    
    

Obtain our train and test data. 70:30 split will do.
```python
training, testing = clean_data.randomSplit([0.7,0.3])
```

Fit the training data.
```python
spam_predictor = nb.fit(training)
```

Run the fitted model on our test data.
```python
test_results = spam_predictor.transform(testing)
test_results.show()
```

    +-----+--------------------+--------------------+--------------------+----------+
    |label|            features|       rawPrediction|         probability|prediction|
    +-----+--------------------+--------------------+--------------------+----------+
    |  0.0|(13461,[0,1,2,14,...|[-612.34877984332...|[0.99999999999999...|       0.0|
    |  0.0|(13461,[0,1,5,15,...|[-742.97388469249...|[1.0,5.5494439698...|       0.0|
    |  0.0|(13461,[0,1,6,16,...|[-1004.8197043274...|[1.0,5.0315468936...|       0.0|
    |  0.0|(13461,[0,1,12,34...|[-879.22017540506...|[1.0,1.0023148863...|       0.0|
    |  0.0|(13461,[0,1,15,33...|[-216.47131414494...|[1.0,1.1962236837...|       0.0|
    |  0.0|(13461,[0,1,16,21...|[-673.71050817005...|[1.0,1.5549413147...|       0.0|
    |  0.0|(13461,[0,1,22,26...|[-382.58333036006...|[1.0,1.7564627587...|       0.0|
    |  0.0|(13461,[0,1,25,66...|[-1361.5572580867...|[1.0,2.1016772175...|       0.0|
    |  0.0|(13461,[0,1,33,46...|[-378.04433557629...|[1.0,6.5844301586...|       0.0|
    |  0.0|(13461,[0,1,156,1...|[-251.74061863695...|[0.88109389963478...|       0.0|
    |  0.0|(13461,[0,1,510,5...|[-325.61601503458...|[0.99999999996808...|       0.0|
    |  0.0|(13461,[0,1,896,1...|[-96.594570068189...|[0.99999996371517...|       0.0|
    |  0.0|(13461,[0,2,3,6,7...|[-2547.2759643071...|[1.0,4.6246337876...|       0.0|
    |  0.0|(13461,[0,2,4,6,8...|[-998.45874047729...|[1.0,1.0141354676...|       0.0|
    |  0.0|(13461,[0,2,4,9,1...|[-1316.5960403060...|[1.0,4.5097599797...|       0.0|
    |  0.0|(13461,[0,2,4,28,...|[-430.36443439941...|[1.0,1.3571829152...|       0.0|
    |  0.0|(13461,[0,2,5,9,7...|[-743.88098750283...|[1.0,1.2520543556...|       0.0|
    |  0.0|(13461,[0,2,5,15,...|[-1081.6827153914...|[1.0,2.5678822847...|       0.0|
    |  0.0|(13461,[0,2,5,25,...|[-833.43388596187...|[0.99999999865682...|       0.0|
    |  0.0|(13461,[0,2,5,25,...|[-492.43437000641...|[1.0,2.6689876137...|       0.0|
    +-----+--------------------+--------------------+--------------------+----------+
    only showing top 20 rows
    
    

We will need to evaluate our test results, we can use the MulticlassClassificationEvaluator method.
```python
from pyspark.ml.evaluation import MulticlassClassificationEvaluator

acc_eval = MulticlassClassificationEvaluator()
acc = acc_eval.evaluate(test_results)
print("Accuracy of model at predicting spam was: {}".format(acc))
```

    Accuracy of model at predicting spam was: 0.9218033435242198
    
## Conclusion
The spam_predictor has an accuracy of about **92.1%**, this is quite good.

Alternatively I can try out other models to see how the accuracy fares. 
_More theory to be studied.._