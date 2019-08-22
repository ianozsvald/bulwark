
# Testing your `pandas` data

Adding checks to make sure that your data is sane as you prototype or engineer a solution is an easy way to make your life easier. No longer worry about back-tracking from a strange error later in your code only to find that earlier you had a bizarre and unwelcome result in your data.

By defensibly writing lightweight tests from an early point in your data transformation journey you'll save development time (keeping your overall velocity higher) and you'll make code reviews easier as your colleagues can see what your data _has to look like_ at this point in the code.

`bulwark` is suitable for use both during rapid R&D on a new dataset and all the way through to writing a properly engineered solution to a well understood problem.

# Common checks you might make

## Check for a unique index

After concatenating you might check for uniqueness in your index. During R&D it is easy to concatenate DataFrames that each have a unique index but combined have overlapping indices yielding a non-unique index in the result.

## After a `join` check that you have the same number of rows

If you `join` and you have duplicates in either DataFrame you will have additional rows in your resulting DataFrame. By doing several manipulations in a sequence you'll quickly lose track of where the error is introduced.

If you're not _expecting_ to see duplicates in either DataFrame, confirm that the resulting DataFrame has the same number of rows that you started with.

```
In[] df_left = pd.DataFrame({'fruit': ["apple", "orange", "pear"], 'colour': ['green', 'orange', 'green']}).set_index('fruit')
In[] df_right = pd.DataFrame({'fruit': ["apple", "orange", "orange"], 'price': [10, 20, 21]}).set_index('fruit')
In[] df_left.join(df_right)

         colour  price
fruit                
apple    green   10.0
orange  orange   20.0
orange  orange   21.0
pear     green    NaN

```

## Before fitting in scikit-learn check for `NaN` values

`sklearn` will throw an error if you call `estimator.fit` on a matrix (which can be a Pandas Dataframe) containing `NaN` values.

```
example - show sklearn tiny demo off of a df
```

## Check for no `inf` or `NaN` after using numpy

```
example - show e.g. log on a series including a 0
```

# Asserts

`assert` statements are "ok" if you have no other data sanity checking in place, they're easy to write and they get you started. They're also "not ok" because they can be disabled when Python is run with optimizations disabled `python -O your_script.py`.

They do combine a logic test and a useful failure message in a one-liner. Philosophically they're used for checking things that should _never_ fail at a logical level in your code so using them for data checking will look odd to many developers.

You might write something like:
`assert df['age'] > 0, "We never expect to see 0- or negative-aged customers"`
but you'll be better off by writing a more engineered solution:
```
df_customers = pd.DataFrame({'age': [23, 72, -1]}, index=['user_a', 'user_b', 'user_c'])
# We only expect positive ages in the range [0, 125]
ck.has_vals_within_range(df_customers, items={'age': [0, 125]}) # raises an AssertionError
```

Recommendation - you should do something more credible using a tool like `bulwark`.

# Custom logic and a reporting function

We can write a test for a data quality check using conventional tools (e.g. logic, `numpy.testing` and `pandas.testing` functions) and upon failure raise a suitable exception with a message. We might raise a `ValueError` if our type is ok but our value is bad, we might raise a `TypeError` if the expected type is wrong (e.g. we got a `str` when we expected an `int`).

```
if user_age_in_years > 125:
    raise ValueError(f"The expected maximum age is exceeded, we really must validate input form data to forbid this sort of foolishness. Fix - go back into the database and fix the bad value")
```

# Rationale

This page of advice was inspired by a [tweeted conversation](https://twitter.com/ianozsvald/status/1157181430595837952) between Ian Ozsvald, Peter Wang and others. [Additional thoughts](https://twitter.com/RaquelHRibeiro/status/1158096411612913669) were provided by Raquel H Ribeiro.
