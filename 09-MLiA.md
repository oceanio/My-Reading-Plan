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


