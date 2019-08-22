
# Testing your `pandas` data

Adding checks to make sure that your data is sane as you prototype or engineer a solution is an easy way to make your life easier. No longer worry about back-tracking from a strange error later in your code only to find that earlier you had a bizarre and unwelcome result in your data.

By defensibly writing lightweight tests from an early point in your data transformation journey you'll save development time (keeping your overall velocity higher) and you'll make code reviews easier as your colleagues can see what your data _has to look like_ at this point in the code.

`bulwark` is suitable for use both during rapid R&D on a new dataset and all the way through to writing a properly engineered solution to a well understood problem.

# Common checks you might make

## Check for a unique index

After concatenating you might check for uniqueness in your index. During R&D it is easy to concatenate DataFrames that each have a unique index but combined have overlapping indices yielding a non-unique index in the result. Checking for uniqueness avoids such errors propagating through our code.

```
df_x = pd.DataFrame({'values': [22, 33, 44]}) 
df_y = pd.DataFrame({'values': [101, 102]}, index=[2, 3])
df_result = pd.concat((df_x, df_y)) 
ck.unique_index(df_result) # raises an exception
```

## After a `join` check that you have the same number of rows as you expect

If you `join` and you have unexpected duplicates in either DataFrame you will have additional rows in your resulting DataFrame. By doing several manipulations in a sequence you'll quickly lose track of where the error is introduced.

If you're not _expecting_ to see duplicates in either DataFrame, confirm that the resulting DataFrame has the same number of rows that you started with. You might combine this with the previous check for unique indices.

```
df_left = pd.DataFrame({'fruit': ["apple", "orange", "pear"], 
                        'colour': ['green', 'orange', 'green']}).set_index('fruit')
df_right = pd.DataFrame({'fruit': ["apple", "orange", "orange"], 
                         'price': [10, 20, 21]}).set_index('fruit')

df_result = df_left.join(df_right)
df_result
         colour  price
fruit                
apple    green   10.0
orange  orange   20.0
orange  orange   21.0
pear     green    NaN

ck.is_shape(df_result, (df_left.shape[0],-1)) # raises an exception
ck.is_shape(df_result, (df_right.shape[0],-1)) # raises an exception
```

## Before fitting in scikit-learn check for `NaN` values

`sklearn` will throw an error if you call `estimator.fit` on a matrix (which can be a Pandas Dataframe) containing `NaN` values.

```
import sklearn.linear_model, sklearn.model_selection
df = pd.DataFrame({'feature1': [1, 1, 8, np.NaN], 'target': [0, 0, 1, 1]})
X_train, X_test, y_train, y_test = sklearn.model_selection.train_test_split(df.drop(columns='target'), df['target'], random_state=0)  
clf = sklearn.linear_model.LogisticRegression()
clf.fit(X_train, y_train) # Raises: ValueError: Input contains NaN, infinity or a value too large for dtype('float64'). 
```

We could check on `df` (or on a specific matrix like `X_train`) for no NaN values to help a user focus in on the exact row that's a problem:

```
ck.has_no_nans(df) # Raises AssertionError: (3, 'feature1')
```

## Check for no `inf` or `NaN` after using numpy

NumPy can cause infinite or NaN results, `log` is a common culprit. We can check for specific conditions such as "no negative infs" in our result to stop errors propagating through our results:

```
df_customers = pd.DataFrame({'age': [23, 72, 0]}, index=['user_a', 'user_b', 'user_c'])
df_customers['age_scaled'] = df_customers['age'].apply(np.log) # causes a -inf on log(0)
ck.has_no_neg_infs(df_customers) # raises an exception
```

# Asserts

`assert` statements are "ok" if you have no other data sanity checking in place, they're easy to write and they get you started. They're also "not ok" because they can be disabled when Python is run with optimizations disabled `python -O your_script.py`.

They do combine a logic test and a useful failure message in a one-liner. Philosophically they're used for checking things that should _never_ fail at a logical level in your code so using them for data checking will look odd to many developers.

You might write something like:
`assert df['age'] >= 1, "We never expect to see 0- or negative-aged customers"`
but you'll be better off by writing a more engineered solution that can't be disabled:

```
df_customers = pd.DataFrame({'age': [23, 72, 0]}, index=['user_a', 'user_b', 'user_c'])
# We only expect positive ages in the range [1, 125]
ck.has_vals_within_range(df_customers, items={'age': [1, 72]})  # raises an exception
```

Recommendation - you should do something more credible using a tool like `bulwark`.

# Custom logic and a reporting function (NEEDS MORE THOUGHT)

We can write a test for a data quality check using conventional tools (e.g. logic, `numpy.testing` and `pandas.testing` functions) and upon failure raise a suitable exception with a message. We might raise a `ValueError` if our type is ok but our value is bad, we might raise a `TypeError` if the expected type is wrong (e.g. we got a `str` when we expected an `int`).

```
if user_age_in_years > 125:
    raise ValueError(f"The expected maximum age is exceeded, we really must validate input form data to forbid this sort of foolishness. Fix - go back into the database and fix the bad value")
```

# Rationale

This page of advice was inspired by a [tweeted conversation](https://twitter.com/ianozsvald/status/1157181430595837952) between Ian Ozsvald, Peter Wang and others. [Additional thoughts](https://twitter.com/RaquelHRibeiro/status/1158096411612913669) were provided by Raquel H Ribeiro.
