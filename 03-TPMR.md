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
