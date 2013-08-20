Notes on “Machine Learning in Action”
===========================

### Chapter 01 Machine Learning basics

>   I keep saying the sexy job in the next ten years will be statisticians. People think I’m
    joking, but who would’ve guessed that computer engineers would’ve been the sexy job of
    the 1990s? The ability to take data—to be able to understand it, to process it, to extract
    value from it, to visualize it, to communicate it—that’s going to be a hugely important
    skill in the next decades, not only at the professional level but even at the educational
    level for elementary school kids, for high school kids, for college kids. Because now we
    really do have essentially free and ubiquitous data. So the complementary scarce factor is
    the ability to understand that data and extract value from it. I think statisticians are
    part of it, but it’s just a part. You also want to be able to visualize the data,
    communicate the data, and utilize it effectively. But I do think those skills—of being
    able to access, understand, and communicate the insights you get from data analysis—
    are going to be extremely important. Managers need to be able to access and understand
    the data themselves.  
                                                    —McKinsey Quarterly, January 2009

### Chapter 02 Classifying with k-Nearest Neighbors

**k-Nearest Neighbors**

* Pros
  High accuracy, insensitive to outliers, no assumptions about data

* Cons
  Computationally expensive, requires a lot of memory

* Work with
  Numeric values, nominal values


The algorithm has to carry around the full dataset; for large datasets, this implies a large amount of storage. (Use big data??) In addition, you need to calculate the distance measurement for every piece of data in the database, and this can be cumbersome. One modification to kNN, called *kD-trees*, allows you to reduce the number of calculations.


### Chapter 03 Splitting dataset one feature at a time: decision tree

> Decision trees are often used in expert systems  

**Decision trees**

* Pros:  
  Computationally cheap to use, easy for humans to understand learned results,
  missing values OK, can deal with irrelevant features  

* Cons:  
  Prone to overfitting  

* Works with:  
  Numeric values, nominal values  

A decision tree classifier is just like a work-flow diagram with the terminating blocks representing classification decisions. Starting with a dataset, you can measure the inconsistency of a set or the entropy to find a way to split the set until all the data belongs to the same class.  

The measure of information of a set is known as the Shannon entropy. Entropy is defined as the expected value of the information. The higher the entropy, the more mixed up the data is. 

entropy = -sum(p(x)*log(p(x), 2))

The ID3 algorithm can split nominal-valued datasets. There are other decision tree–generating algorithms. The most popular are C4.5 and CART.

### Chapter 04 Classifying with probability theory: naive Bayes

**Naive Bayes**

* Pros: 
  Works with a small amount of data, handles multiple classes
* Cons: 
  Sensitive to how the input data is prepared
* Works with: 
  Nominal values

There are a number of practical considerations when implementing naïve Bayes in
a modern programming language. Underflow is one problem that can be addressed
by using the logarithm of probabilities in your calculations. The bag-of-words model is
an improvement on the set-of-words model when approaching document classification.
There are a number of other improvements, such as removing stop words, and
you can spend a long time optimizing a tokenizer.

bayes方法classifier是比较概率大小，可以将左右两边取对数，可以将乘法变为加法

    p(x|a)*p(a) > p(x|b)*p(b)  
    
    ==>  ln(p(x|a)*p(a)) > ln(p(x|b)*p(b)) 
    
    ==> ln(p(x|a)) + ln(p(a)) > ln(p(x|b)) + ln(p(b))

另外，避免概率为0，使用拉普拉斯法p(x) = (x + 1)/(N + 2)

### Chapter 05 Logistic regression

**Logistic regression**

* Pros:  
  Computationally inexpensive, easy to implement, knowledge representation easy to interpret
* Cons:  
  Prone to underfitting, may have low accuracy
* Works with:  
  Numeric values, nominal values

Logistic regression is finding best-fit parameters to a nonlinear function called the sigmoid.
Methods of optimization can be used to find the best-fit parameters. Among the
optimization algorithms, one of the most common algorithms is gradient ascent. Gradient
ascent can be simplified with stochastic gradient ascent.

Stochastic gradient ascent can do as well as gradient ascent using far fewer computing
resources. In addition, stochastic gradient ascent is an online algorithm; it can
update what it has learned as new data comes in rather than reloading all of the data
as in batch processing.

**dealing with missing values in the data**

* Use the feature’s mean value from all the available data.
* Fill in the unknown with a special value like -1.
* Ignore the instance.
* Use a mean value from similar items.
* Use another machine learning algorithm to predict the value.

### Chapter 06 Support vector machines

**Support vector machines**

* Pros:  
  Low generalization error, computationally inexpensive, easy to interpret results

* Cons:  
  Sensitive to tuning parameters and kernel choice; natively only handles binary classification

* Works with:  
  Numeric values, nominal values

The points closest to the separating hyperplane are known as *support vectors*.  

Support vector machines try to maximize margin by solving a quadratic optimization
problem. Kernel methods, or the kernel trick, map data (sometimes nonlinear data) from a
low-dimensional space to a high-dimensional space. The radial-bias function is a popular
kernel that measures the distance between two vectors.


