---
navigation_title: "Random sampler"
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-random-sampler-aggregation.html
---

# Random sampler aggregation [search-aggregations-random-sampler-aggregation]


::::{warning}
This functionality is in technical preview and may be changed or removed in a future release. Elastic will work to fix any issues, but features in technical preview are not subject to the support SLA of official GA features.
::::


The `random_sampler` aggregation is a single bucket aggregation that randomly includes documents in the aggregated results. Sampling provides significant speed improvement at the cost of accuracy.

The sampling is accomplished by providing a random subset of the entire set of documents in a shard. If a filter query is provided in the search request, that filter is applied over the sampled subset. Consequently, if a filter is restrictive, very few documents might match; therefore, the statistics might not be as accurate.

::::{note}
This aggregation is not to be confused with the [sampler aggregation](/reference/aggregations/search-aggregations-bucket-sampler-aggregation.md). The sampler aggregation is not over all documents; rather, it samples the first `n` documents matched by the query.
::::


```console
GET kibana_sample_data_ecommerce/_search?size=0&track_total_hits=false
{
  "aggregations": {
    "sampling": {
      "random_sampler": {
        "probability": 0.1
      },
      "aggs": {
        "price_percentiles": {
          "percentiles": {
            "field": "taxful_total_price"
          }
        }
      }
    }
  }
}
```

## Top-level parameters for random_sampler [random-sampler-top-level-params]

`probability`
:   (Required, float) The probability that a document will be included in the aggregated data. Must be greater than 0, less than `0.5`, or exactly `1`. The lower the probability, the fewer documents are matched.

`seed`
:   (Optional, integer) The seed to generate the random sampling of documents. When a seed is provided, the random subset of documents is the same between calls.


## How does the sampling work? [random-sampler-inner-workings]

The aggregation is a random sample of all the documents in the index. In other words, the sampling is over the background set of documents. If a query is provided, a document is returned if it is matched by the query and if the document is in the random sampling. The sampling is not done over the matched documents.

Consider the set of documents `[1, 2, 3, 4, 5]`. Your query matches `[1, 3, 5]` and the randomly sampled set is `[2, 4, 5]`. In this case, the document returned would be `[5]`.

This type of sampling provides almost linear improvement in query latency in relation to the amount by which sampling reduces the document set size:

:::{image} images/random-sampler-agg-graph.png
:alt: Graph of the median speedup by sampling factor
:::

This graph is typical of the speed up for the majority of aggregations for a test data set of 63 million documents. The exact constants will depend on the data set size and the number of shards, but the form of the relationship between speed up and probability holds widely. For certain aggregations, the speed up may not be as dramatic. These aggregations have some constant overhead unrelated to the number of documents seen. Even for those aggregations, the speed improvements can be significant.

The sample set is generated by skipping documents using a geometric distribution (`(1-p)^(k-1)*p`) with success probability being the provided `probability` (`p` in the distribution equation). The values returned from the distribution indicate how many documents to skip in the background. This is equivalent to selecting documents uniformly at random. It follows that the expected number of failures before a success is `(1-p)/p`. For example, with the `"probability": 0.01`, the expected number of failures (or average number of documents skipped) would be `99` with a variance of `9900`. Consequently, if you had only 80 documents in your index or matched by your filter, you would most likely receive no results.

:::{image} images/relative-error-vs-doc-count.png
:alt: Graph of the relative error by sampling probability and doc count
:::

In the above image `p` is the probability provided to the aggregation, and `n` is the number of documents matched by whatever query is provided. You can see the impact of outliers on `sum` and `mean`, but when many documents are still matched at higher sampling rates, the relative error is still low.

::::{note}
This represents the result of aggregations against a typical positively skewed APM data set which also has outliers in the upper tail. The linear dependence of the relative error on the sample size is found to hold widely, but the slope depends on the variation in the quantity being aggregated. As such, the variance in your own data may cause relative error rates to increase or decrease at a different rate.
::::



## Random sampler consistency [random-sampler-consistency]

For a given `probability` and `seed`, the random sampler aggregation is consistent when sampling unchanged data from the same shard. However, this is background random sampling if a particular document is included in the sampled set or not is dependent on current number of segments.

Meaning, replica vs. primary shards could return different values as different particular documents are sampled.

If the shard changes in via doc addition, update, deletion, or segment merging, the particular documents sampled could change, and thus the resulting statistics could change.

The resulting statistics used from the random sampler aggregation are approximate and should be treated as such.


## Random sampling special cases [random-sampler-special-cases]

All counts returned by the random sampler aggregation are scaled to ease visualizations and calculations. For example, when randomly sampling a [date histogram aggregation](/reference/aggregations/search-aggregations-bucket-datehistogram-aggregation.md) every `doc_count` value for every bucket is scaled by the inverse of the random_sampler `probability` value. So, if `doc_count` for a bucket is `10,000` with `probability: 0.1`, the actual number of documents aggregated is `1,000`.

An exception to this is [cardinality aggregation](/reference/aggregations/search-aggregations-metrics-cardinality-aggregation.md). Unique item counts are not suitable for automatic scaling. When interpreting the cardinality count, compare it to the number of sampled docs provided in the top level `doc_count` within the random_sampler aggregation. It gives you an idea of unique values as a percentage of total values. It may not reflect, however, the exact number of unique values for the given field.


