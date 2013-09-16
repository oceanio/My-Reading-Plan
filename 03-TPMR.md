Data-Intensive Text Processing with MapReduce
============================

### Chapter 01 Intro

This book is about mr algo design, particularly for text processing.

### Chapter 02 MR basics

**Algo 2.1 Word Count**

    class MAPPER
        method MAP(docid a, doc d)
            for all term t in doc d do
                EMIT(term t, count 1)
                
    class REDUCER
        method REDUCE(term t, counts [c1, c2, ...])
            sum = 0
            for all count c in counts [c1, c2, ...] do
                sum += c
            EMIT(term t, count sum)
            
**The Execution Framework**

* Scheduling 

    Due to the barrier between the map and reduce tasks, the map phase of a job is only
    as fast as the slowest map task. Similarly, the completion time of a job is
    bounded by the running time of the slowest reduce task. As a result, the speed
    of a MapReduce job is sensitive to what are known as *stragglers*, or tasks
    that take an usually long time to complete.  

    In text processing we often
    observe Zipfian distributions, which means that the task or tasks responsible
    for processing the most frequent few elements will run much longer than the
    typical task. Better local aggregation, discussed in the next chapter, is one
    possible solution to this problem.  

* Data/code co-location

    To achieve data locality, the scheduler
    starts tasks on the node that holds a particular block of data (i.e., on its local
    drive) needed by the task. This has the effect of moving code to the data. If
    this is not possible (e.g., a node is already running too many tasks), new tasks
    will be started elsewhere, and the necessary data will be streamed over the
    network. An important optimization here is to prefer nodes that are on the
    same rack in the datacenter as the node holding the relevant data block, since
    inter-rack bandwidth is signicantly less than intra-rack bandwidth.
    
* Synchronization

    Note that the reduce computation cannot start until all the mappers have
    finished emitting key-value pairs and all intermediate key-value pairs have been
    shued and sorted, since the execution framework cannot otherwise guarantee
    that all values associated with the same key have been gathered. However, it is
    possible to start copying intermediate key-value pairs over the network to the
    nodes running the reducers as soon as each mapper finishes - this is a common
    optimization and implemented in Hadoop.

* Error and fault handling

    The MapReduce execution framework must accomplish
    all the tasks above in an environment where errors and faults are
    the norm, not the exception.
    
**Partitioners and Combiners**

Partitioners are responsible for dividing up the intermediate key space and
assigning intermediate key-value pairs to reducers. The simplest partitioner involves computing
the hash value of the key and then taking the mod of that value with the
number of reducers.  

Note, however, that the partitioner only considers the key and ignores the value|therefore,
a roughly-even partitioning of the key space may nevertheless yield large differences
in the number of key-values pairs sent to each reducer (since different
keys may have different numbers of associated values). This imbalance in the
amount of data associated with each key is relatively common in many text
processing applications due to the Zipfian distribution of word occurrences.

Combiners are an optimization in MapReduce that allow for local aggregation
before the shuffle and sort phase. In many cases, proper use of combiners can spell the difference between an
impractical algorithm and an efficient algorithm. It
suffices to say for now that a combiner can signifficantly reduce the amount
of data that needs to be copied over the network, resulting in much faster
algorithms.  

