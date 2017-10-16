# Chaper 4 CRUD Optimizations

## Optimizing CRUD Iterations

### Index Selectivity

Always avoid things like `COLLSCAN` and `SORT` at all costs.

Index selectivity means how accurately/selective will the fields you're building an index on narrow your query results. For example, fields with high cardinality and equality conditions will be more selective. Things like range queries for example tend to far less selective than an exact/equality match.

### Equality Sort and Range

The idea is to always the keys in your index in the order:

1. Equality conditions
2. Sort conditions
3. Range conditions

Although you may scan extra index keys this will generally help you build performant indexes for your application. It will also help avoid doing unnecessary collection scans and in memory sorts.

For example in the following query:

```

```


### Performance Tradeoffs



## Covered Queries



## Regex Performance



## Insert Performance



## Data Type Implications



## Aggregation Performance
