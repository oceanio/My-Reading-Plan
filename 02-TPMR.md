Data-Intensive Text Processing with MapReduce
============================

### Chapter 01 Intro

This book is about mr algo design, particularly for text processing.

### Chapter 02 MR basics

**Algo 2.1 Word Count**

    class Mapper
		method Map(docid a, doc d):
			for all term t in doc d:
				Emit(term t, count 1)

	class Reducer
		method Reduce(term t, counts [c1, c2, ...]):
			sum = 0
			for all count c 2 counts [c1, c2, ...]:
				sum = sum + c
			Emit(term t, count sum)
            
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

**HDFS**

**Hadoop Architecture**

### Chapter 03 Basic MR Alog Design

**Combiners and In-Mapper Combining**

Instead of emitting intermediate output for every input
key-value pair, the mapper aggregates partial results across multiple
input records and only emits intermediate key-value pairs after some
amount of local aggregation is performed.  

	class Mapper
		method Initialize():
			H = {}

		method Map(docid a, doc d):
			for all term t in doc d:
				H[t] = H[t] + 1
		
		method Close():
			for all term t in H:
				Emit(term t, count H[t])

there is a fundamental scalability bottleneck
associated with the in-mapper combining pattern. It critically depends
on having sufficient memory to store intermediate results until the mapper has
completely processed all key-value pairs in an input split.  

> Heap's Law, a well-known result in information retrieval, accurately models the growth of vocabulary size as a function of the collection size - the somewhat surprising fact is that the vocabulary size never stops growing.

One common solution to limiting memory usage when using the in-mapper
combining technique is to "block" input key-value pairs and "flush" in-memory
data structures periodically.  

**Algorithmic Correctness with Local Aggregation**

In general, the mean of means of arbitrary subsets of a set of numbers is not
the same as the mean of the set of numbers. So how to calculate average?

	class Mapper
		method Map(string t, integer r):
			Emit(string t, integer r)

	class Reducer
		method Reduce(string t, integers [r1, r2, ...]):
			sum = 0
			cnt = 0
			for all integer r in integers [r1, r2, ...]:
				sum = sum + r
				cnt = cnt + 1
			ravg = sum/cnt
			Emit(string t, integer ravg)

After optimization

	class Mapper
		method Initialize():
			S = {}
			C = {}

		method Map(string t, integer r):
			S[t] = S[t] + r
			C[t] = C[t] + 1

		method Close():
			for all term t in S:
				Emit(term t, pair (S[t], C[t]))

	class Reducer
		method Reduce(string t, pairs [(s1, c1), (s2, c2), ...]):
			sum = 0
			cnt = 0
			for all pair (s, c) in pairs [(s1, c1), (s2, c2), ...]:
				sum = sum + s
				cnt = cnt + c
			ravg = sum/cnt
			Emit(string t, integer ravg)

**Pairs and Stripes**

The co-occurance of keywords, The neighbors of a word can either
be defined in terms of a sliding window or some other contextual unit such as
a sentence.  

the "pairs" approach

	class Mapper
		method Map(docid a, doc d):
			for all term w in doc d:
				for all term u in Neighbors(w):
					Emit(pair (w, u), count 1)  #Emit count for each occurrence

	class Reducer
		method Reduce(pair p, counts [c1, c2, ...]):
			s = 0
			for all count c in counts [c1, c2, ...]:
				s = s + c 	#Sum co-occurrence counts
			Emit(pair p, count s)

the "stripes" approach
	
	class Mapper
		method Map(docid a, doc d):
			for all term w in doc d:
				H = {}
				for all term u in Neighbors(w):
					H[u] = H[u] + 1 	#Tally words co-occurring with w
				Emit(Term w, Stripe H)

	class Reducer
		method Reduce(term w, stripes [H1, H2, H3, ...]):
			Hf = {}
			for all stripe H in stripes [H1, H2, H3, ...]:
				Sum(Hf, H) 		#Element-wise sum
			Emit(term w, stripe Hf)

In the pairs approach, we
keep track of each joint event separately, whereas in the stripes approach
we keep track of all events that co-occur with the same event. Although
the stripes approach is signicantly more ecient, it requires memory
on the order of the size of the event space, which presents a scalability
bottleneck.

**Computing Relative Frequencies**

"Order inversion", where the main idea is to convert the sequencing of
computations into a sorting problem. Through careful orchestration, we
can send the reducer the result of a computation (e.g., an aggregate statistic)
before it encounters the data necessary to produce that computation.

* Emitting a special key-value pair for each co-occurring word pair in the
mapper to capture its contribution to the marginal.

* Controlling the sort order of the intermediate key so that the key-value
pairs representing the marginal contributions are processed by the reducer
before any of the pairs representing the joint word co-occurrence counts.

* Defining a custom partitioner to ensure that all pairs with the same left
word are shuffled to the same reducer.

* Preserving state across multiple keys in the reducer to first compute the
marginal based on the special key-value pairs and then dividing the joint
counts by the marginals to arrive at the relative frequencies.

**Secondary Sorting**

"Value-to-key conversion", which provides a scalable solution for secondary
sorting. By moving part of the value into the key, we can exploit
the MapReduce execution framework itself for sorting.

**Techniques used in controlling synchronization**

1. Constructing complex keys and values that bring together data necessary
for a computation. This is used in all of the above design patterns.

2. Executing user-specified initialization and termination code in either the
mapper or reducer. For example, in-mapper combining depends on emission
of intermediate key-value pairs in the map task termination code.

3. Preserving state across multiple inputs in the mapper and reducer. This
is used in in-mapper combining, order inversion, and value-to-key conversion.

4. Controlling the sort order of intermediate keys. This is used in order
inversion and value-to-key conversion.

5. Controlling the partitioning of the intermediate key space. This is used
in order inversion and value-to-key conversion.


### Chapter 04 Inverted Indexing for Text Retrieval

**Web Crawling**

* A web crawler must practice good "etiquette" and not overload web
servers.

* Since a crawler has finite bandwidth and resources, it must prioritize
the order in which unvisited pages are downloaded. Such decisions must
be made online and in an adversarial environment, in the sense that
spammers actively create "link farms" and "spider traps" full of spam
pages to trick a crawler into overrepresenting content from a particular
site.

* Most real-world web crawlers are distributed systems that run on clusters
of machines, often geographically distributed.

* Web content changes, but with different frequency depending on both the
site and the nature of the content.

* The web is full of duplicate content.

* The web is multilingual.

**Inverted Indexes: Baseline Implementation**

	class Mapper
		procedure Map(docid n, doc d):
			H = {}
			for all term t in doc d:
				H[t] = H[t] + 1
			for all term t in H:
				Emit(term t, posting (n,H[t]))

	class Reducer
		procedure Reduce(term t, postings [(n1, f1), (n2, f2), ...]):
			P = []
			for all posting (a, f) 2 postings [(n1, f1i, (n2, f2), ...]:
				Append(P, (a, f))
			Sort(P)
			Emit(term t, postings P)

**Inverted Indexing: Revised Implementation**

class Mapper
	method Map(docid n, doc d):
		H = {}
		for all term t in doc d:
			H[t] = H[t] + 1
		for all term t in H:
			Emit(tuple (t, n), tf H[t])

class Reducer
	method Initialize():
		tprev = None
		P = []

	method Reduce(tuple (t, n), tf [f])
		if t != tprev and tprev != None then:
			Emit(term tprev, postings P)
			P.Reset()
		P:Add((n, f))
		tprev = t

	method Close():
		Emit(term t, postings P)

**Index Compression**

**What About Retrieval?**

### Chapter 05 Graph Algorithms


