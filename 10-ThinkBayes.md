Think Bayes
=======================

### Chapter 01 Bayes's Therorem

**条件概率**:  P(A|B)  
**联合概率**:  P(AB) = P(A)P(B|A)  

    P(AB) = P(BA)

    P(H|D) = P(H)P(D|H)/P(D)

* P(H) is the probability of the hypothesis before we see the data, called the prior probability, or just **prior**.
* P(H|D) is what we want to compute, the probability of the hypothesis after we see the data, called the **posterior**.
* P(D|H) is the probability of the data under the hypothesis, called the **likelihood**.
* P(D) is the probability of the data under any hypothesis, called the **normalizing constant**.

Most often we simplify things by specifying a set of hypotheses that are
* **Mutually exclusive**: At most one hypothesis in the set can be true, and
* **Collectively exhaustive**: There are no other possibilities; at least one of the
hypotheses has to be true.

**The M&M problem**

In 1995, they introduced blue M&M’s. Before then, the color mix in a bag
of plain M&M’s was 30% Brown, 20% Yellow, 20% Red, 10% Green, 10%
Orange, 10% Tan. Afterward it was 24% Blue , 20% Green, 16% Orange,
14% Yellow, 13% Red, 13% Brown.  

Suppose a friend of mine has two bags of M&M’s, and he tells me that one
is from 1994 and one from 1996. He won’t tell me which is which, but he
gives me one M&M from each bag. One is yellow and one is green. What is
the probability that the yellow one came from the 1994 bag?

* A: Bag 1 is from 1994, which implies that Bag 2 is from 1996.
* B: Bag 1 is from 1996 and Bag 2 from 1994.

| hypo  | Prior | Likelihood |     | Posterior |
| ----- | ----- | ---------- | --- | --------- |
| A     | 1/2   | 20*20      | 200 | 20/27     |
| B     | 1/2   | 10*14      | 70  | 7/27      |

**The Monty Hall problem**

* Monty shows you three closed doors and tells you that there is a prize
behind each door: one prize is a car, the other two are less valuable
prizes like peanut butter and fake finger nails. The prizes are arranged
at random.

* The object of the game is to guess which door has the car. If you guess
right, you get to keep the car.

* You pick a door, which we will call Door A. We’ll call the other doors
B and C.

* Before opening the door you chose, Monty increases the suspense by
opening either Door B or C, whichever does not have the car. (If the
car is actually behind Door A, Monty can safely open B or C, so he
chooses one at random.)

* Then Monty offers you the option to stick with your original choice or
switch to the one remaining unopened door.

* D: Monty chooses Door B and there is no car there.
* A/B/C: the car is behind Door A, Door B, or Door C.

| hypo  | Prior  | Likelihood |            | Posterior |
| ----- | -----  | ---------- | ---------- | --------- |
| A     | 1/3    | 1/2        | 1/6        | 1/3       |
| B     | 1/3    | 0          | 0          | 0         |
| C     | 1/3    | 1          | 1/3        | 2/3       |

### Chapter 02 Computational Statistics

Distribution and PMF

### Chapter 03 Estimation

幂次原理：PMF(x)和(1/x)**alpha成正比 (alpha一般接近于1.0)

Among Bayesians, there are two approaches to choosing prior distributions: informative and uninformative.  
informative priors often seem subjective.

CDF vs PMF
We have seen two representations of distributions: Pmfs and Cdfs. These
representations are equivalent in the sense that they contain the same information,
so you can convert from one to the other. The primary difference
between them is performance: some operations are faster and easier with a
Pmf; others are faster with a Cdf.

### Chapter 04 More Estimation

**Cromwell’s rule**, which is the recommendation
that you should avoid giving a prior probability of 0 to any hypothesis
that is even remotely possible  

### Chapter 05 Odds and Addends

The odds form of Bayes's theorem

    o(A|D) = o(A) p(D|A)/p(D|B)

p(D|A)/p(D|B), is also called **Bayes factor**

### Chapter 06 Decision Analysis

PDF: is the continuous version of a PMF, where the possible values make up a continuous range rather than a discrete
set.  

A density is not a probability. If you integrate a density over a continuous range, the result is a probability  

KDE: is an algorithm that takes a sample and finds an appropriately smooth PDF that fits the data.  

    class EstimatedPdf(Pdf):
        def __init__(self, sample):
            self.kde = scipy.stats.gaussian_kde(sample)
        
        def Density(self, x):
            return self.kde.evaluate(x)

    prices = ReadData()
    pdf = thinkbayes.EstimatedPdf(prices)

    low, high = 0, 75000
    n = 101
    xs = numpy.linspace(low, high, n)
    pmf = pdf.MakePmf(xs)

Likelihood doesn’t need to compute a probability; it only has
to compute something proportional to a probability.

    def Likelihood(self, data, hypo):
        price = hypo
        guess = data
        error = price - guess
        like = self.player.ErrorDensity(error)
        return like 

Bayesian methods are most useful when you can carry the posterior distribution
into the next step of the analysis to perform some kind of decision analysis

    def ExpectedGain(self, bid):
        suite = self.player.posterior
        total = 0
        for price, prob in sorted(suite.Items()):
            gain = self.Gain(bid, price)
            total += prob * gain
        return total

### Chapter 07 Prediction

In mathematical statistics, a **process** is a stochastic model of a physical system.

A Bernoulli process is a model of a sequence of events, called
trials, in which each trial has two possible outcomes, like success and failure.

A Poisson process is the continuous version of a Bernoulli process, where
an event can occur at any point in time with equal probability. Poisson
processes can be used to model customers arriving in a store, buses arriving
at a bus stop, or goals scored in a hockey game.






