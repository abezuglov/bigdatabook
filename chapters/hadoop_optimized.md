# Hadoop Optimizations

## Combiner
A brief analysis of the word count example in MapReduce can reveal at least one area where the computation can be optimized. For each word, the mapper issues tuples like `<word,1>' that are transmitted (potentially over the network) to the reducers. As a matter of preprocessing, each node can aggregate the data output by the local mapper. This intermediate step between the mapper and reducer is what the combiner is doing. 

Suppose the mapper output is a series of tuples: <the,1>, <lock,1>,<the,1>,<series,1>. Then the combiner puts together the counts for 'the' and issues <the,2>, <lock,1>,<series,1>. For the word count problem, the combiner will look and function identically to the reducer. To 


