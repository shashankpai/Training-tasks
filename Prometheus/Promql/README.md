# Promql

# Promql querying basics

# 1. Data Types

there are 4 data types that a promql evaluates

```
1. Instant vector
2. Scalar
3. Range vector
4. Strings
```

`Instant vector` : A set of time series containing a single sample for each time series , all sharing the same timestamp

ex - `prometheus_http_requests_total`

```
prometheus_http_requests_total{code="200",handler="/api/v1/label/:name/values",instance="localhost:9090",job="prometheus"}	2
prometheus_http_requests_total{code="200",handler="/api/v1/query",instance="localhost:9090",job="prometheus"}	4
prometheus_http_requests_total{code="200",handler="/api/v1/query_range",instance="localhost:9090",job="prometheus"}	1
prometheus_http_requests_total{code="200",handler="/graph",instance="localhost:9090",job="prometheus"}	2
prometheus_http_requests_total{code="200",handler="/metrics",instance="localhost:9090",job="prometheus"}	92
prometheus_http_requests_total{code="200",handler="/static/*filepath",instance="localhost:9090",job="prometheus"}	2
prometheus_http_requests_total{code="200",handler="/targets",instance="localhost:9090",job="prometheus"}	1
prometheus_http_requests_total{code="302",handler="/",instance="localhost:9090",job="prometheus"}	1
```

this falls under `instant vector category` as it returns 1 value of given timestamp for a given instant

`Range Vector` : A set of timeseries containing a range of datapoints over time for each time series

`prometheus_http_requests_total[1m]`

```
rometheus_http_requests_total{code="200",handler="/api/v1/label/:name/values",instance="localhost:9090",job="prometheus"}	1 @1638366514.816
1 @1638366529.816
1 @1638366544.816
prometheus_http_requests_total{code="200",handler="/api/v1/query",instance="localhost:9090",job="prometheus"}	2 @1638366514.816
4 @1638366529.816
6 @1638366544.816
prometheus_http_requests_total{code="200",handler="/api/v1/query_range",instance="localhost:9090",job="prometheus"}	1 @1638366529.816
4 @1638366544.816
prometheus_http_requests_total{code="200",handler="/graph",instance="localhost:9090",job="prometheus"}	1 @1638366514.816
1 @1638366529.816
1 @1638366544.816
prometheus_http_requests_total{code="200",handler="/metrics",instance="localhost:9090",job="prometheus"}	1 @1638366499.816
2 @1638366514.816
3 @1638366529.816
4 @1638366544.816
prometheus_http_requests_total{code="200",handler="/static/*filepath",instance="localhost:9090",job="prometheus"}	1 @1638366514.816
1 @1638366529.816
1 @1638366544.816

```

`scalar`: A simple numeric floating point value

`String` : A simple string value , currently unused



# Selectors and matchers

`filtering` of timeseries using labels is done by slectors

`process_cpu_seconds_total{job="node"}`

# Matcher types

```
Equality matcher (=)

`ex`: process_cpu_seconds_total{job="node"}

Negative equality matcher (!=)

`ex`: process_cpu_seconds_total{job!="node"}

Regular expression matcher (~=)  - Select labels that regex-match with provided string

`ex`:prometheus_http_requests_total{handler=~"/api.*"}

Negative expression matcher (!~) - Select labels that do not reg-ex match with provided string

`ex` :prometheus_http_requests_total{handler!~"/api.*"}

```

# Operators

```
1 . Binary Operators - binary operators are operators that take two operands and performs specified calculations on them
    - Arithmetic binary operator - + , - , *,/,%,^
      To find total memory available in `GB` on node 
        `node_memory_MemTotal_bytes/1024/1024`
    - Comparison binary operators -  ==, !=, >, <, >=,<=   
       `ex`: `node_cpu_seconds_total > 500`  
    - Logical set binary operators - and, or, unless 
        `ex`: vector 1 and vector2, vector1 or vector2, vector1 unless vector2
        prometheus_http_requests_total  or promhttp_metric_handler_requests_total
        prometheus_http_requests_total  and promhttp_metric_handler_requests_total


```

# on and ignore keyword

example of `ignoring`
`prometheus_http_requests_total  and ignoring(handler) promhttp_metric_handler_requests_total`

example of `on`
`prometheus_http_requests_total  and on(code) promhttp_metric_handler_requests_total`


# Aggregation operators

Aggregation operators are special mathematical functions that are used to combine information.

```
sum - calculate sum over dimensions
ex: `sum(prometheus_http_requests_total)`
     `sum(prometheus_http_requests_total) by(code)`
min - minimum over dimensions
ex: `min(node_cpu_seconds_total)` 
max - max over dimensions
ex: `max(node_cpu_seconds_total)`
avg - avg over dimensions
stddev - calculate population standard deviation over dimensions
stdvar - calculate population standard variance over dimensions
count - count number of elements in vector
ex: `count(node_cpu_seconds_total)`
count_values - count number of elements with same value
bottomk - smallest k elements by sample value
ex: to find bottom 3 modes `bottomk(3, sum(node_cpu_seconds_total) by(mode))`
topk - large k elements by sample value
 ex: To find top3 modes using topk `topk(3, sum(node_cpu_seconds_total) by(mode))`
quantile - 
```

# Functions

`rate and irate`

`rate()` : calculates the per-second average rate of increase of the timeseries in the range vector
`layman terms`: outputs the rate at which particular counter is increasing
ex: rate(prometheus_http_requests_total[1m])

`irate` : calculates the instant rate of increase of time series in range vector

ex : irate(prometheus_http_requests_total[1m])
when to use `irate` and `rate`

rate is used when you are calculating the slow moving counters, irate on the other hand should be useed when you are graphing fast or volatile moving counters

`changes`

ex : changes(process_start_time_seconds{job="node"}[1h])

`deriv` : changes in last one hour

deriv(process_resident_memory_bytes{job="prometheus"}[1h])

`predict_linear` : predicts the future value of gauge , by looking into previous pattern

predict how much memory will be free in next few hours 

`predict_linear(node_memory_MemFree_bytes{job="node"}[1h],2*60*60)/1024/1024`








