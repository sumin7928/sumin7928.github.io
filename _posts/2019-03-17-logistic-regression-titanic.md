---
layout: post
title: "Alpaca Beam"
header: "Logistic Regression with Titanic data using PySpark"
date: 2019-03-17
category: machine-learning
tags: python spark
---

This is a code along of the famous titanic dataset, its always nice to start off with this dataset because it is an example you will find across pretty much every data analysis language.
We will use the [titanic](https://www.kaggle.com/c/titanic) dataset to determine what type of person is likely to survive the Titanic crash.

## Set spark context and import data to spark RDD

First we need to create a spark session and import data into our spark RDD.

```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName('myproj').getOrCreate()
data = spark.read.csv('titanic.csv',inferSchema=True,header=True)
```

Just confirm what our schema looks like:
```python
data.printSchema()
```

    root
     |-- PassengerId: integer (nullable = true)
     |-- Survived: integer (nullable = true)
     |-- Pclass: integer (nullable = true)
     |-- Name: string (nullable = true)
     |-- Sex: string (nullable = true)
     |-- Age: double (nullable = true)
     |-- SibSp: integer (nullable = true)
     |-- Parch: integer (nullable = true)
     |-- Ticket: string (nullable = true)
     |-- Fare: double (nullable = true)
     |-- Cabin: string (nullable = true)
     |-- Embarked: string (nullable = true)
    
    

## Obtain features
Let extract the features that would actually be useful for the logistic regression model:

```python
my_cols = data.select(['Survived',
 'Pclass',
 'Sex',
 'Age',
 'SibSp',
 'Parch',
 'Fare',
 'Embarked'])
```

For now lets drop any entries with missing data:
```python
my_final_data = my_cols.na.drop()
```

## Working with Categorical Columns

Our data is not yet in a good state, there are categorical columns (Sex and Embarked) which are strings and we need to convert to numeric.
We can break this down into steps as follows using VectorAssembler, VectorIndexer, StringIndexer and OneHotEncoder:

```python
from pyspark.ml.feature import (VectorAssembler,VectorIndexer,
                                OneHotEncoder,StringIndexer)

gender_indexer = StringIndexer(inputCol='Sex',outputCol='SexIndex')
gender_encoder = OneHotEncoder(inputCol='SexIndex',outputCol='SexVec')

embark_indexer = StringIndexer(inputCol='Embarked',outputCol='EmbarkIndex')
embark_encoder = OneHotEncoder(inputCol='EmbarkIndex',outputCol='EmbarkVec')

assembler = VectorAssembler(inputCols=['Pclass','SexVec','Age','SibSp','Parch','Fare','EmbarkVec'], outputCol='features')
```
_Note that our assembler object has not yet been run on the data yet, we shall use pipelines to put our data through the above transformations and then fitting the data._


## Pipelines 

First we'll need to import the Pipeline and LogisticRegression methods and lets also create our Log regression object:

```python
from pyspark.ml import Pipeline
from pyspark.ml.classification import LogisticRegression
```

The field we care about is the "Survived" field and will be used as our label column for LogisticRegression object:
```python
log_reg_titanic = LogisticRegression(featuresCol='features',labelCol='Survived')
```

We assemble our pipeline as follows:
```python
pipeline = Pipeline(stages=[gender_indexer,embark_indexer,
                           gender_encoder,embark_encoder,
                           assembler,log_reg_titanic])
```

We will randomly split our data for our train and test data. (70:30 split) 
Using the pipeline we can fit our training data to generate a model.
```python
train_titanic_data, test_titanic_data = my_final_data.randomSplit([0.7,0.3])
fit_model = pipeline.fit(train_titanic_data)
```

Using our fitted model we can run this on our test data and obtain our predictions:
```python
results = fit_model.transform(test_titanic_data)
```
_The results variable contains our predictions_

## Evaluation

As this problem requires checking of one survives or does not survive the Titanic crash, we can use the BinaryClassificationEvaluator to evaluate our results.

Lets create en evaluator object:
```python
from pyspark.ml.evaluation import BinaryClassificationEvaluator

my_eval = BinaryClassificationEvaluator(rawPredictionCol='prediction',
                                       labelCol='Survived')
```

Lets just check what our predictions looks like vs the actual Survived field:
```python
results.select('Survived','prediction').show()
```

    +--------+----------+
    |Survived|prediction|
    +--------+----------+
    |       0|       1.0|
    |       0|       0.0|
    |       0|       1.0|
    |       0|       1.0|
    |       0|       1.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       1.0|
    |       0|       1.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    |       0|       0.0|
    +--------+----------+
    only showing top 20 rows
    
    
It's a bit of a mix just from this observation but using our evaluator object we can determine how well our model performs. 
When evaluating this will return thte AUC (Area Under Curve), a value close to 1 means our model is accurate. 
```python
my_eval.evaluate(results)
```

    0.7438423645320198

In this case the model has an about a 74% accuracy.
