---
layout: post
title:  "Oracle ML with R and SQL"
date:   2021-04-06 08:52:24 +0200
categories: jekyll update
---

Oracle and R works in mysterious ways, and to get started down that path I pretty much copied a tutorial from the Oracle dev blog, altered it a little (mostly for my own learning) and wrote some notes along the way. And I added one tiny twist. All this is done on the Oracle Bigdata Lite virtual machine.

Basically, what we want to do here is the "hello world" of ML: Predict species in the Iris dataset. Not because this is particularly interesting, but because it is readily available and everybody knows it.

The general steps to do that involves the following:

1. In R, prepare the Iris dataset just a smidge, because Oracle seems to have some peculiarities.
1. From R, connect to Oracle, and write the prepared dataset to a table.
1. From R, create an ML model, and save it through ORE (we'll get back to that)
1. From SQL Developer, register a function that calls the ML model and returns the result

## The data prep

Most tasks involve a lot more dataprep than this, the one thing we need to do is to recode the "species" variable from text to numeric. Somehow, returning text from the ML model to a SQL table gave me strange errors about column lengths etc. So for the time being, we recode to numeric and wonder about the rest later.

```R
# making iris visible, and DB-izing variable names
iris <- iris
names(iris) <- c("SEPAL_LENGTH", "SEPAL_WIDTH", "PETAL_LENGTH", "PETAL_WIDTH", "SPECIES_NAME")

# Not very far under the hood, there are a lot of matrixes, so far it appears that categorical 
# response variables lead to strange errors. We recode species to numerical.

species_list <- c('setosa'=1, 'versicolor'=2, 'virginica'=3)
iris$SPECIES <- species_list[iris$SPECIES]

iris_to_db <- iris[, c("SEPAL_LENGTH", "SEPAL_WIDTH", "PETAL_LENGTH", "PETAL_WIDTH", "SPECIES")]
```

Now that we have the data frame we want, let's save it to Oracle.

## ORE, and connecting to Oracle

Connecting to a database through R can be a hassle, but usually not because of R. On our system we have ROracle and ORE set up and working. ROracle is a simple library for connecting to Oracle from R, we won't use it directly but I want to mention it is there under the hood. ORE on the other hand is a large and strange library, containing a type of hybrid interface between R and SQL. If you like either R or SQL, my guess is you are not going to like ORE. But for tasks like this, which are likely written once and run often, it may be worth accepting some of the strangeness in ORE.

```R
library(ORE)
ore.connect(user = "moviedemo", sid = "orcl", host = "localhost", 
            password = "welcome1", port = 1521, all=TRUE)

ore.create(iris_to_db, table="IRIS_R")
```

This last command simple writes the dataset to the `moviedemo` schema (since that was the user we connected with)


## Creating the model

We are creating a Naive Bayes model through oracle specific `ore.odmNB` function (next time try less oracle-y stuff)

```R

species_nb_model <- ore.odmNB(SPECIES ~ SEPAL_WIDTH +PETAL_LENGTH +PETAL_WIDTH + SEPAL_LENGTH, IRIS_R)
ore.save(species_nb_model, name="species_nb_model")

```

Let's do a dry run

```R
# dry-run: We can now use the familiar predict() function to get some predictions

r <- predict(species_nb_model, ore.frame(IRIS_R), "SPECIES")


with(r, table(SPECIES, PREDICTION))
```

We have now done what is to do in R, the rest is done from SQL Developer (or similar).

It looks quite weird, but basically registers an R-script (function) that accepts three arguments:

- dat: Actual Dataset coming from an SQL select statement (cursor)
- ds: Some settings including name of ML-function, acaing coming from an SQL select statement
- obj: A really cool strange thing basically defining the structure of the returned table, idk.


```sql
  BEGIN
sys.rqScriptCreate('iris_species_nb_script',
                   'function(dat, ds, obj) {
       ore.load(name=ds, list=obj)
       mod <- get(obj)
       dat <- ore.frame(dat)
       prd <- predict(mod, dat,supplemental.cols="SPECIES")
       ore.pull(prd)
    }');
END;
/
```

```R
ore.load(name='species_nb_model')

# This loads the actual model that was saved to oracle
mod <- get('species_nb_model')

# Just some type-conversion (or possibly superflous)
dat <- ore.frame(dat)

# The predict function we did above:
prd <- predict(mod, dat,supplemental.cols="SPECIES")

# ore.pull pulls the entire resultset into a dataframe, which gets returned from the function
ore.pull(prd)
```

Next: Selecting from this strange R-object

```sql

SELECT * FROM table(rqTableEval(
  cursor(select SEPAL_LENGTH, SEPAL_WIDTH, PETAL_LENGTH, PETAL_WIDTH, SPECIES from TEST_IRIS3),
  cursor(SELECT 'species_nb_model' as "ds", 'species_nb_model' as "obj", 1 as "ore.connect" FROM dual),
  'SELECT 1 "SETOSA", 1 "VERSICOLOR", 1 "virginica", 1 "SPECIES", 1 "PREDICTION" FROM dual',
  'iris_species_nb_script'));


```

The result here is an actual table, which is cool.We can even use it to create a view.

```sql
CREATE OR REPLACE VIEW iris_predict AS (
SELECT * FROM table(rqTableEval(
             cursor(select SEPAL_LENGTH, SEPAL_WIDTH, PETAL_LENGTH, PETAL_WIDTH, SPECIES from IRIS_R),
             cursor(SELECT 'species_nb_model' as "ds", 'species_nb_model' as "obj", 1 as "ore.connect" FROM dual),
             'SELECT 1 "SETOSA", 1 "VERSICOLOR", 1 "virginica", 1 "SPECIES", 1 "PREDICTION" FROM dual',
            'iris_species_nb_script'))
)    
;
```
