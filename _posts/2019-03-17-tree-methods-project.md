---
layout: post
title: "Alpaca Beam"
header: "Decision Tree Methods Project using PySpark"
date: 2019-03-17
category: machine-learning
tags: python spark
---

Just a project on how one can use random forests and decision trees to approach a problem.

__Scenario__

> You've been hired by a dog food company to try to predict why some batches of their dog food are spoiling much quicker than intended! Unfortunately this Dog Food company hasn't upgraded to the latest machinery, meaning that the amounts of the five preservative chemicals they are using can vary a lot, but which is the chemical that has the strongest effect? The dog food company first mixes up a batch of preservative that contains 4 different preservative chemicals (A,B,C,D) and then is completed with a "filler" chemical. The food scientists believe one of the A,B,C, or D preservatives is causing the problem, but need your help to figure out which one! 

Use Machine Learning with random forests to find out which parameter had the most predicitive power, thus finding out which chemical causes the early spoiling! So lets create a model and find out how we can decide which chemical is the problem!

We can try and determine the following:
* Percentage of preservative A in the mix
* Percentage of preservative B in the mix
* Percentage of preservative C in the mix
* Percentage of preservative D in the mix
* Spoiled: Label indicating whether or not the dog food batch was spoiled.

## Import data into a spark RDD
First lets import the data into a spark RDD:

```python
import findspark
findspark.init('/home/anthony/spark-2.4.0-bin-hadoop2.7')
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName('tree5').getOrCreate()
df = spark.read.csv('dog_food.csv', inferSchema=True, header=True)
```

Lets check the schema:
```python
df.printSchema()
```

    root
     |-- A: integer (nullable = true)
     |-- B: integer (nullable = true)
     |-- C: double (nullable = true)
     |-- D: integer (nullable = true)
     |-- Spoiled: double (nullable = true)
    
    

Lets also check what the data looks like too:
```python
df.head()
```

    Row(A=4, B=2, C=12.0, D=3, Spoiled=1.0)

We should also check if there are any missing data:
```python
df.describe().show()
```

    +-------+------------------+------------------+------------------+------------------+-------------------+
    |summary|                 A|                 B|                 C|                 D|            Spoiled|
    +-------+------------------+------------------+------------------+------------------+-------------------+
    |  count|               490|               490|               490|               490|                490|
    |   mean|  5.53469387755102| 5.504081632653061| 9.126530612244897| 5.579591836734694| 0.2857142857142857|
    | stddev|2.9515204234399057|2.8537966089662063|2.0555451971054275|2.8548369309982857|0.45221563164613465|
    |    min|                 1|                 1|               5.0|                 1|                0.0|
    |    max|                10|                10|              14.0|                10|                1.0|
    +-------+------------------+------------------+------------------+------------------+-------------------+
    
_Confirmed that there are no empty fields. (All fields have 490 values)_

## Create our RandomForestClassifier, DecisionTreeClassifier, GBTClassifier Classifier objects

We will use Random Forest, Decision Tree and Gradient Boosted Classifier (GBT) to create our models so we can compare later.
```python
from pyspark.ml.classification import (RandomForestClassifier, GBTClassifier, DecisionTreeClassifier)
#Decision Tree Classifier
dtc = DecisionTreeClassifier(labelCol= 'Spoiled')

#Random Forest Classifier
rfc = RandomForestClassifier(numTrees = 100, labelCol  ='Spoiled')

#Gradient boosted classifier
gbt = GBTClassifier(labelCol = 'Spoiled')
```

We will use VectorAssembler to assemble our features into a vector.
```python
from pyspark.ml.feature import VectorAssembler

assembler = VectorAssembler(inputCols = ['A','B','C','D'],outputCol = 'features')
output = assembler.transform(df)
output.head()
```

    Row(A=4, B=2, C=12.0, D=3, Spoiled=1.0, features=DenseVector([4.0, 2.0, 12.0, 3.0]))

_The features field is a DenseVector as per VectorAssembler_


## Train our models

We need to split our data into our training and test data. 70:30 split.
```python
final_data = output.select(['features','Spoiled'])
train_data, test_data = final_data.randomSplit([0.7,0.3])
```

Use the train_data to train our models.
```python
dtc_model = dtc.fit(train_data)
rfc_model = rfc.fit(train_data)
gbt_model = gbt.fit(train_data)
```

Run our trained model on our test data to obtain our predictions.
```python
dtc_preds = dtc_model.transform(test_data)
rfc_preds = rfc_model.transform(test_data)
gbt_preds = rfc_model.transform(test_data)
```

Lets see what we have:
```python
dtc_preds.show()
```

    +-------------------+-------+-------------+-----------+----------+
    |           features|Spoiled|rawPrediction|probability|prediction|
    +-------------------+-------+-------------+-----------+----------+
    |  [1.0,2.0,9.0,4.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,3.0,8.0,5.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,4.0,9.0,3.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,4.0,9.0,6.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,5.0,8.0,3.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,6.0,8.0,1.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,6.0,8.0,3.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,6.0,8.0,9.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |[1.0,6.0,11.0,10.0]|    1.0|   [0.0,76.0]|  [0.0,1.0]|       1.0|
    |  [1.0,7.0,7.0,2.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,7.0,7.0,6.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,7.0,8.0,2.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    | [1.0,7.0,11.0,9.0]|    1.0|   [0.0,76.0]|  [0.0,1.0]|       1.0|
    |  [1.0,8.0,6.0,6.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    | [1.0,8.0,7.0,10.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [1.0,8.0,8.0,8.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    | [1.0,8.0,12.0,1.0]|    1.0|   [0.0,76.0]|  [0.0,1.0]|       1.0|
    |  [1.0,9.0,7.0,5.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [2.0,1.0,7.0,9.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    |  [2.0,1.0,9.0,1.0]|    0.0|  [217.0,0.0]|  [1.0,0.0]|       0.0|
    +-------------------+-------+-------------+-----------+----------+
    only showing top 20 rows
    
    
## Using the model's .featureImportances attribute

With our models, we can use the _.featureImportances_ attribute to see which feature has the biggest impact with the models.


__Decision Tree Classifier Model:__
```python
dtc_model.featureImportances
```

    SparseVector(4, {0: 0.0052, 1: 0.0018, 2: 0.9587, 3: 0.0343})

The third feature which corresponds to **Preservative C** has the biggest impact on the model. (95%)


__Random Forest Classifier Model:__
```python
rfc_model.featureImportances
```

    SparseVector(4, {0: 0.0255, 1: 0.0244, 2: 0.9204, 3: 0.0297})

The third feature which corresponds to **Preservative C** also has the biggest impact on this model. (92%)



__Gradient Boosted Classifier Model:__
```python
gbt_model.featureImportances
```


    SparseVector(4, {0: 0.0529, 1: 0.0706, 2: 0.8345, 3: 0.0419})

The third feature which corresponds to **Preservative C** also has the biggest impact on this model. (83%)


## Conclusion

All models indicate that preservative C has higher feature importance.
