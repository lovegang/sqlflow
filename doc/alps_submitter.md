# ALPS Submitter

## Precondition
1. Custom estimator is not supported.
1. Train with evaluate is not supported.
1. Inputs needs to set the type of the dense/sparse, shape, etc.

## Data Pipeline        
Standard Select -> Train Input Table -> Parser -> Input Fn and TrainSpec

Parser is implemented in `COLUMN`, using expr like `DENSE`, `SPARSE`, `IDENTITY_SPARSE`.

Data Example
1. numeric feature：float
1. dense feature: "0.1, 2.9, 20, 13.2"
1. sparse feature:  "city: hangzhou, sex: male, interest: book"
1. identity sparse feature："1, 100, 2000, 31"

```sql
select 
  c1, c2, c3, c4, c5 as class
from kaggle_credit_fraud_training_data
TRAIN DNNClassifier
WITH
  batch_size = 512
  train_spec.max_step = 100
  ...
COLUMN
  DENSE(c1, shape=[5], dtype=float)
  SPARSE(c2, hash_bucket_size=10, dtype=float)
  IDENTITY_SPARSE(c3, shape=[2000], dtype=float)
LABEL class
...
```

ALPS submitter generates yaml format file like this: 
```text
# graph.conf

schema = [c1, c2, c3, c4]

x = [
  {feature_name: c1, type: dense, shape:[5], dtype: float, separator:","},
  {feature_name: c2, type: sparse, hash_bucket_size:10, dtype: float, separator:",", kv_separator:":"},
  {feature_name: c3, type: sparse, shape:[2000], dtype: float, separator:",", kv_separator:":"}
  {feature_name: c4, type: dense, shape:[1], dtype: float, separator: ","}
]

y = {feature_name: c5, type: dense, shape:[1], dtype: int}

batch_size = 512

TrainSpec = {
  max_step = 100
}
```
## Feature Pipeline
Feature Expr -> Semantic Analyze  -> Feature Columns Code Generation -> Estimator Args

### Feature Expr 
Tensorflow Feature Column API
>NUMERIC(key, dtype, shape)

>BUCKETIZED(source_column, boundaries)

>CATEGORICAL_IDENTITY(key, num_buckets, default_value)

>CATEGORICAL_HASH(key, hash_bucket_size, dtype)

>CATEGORICAL_VOCABULARY_LIST(key, vocabulary_list, dtype, default_value, num_oov_buckets)

>CATEGORICAL_VOCABULARY_FILE(key, vocabulary_file, vocabulary_size, num_oov_buckets, default_value, dtype)

>CROSS(keys, hash_bucket_size, hash_key)

```sql
select 
    c1, c2, c3, c4, c5 as class
from kaggle_credit_fraud_training_data
TRAIN DNNClassifier
WITH
  batch_size = 512
  train_spec.max_step = 100
  ...
COLUMN
  CROSS([NUMERIC(DENSE(c1, shape=[5], dtype=float)), BUCKETIZED(NUMERIC(DENSE(c1, shape=[5], dtype=float)), [0, 10, 100])])
LABEL class
...
```

### Semantic Analyze
Expr except for Tensorflow Feature Column API should raise error.
```sql
/* Not supported */
select * from kaggle_credit_fraud_training_data
TRAIN DNNClassifier
WITH
    ...
COLUMN
    NUMERIC(f1 * 10)
```

### Feature Columns Code Generation
```python
from alps.feature import FeatureColumnsBuilder

class CustomFCBuilder(FeatureColumnsBuilder):
  def build_feature_columns():
    /* code generated here*/
    return feature_columns_dict
```

### Estimator Args
```sql
select 
    c1, c2, c3, c4, c5 as class
from kaggle_credit_fraud_training_data
TRAIN DNNClassifier
WITH
  ...
COLUMN
    CROSS([NUMERIC(DENSE(c1, shape=[5], dtype=float)), BUCKETIZED(NUMERIC(DENSE(c1, shape=[5], dtype=float)), [0, 10, 100])])
LABEL class
...
```


The `as` syntax is temporarily not supported in `COLUMN`, so the case where Estimator requires multiple group of feature columns is not supported at this time.
```sql
select 
    c1, c2, c3, c4, c5 as class
from kaggle_credit_fraud_training_data
TRAIN DNNLinearCombinedClassifier
WITH
  linear_feature_columns = [fc1, fc2]
  dnn_feature_columns = [fc3]
  ...
COLUMN
  NUMERIC(f1) as fc1,
  BUCKETIZED(fc1, [0, 10, 100]) as fc2,
  CROSS([fc1, fc2, f3]) as fc3
LABEL class
...
```