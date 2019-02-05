---
title: "Data Manipulation in the Tidyverse"
author: "Benjamin W. Campbell, Senior Fellow, Program in Statistics and Methodology (PRISM)"
date: "25 September, 2018"
output:
  html_document:
    toc: true
    toc_depth: 3
    toc_float: true
    number_sections: true
---

# Introduction
Welcome to the Program in Statistics and Methodology's (PRISM) Data Manipulation in the Tidyverse Workshop.  This workshop was delivered on 25 September, 2018 as part of PRISM's regularly scheduled programming and Data Analytics Month @ Ohio State.  

## What is the Tidyverse? 

The Tidyverse is a suite of packages and tools designed to help data scientists import, clean, manipulate, and visualize their data.  The Tidyverse packages are unified by a shared philosophy of data structure ("tidy data") and programming.  The Tidyverse suite of packages consists of `readr`, `tibble`, `tidyr`, `dplyr`, `purrr`, `ggplot2`, `stringr`, and `forcats`.  The Tidyverse, itself, is also a package that can be installed and accessed.

```{r install}
set.seed(1234)
install.packages(c("tidyverse", "testit", "microbenchmark", "RMySQL"), repos = "http://cran.us.r-project.org")
library("tidyverse")
library("testit")
library("microbenchmark")
library("RMySQL")
```

## Why use the Tidyverse?
There are significant differences between how Base R and the Tidyverse handle particular tasks.  Typically, proponents of the Tidyverse argue that there are several reasons it should be used:

* Consistency across stages of data analysis, from data importation to tidying, manipulation, and visualization.  
    + Common grammar across stages.
    + Pipe operator (`%>%`).
* Parsimonious, vectorized code. 
* Very computationally efficient, often quicker than Base R.
* Has reached critical mass.
    + More downstream dependencies.
    + Many collaborators may use it (100,000+ installs).  

There are of course some drawbacks for Base R users:

* The Tidyverse packages may not behave well with other packages, it is designed to be self-contained.
* New syntax is introduced.
* New classes (`tbl`) may not behave well with other functions, unconventional row labeling behavior.

## What is our goal for today?
By the end of this workshop, it is my hope that you will feel comfortable using the Tidyverse to tidy and transform data.  You will be familiar with the basic Tidyverse packages for tidying data (`tibble`, `tidyr`) and manipulating data (`dplyr`).  

As this workshop is primarily focused on wrangling data, we will punt on data importation (`haven`, `readr`) and data visualization (`ggplot2`) in the Tidyverse. 

# Tidy Data and Tidying Data
There are two packages for producing "tidy" data within the Tidyverse framework.  The first, `tibble`, is a package introducing a new class of data frames, the `tbl`.  The second, `tidyr`, is a package designed to assist in producing "tidy" data.  Tidy data is described as data where each row is a different observation and each column is a different feature measured on each observation.

## `tibble`
### Features
The `tibble` package introduces a new class of data frame, the `tbl`, which has a lot of cool features:

* It never changes an input’s type! For example, character data are not automatically converted to factors.
* You can produce data frames with list or vector columns.
    + `tibble(x = 1:3, y = list(1:5, 1:10, 1:20))`
* Column names are preserved (periods aren't inserted in whitespace).
    + `names(data.frame('ben likes cats' = 1))` v. `names(tibble('ben likes cats' = 1))`
* Arguments are evaluated sequentially.
     + `tibble(x = 1:5, y = x - mean(x))`

There are a few things, however, that trip people up:  

* Tibbles do not have a row name feature
* Tibbles only recycle vectors of length 1. 

Here are some examples:
```{r, tbl}
# Input types retain type (no factor switching)
  # Base R data.frame
df <- data.frame(A = letters)
class(letters)
class(df$A)
  # tibble tibble
tib <- tibble(A = letters)
class(tib$A)
  # tibble data_frame
tib_df <- data_frame(A = letters)
class(tib_df$A)
# Data frames with list-columns
tibble(x = 1:3, y = list(1:5, 1:10, 1:20))
# Column naming
names(data.frame('ben likes cats' = 1))
names(tibble('ben likes cats' = 1))
# Sequential naming
tibble(x = 1:5, y = x - mean(x))
````

### Efficient Coercion
You can also coerce data to a tibble using the `as_tibble()` command.  Such a command is written explicitly to be much more efficient than the base `as.data.frame()` command.
```{r, coercion}
if (requireNamespace("microbenchmark")) {
  l <- replicate(26, sample(100), simplify = FALSE)
  names(l) <- letters
  microbenchmark::microbenchmark(
    as_tibble(l),
    as.data.frame(l)
  )
}
````

### Mangable Printing
When printed, tibbles will be much more manageable, printing only the first ten rows and all the columns that will fit on one screen.  This of course can be adjusted with a series of options:

* `options(tibble.print_max = n, tibble.print_min = m)`: if there are more than n rows, print only the first m rows. Use `options(tibble.print_max = Inf)` to show all rows, similar to the default print method for data frames.
* `options(tibble.width = Inf)` will always print all columns, regardless of the width of the screen.
```{r, print}
tibble(alphabet = LETTERS)
````

### Subsetting and Indexing
Tibbles have more consistent behavior when subsetting.  When subsetting or indexing data frames, using square brackets `[,]` may return a vector or a data frame.  However, when using square bracket indexing `[,]` for tibbles, a tibble will always be returned.  When you want a single column vector from a tibble you may use `$` or `[[]]` indexing.  
```{r, indexing}
# Indexing and subsetting using square brackets
df1 <- data.frame(x = letters, y = 1:length(letters))
class(df1[1:10,]) # indexing > 1 rows
class(df1[,1]) # indexing 1 column 
df2 <- tibble(x = letters, y = 1:length(letters))
class(df2[1:10,]) # indexing > 1 rows
class(df2[,1]) # indexing 1 column
# Indexing columns to vectors
class(df2[[1]])
class(df2$x)
````

Data frames allow for partial matching for column names using the `$` indexing method.  This does not work for tibbles.
```{r, $match}
df <- data.frame(let = letters)
df$l
df2 <- tibble(let = letters)
df2$l
````

### Recycling
Finally, tibbles are more conservative in how they handle data recycling relative to data frames.  In data frames, recycling occurs when full recycling is allowed. For example, if a data frame is to have 20 rows and for one column only 5 values are supplied, those 5 values will be repeated 4 times to populate all 20 rows. This will not work if a data frame is to have 20 rows and for one column only 6 values are supplied, as 20 divided by 6 does not produce an integer. Tibbles will only recycle values of length 1.  In other words, you cannot truly recycle values with tibbles, but you can broadcast.  
```{r, recycling}
# Tibbles and data frames allow for broadcasting, expanding b to match the length of letters
data.frame(a = letters[1:4], b = 1)
tibble(a = letters[1:4], b = 1)
# Data frames will allow for full recycling
data.frame(a = letters[1:4], b = 1:2)
# but not partial recycling
testit::has_error(data.frame(a = letters[1:4], b = 1:3))
# Neither of these will work for tibbles
testit::has_error(tibble(a = letters[1:4], b = 1:2))
testit::has_error(tibble(a = letters[1:4], b = 1:5))
````


## `tidyr`
### Tidy Data
Central to data analysis, and even data preparation, is structuring datasets to the format necessary for analysis. Hadley Wickham refers to this step as data tidying.  There are several principles of tidy data laid out in Wickham's *The Journal of Statistical Software* piece "Tidy Data."  These include:

* A dataset that is a collection of values, either stored as numbers (if quantitative) or strings (if qualitative).  
* Each value belongs to a variable, stored column-wise, or an observation, stored row-wise.  
* A variable contains all values that measure a common attribute across observations (such as GDP).
* An observation contains a unit's collection of values measured across variables. 
* Data can be relational across tables.
    + Observational units of the same type should be stored in the same table.
    + Observational units of different types should be stored in different tables.
    
Tidy data is designed to work well with vectorized languages like R because it ensures that observations' variable values are always paired.  Data is not tidy when:

* Column headers are not variable names.
* Multiple variables are stored in the same column.
* Variables are stored in rows in addition to columns.
* Observational units stored in a table are of different types.  
* A single observation is stored across multiple tables.
    
Wickham's `tidyr` package is designed to help analysts deal with untidy data and introduces three tools: gathering, separating, and spreading.  

### Gathering Data
The `gather()` command will take multiple columns and gather them into tidy data, making landscape data into portrait data. If we have columns that are not variables, then gather is useful.  Here is an example of how `gather()` is useful.  In this case, we have heart rates for 3 people over 2 treatment conditions.
```{r, gather}
messy <- tibble(
  name = c("Bob", "Peter", "Jan"),
  a = c(67, 130, 150),
  b = c(56, 121, 20)
)
messy
````

In this case we have three variables but currently we only have one of them in a column (name).  There are two other variables that we need to create: treatment and heart rate.  We can use `gather` to produce these variables and tidy the data:
```{r, gather2}
messy %>%
  gather(drug, heartrate, a:b)
````

### Separating Data
If variables are grouped together into one, we may be interested in using the `separate()` function to disentangle them.  This function is related to `extract()`, which uses regular expressions for splitting instead of pattern.  Here we have some fake data examining two outcomes observed over two different time periods according to a treatment or control condition. 
```{r, separate}
messy <- data.frame(
  id = 1:4,
  trt = sample(rep(c('control', 'treatment'), each = 2)),
  outcome1.before = runif(4),
  outcome2.before = runif(4),
  outcome1.after = runif(4),
  outcome2.after = runif(4)
)
messy
````

We will begin tidying this messy data by using the `gather()` command to turn the treatment columns into key-value paired data according to the key `id` and `time`:
```{r, gathering}
# Here we are using the pipe operator which takes an argument or object on the left and passess it as the first argument to the function on the right hand side.  
# We are specifying two new columns for key and value and specifying that the variables other than id and trt should be gathered
tidier <- messy %>%
  gather(time, value, -id, -trt)
tidier %>% head(8)
````

Then, using `separate()` we will split the key into the outcome number and the time period using regular expressions: 
```{r, separateregex}
tidy <- tidier %>%
  separate(time, into = c("outcome", "time"), sep = "\\.")
tidy %>% head(8)
````

### Spreading Data
The `spread()` function takes two columns, a key and a value, and then spreads them into multiple columns.  This takes portrait data and makes it into landscape data.  If you have variables that are in rows instead of columns, this function is particularly useful.  Here is an example:
```{r, spreading}
# Generate fake stock price data
stocks <- data.frame(
  time = as.Date('2009-01-01') + 0:9,
  X = rnorm(10, 100, 10),
  Y = rnorm(10, 125, 20),
  Z = rnorm(10, 150, 4)
)
head(stocks)
# Gather to make these into long data
stocks_long <- stocks %>% gather(stock, price, -time)
head(stocks_long)
# Make this back into wide data, which is the same as the original
stocks_wide <- stocks_long %>% spread(stock, price)
head(stocks_wide)
setequal(stocks, stocks_wide)
# We can make this even wider where the row is the stock
stocks_wider <- stocks_long %>% spread(time, price)
head(stocks_wider)
````

### Note on `reshape2`
When you're interested in the general reshaping of data, the `reshape2` package may be useful.  Given that most analysis requires tidy data, the `tidyr` package should suffice for most cases.  

# Data Manipulation
Data manipulation is a core task in data analysis.  Data manipulation refers to the process of changing or modifying data so you can efficiently extract, filter, and transform it.  In the Tidyverse, the `dplyr` package consists of five "verbs" that are useful for data manipulation: `select()`, `mutate()`, `filter()`, `arrange()`, and `summarize()`.  `dplyr` also assists in binding, grouping, and merging data, as well as accessing external databases. 

## Selecting Columns
Often we will want to select columns from a data frame.  To do this in `dplyr` there is the `select()` function which takes a table and a number of variable names.  You can also use `-` to exclude some variables, or index variables using column numbers.  
```{r, select}
data(iris)
reduced <- iris %>% select(Species, Sepal.Width)
head(reduced)
````

We can also link this with a series of helper functions to assist in selection.  This makes `select()` particularly helpful:

* `starts_with("X")`: every name that starts with "X",
* `ends_with("X")`: every name that ends with "X",
* `contains("X")`: every name that contains "X",
* `matches("X")`: every name that matches "X", where "X" can be a regular expression,
* `num_range("x", 1:5)`: the variables named x01, x02, x03, x04 and x05,
* `one_of(x)`: every name that appears in x, which should be a character vector.
```{r, helper}
# Helper functions are useful for selecting columns
s_cols <- iris %>% select(starts_with("S"))
colnames(s_cols)
dot_cols <- iris %>% select(contains("."))
colnames(dot_cols)
````

## Mutating Data
`mutate()` does the opposite of `select()`, it allows you to add new columns to a tibble.  The syntax of `mutate()` is similar to `select()`.  The first argument is a tibble, then the second is a statement about how to define the new column.  

```{r, mutate}
# You can add multiple columns at once
iris <- iris %>% mutate(Sepal.Area = Sepal.Length * Sepal.Width,
                        Petal.Area = Petal.Length * Petal.Width)
summary(iris$Sepal.Area)
summary(iris$Petal.Area)
````

## Filtering Data
`filter()` filters out rows based upon some expression.  Think about it as the row-wise function of `select()`.  We can narrow observations based upon Boolean logicals.  `filter()` will return all rows that pass the logical test that is specified.  This returns a new dataset that must be saved. 

```{r, filtering}
setosa_df <- iris %>% filter(Species == "setosa")
large_sepals <- iris %>% filter(Sepal.Area > mean(Sepal.Area))
# Let's show how we can create combine these
large_irises <- iris %>%
  mutate(Sepal.Area = Sepal.Length * Sepal.Width, Petal.Area = Petal.Length * Petal.Width) %>%
    filter(Sepal.Area > mean(Sepal.Area) & Petal.Area > mean(Petal.Area))
head(large_irises)  
````

## Arranging Data
`arrange()` reorders rows based upon certain operators.  The analyst tells the function which columns to arrange the rows by.  This will default to increasing/ascending order, but you can specify descending order using `desc()`.  You can also list a second variable as a tie breaker which will sort ties among the first variable.  For a character variable, rows will be arranged in alphabetical order.  For factors, they'll be arranged according to the factor ordering.  Let's build on the prior data frame and arrange it in descending order of sepal size using `desc()`. 

```{r, arranging}
head(large_irises)  
sorted_large_irises <- iris %>%
  mutate(Sepal.Area = Sepal.Length * Sepal.Width, Petal.Area = Petal.Length * Petal.Width) %>%
    filter(Sepal.Area > mean(Sepal.Area) & Petal.Area > mean(Petal.Area)) %>%
      arrange(desc(Sepal.Area)) # removing desc() produces ascending order
head(sorted_large_irises)  
````

## Summarizing Data
We can also use the `summarise()` or `summarize()` function to create a new dataset of summary statistics.  These functions include the same syntax as the prior commands and allows you to choose summary statistics to include in a new aggregated dataframe.  Aggregate functions should be used, these are functions that take on a vector of values and return a single number.

`dplyr` also comes with a variety of aggregate functions that are in addition to those defined in R:

* `first(x)`: The first element of vector x.
* `last(x)`: The last element of vector x.
* `nth(x, n)`: The nth element of vector x.
* `n()`: The number of rows in the data.frame or group of observations that `summarise()` describes.
* `n_distinct(x)`: The number of unique values in vector x.

```{r, summarize}
iris_summary <- iris %>%
  mutate(Sepal.Area = Sepal.Length * Sepal.Width, Petal.Area = Petal.Length * Petal.Width) %>%
    summarize(n_obs = n(),
              n_species = n_distinct(Species),
              average_sepal_area = mean(Sepal.Area),
              average_petal_area = mean(Petal.Area))
iris_summary
````


## Grouping Data
`dplyr` also has built-in means to group data.  `summarize()` returns aggregated measures for a vector or for an entire dataset.  But, what if we are interested in doing aggregate measures for groups of rows? We can look at group-based differences by using the `group_by()` function.  This function takes a tibble and some group-indicator.  The dataset will look the same, but then when you `summarise()` a grouped dataset, the cool part comes in, you get grouped aggregate statistics. 

```{r, group_by}
iris_summary <- iris %>%
  mutate(Sepal.Area = Sepal.Length * Sepal.Width, Petal.Area = Petal.Length * Petal.Width) %>%
    group_by(Species) %>% 
      summarize(n_obs = n(),
                average_sepal_area = mean(Sepal.Area),
                average_petal_area = mean(Petal.Area))
iris_summary
````


## Combining Data
### Binding Data
The easiest way to combine datasets is through binding.  Binding requires that dataframes have the same columns in the same order and meaning (row binding), or when they have the same rows and just different columns (column binding).  In base R you would typically use `rbind()` or `cbind()`.  `dplyr` has an alternative option, `bind_rows()` and `bind_cols()`.  

`bind_rows()` combines two datasets with the same columns in the same orders to merge a dataset on top of one another.  `bind_cols()` combines two datasets with the same rows in the same order to merge a dataset side by side.  This assumes that order is matching for rows or columns.  Joins solve this problem in binding by using keys, but binds require correct orders. These are quicker and return a tibble.  In addition, the syntax is a little more flexible as you can pass it lists of dataframes.  You can also add the `.id` argument that will give a new column that can help with grouping.  

```{r, binding}
# Column binding is easy!
new_vars <- tibble(variable_1 = rnorm(nrow(iris), 0, 1),
                   variable_2 = rnorm(nrow(iris), 1, 1))
iris <- bind_cols(iris, new_vars)
# Subset into list
iris_subsetted <- list(filter(iris, Species == "setosa"),
                       filter(iris, Species == "versicolor"),
                       filter(iris, Species == "virginica"))
# See length of different data frames
unlist(lapply(iris_subsetted, nrow))
# Bind rows together
iris_whole <- bind_rows(iris_subsetted)
# We can also use use purrr when we want to feed multiple objects torepeat some pipeline over an object
# This is useful when you are merging or joining multiple data frames together
library(purrr)
iris_whole_purr <- iris_subsetted %>% reduce(bind_rows)
# Objects are the same
setequal(iris_whole, iris_whole_purr)
````

## Joining Data
Often our data in R exist across multiple datasets.  As such, we will want to join datasets so that assembling datasets will be much easier and more effective. In base R there is the `merge()` function.  `dplyr`'s join functions are better than base R's `merge()` in that it preserves row order, has better syntax, and can be used for different databases.  The grammar of `dplyr`'s data joins are similar to PostgreSQL's grammar.  

### Keys
A join combines columns of two or more dataframe according to matching values in some identifier variables.  To do this, analysts identify a set of keys that exist in both dataframes and combine tables according to matching key values in both dataframes. The first table's key is the primary key and the second table's key is the foreign key.  The primary key should uniquely identify each row in a dataframe.  The foreign key only has to match with a primary key.  Sometimes you will have multiple keys.  

### Joins
The `dplyr` join functions appear mirror the PostgreSQL join function.  The primary join function in `dplyr` is the `left_join` which takes a primary table, a secondary table containing information that will be added to the primary table, and a `by` argument which takes the key that is common across data tables.  If a row in the first dataframe doesn't have a match then an NA will be added.  If an ID is in the secondary table but not in the first, then it will be dropped.  We can also specify a vector of keys.  

`right_join` treats the second table taken as the primary data frame, and ultimately does the opposite of the `left_join`.  These are designed to work for data frames, but they also works with tibbles and \code{tbl} references.  Here's an example:
```{r, leftjoins}
tb1 <- tibble(key = letters,
              variable1 = rnorm(key, 0, 1),
              variable2 = rnorm(key, 1, 1))
tb2 <- tibble(key = letters[1:10],
               variable3 = rnorm(key, 2, 1))
# Left Join: Preserve all the first table
left_joined <- left_join(tb1, tb2, by = c("key"))
head(left_joined)
nrow(left_joined)
left_joined$key
# Right Join: Preserve all the second table
right_joined <- right_join(tb1, tb2, by = c("key"))
head(right_joined)
nrow(right_joined)
right_joined$key
````


There are also some alternatives. `inner_join` is the most exclusive as it returns the rows present in both datasets.  `full_join` is the most inclusive, returning all rows that appear in either dataset.  All joins rely upon the same syntax.  
```{r, otherjoins}
tb1 <- tibble(key = c("A", "B", "C", "D", "E"),
              variable1 = rnorm(key, 0, 1),
              variable2 = rnorm(key, 1, 1))
tb2 <- tibble(key = c("B", "C", "D", "X", "Y", "Z"),
               variable3 = rnorm(key, 2, 1)) 
# Inner join: only overlapping rows (B, C, D)
inner_joined <- inner_join(tb1, tb2, by = c("key"))
head(inner_joined)
nrow(inner_joined)
inner_joined$key
# Full join: All rows included
full_joined <- full_join(tb1, tb2, by = c("key"))
head(full_joined)
nrow(full_joined)
full_joined$key
````

You use `filter()` to return a subset of rows.  But you can also use filtering joins to take two dataframes and then return a copy of the primary data frame that has been filtered, not augmented.  Using `semi_join()`, we can filter the first dataframe by whether there is a match in the second dataframe.  This is useful for filtering on complex criteria and may be easier than creating a set of filter statements.  In other words, semi joins are much more efficient:
```{r, semijoins}
# Semi join: Filter first data frame by matches in the second
# This will not include the new column
semijoined <- semi_join(tb1, tb2, by = c("key"))
````

There are also anti-joins, these are the opposite of semi-joins and return the rows in table 1 that do not match in table 2:
```{r, antijoins}
# Anti join: Records in the first table that are not in the second
# This will not include the new column
antijoined <- anti_join(tb1, tb2, by = c("key"))
````

## External Database Access
In addition to being a grammar of data manipulation, `dplyr` offers an opportunity to engage with databases that are not local.  Tabular data can be in a data table or a database, and these verbs can be used to deal with all of these different data formats.  This universality means you can store large datasets on a database and then manipulate them.  For example, if someone uses `data.table` you can still use \code{dplyr} functions on it.  You can also communicate with a foreign SQL database:
```{r, foreigndatabase}
# Set up a connection to the mysql database
my_db <- src_mysql(dbname = "dplyr", 
                   host = "courses.csrrinzqubik.us-east-1.rds.amazonaws.com", 
                   port = 3306, 
                   user = "student",
                   password = "datacamp")
# Reference a table within that source: nycflights
nycflights <- tbl(my_db, "dplyr")
# glimpse at nycflights
glimpse(nycflights)
# Ordered, grouped summary of nycflights
flight_summaries <- nycflights %>%
  group_by(carrier) %>%
    summarise(n_flights = n(), avg_delay = mean(arr_delay, na.rm = TRUE)) %>%
      arrange(avg_delay)
head(flight_summaries)
````

# Concluding Thoughts
I hope I've convinced you that the Tidyverse can be useful for data processing and manipulation! There is much more to these packages, their functionality, and the broader Tidyverse than I am able to present.  For example, check out the other packages that I haven't discussed, including `ggplot2`.  

If you have any questions, please feel free to contact me at <campbell.1721@osu.edu>!  You can also access other workshops I've conducted through [my website](https://www.benjaminwcampbell.com) or [my GitHub](https://github.com/benjamin-w-campbell).  Thank you!

