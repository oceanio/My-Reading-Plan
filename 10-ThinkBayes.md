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

**The Monty Hall problem**

    class Monty(Suite):
        def Likelihood(self, data, hypo):
            """Computes the likelihood of the data under the hypothesis.
            hypo: string name of the door where the prize is
            data: string name of the door Monty opened
            """
            if hypo == data:
                return 0
            elif hypo == 'A':
                return 0.5
            else:
                return 1

    def main():
        suite = Monty('ABC')
        suite.Update('B')
        suite.Print()

**The M&M problem**

    class M_and_M(Suite):
        """Map from hypothesis (A or B) to probability."""
        mix94 = dict(brown=30,
                     yellow=20,
                     red=20,
                     green=10,
                     orange=10,
                     tan=10)

        mix96 = dict(blue=24,
                     green=20,
                     orange=16,
                     yellow=14,
                     red=13,
                     brown=13)

        hypoA = dict(bag1=mix94, bag2=mix96)
        hypoB = dict(bag1=mix96, bag2=mix94)
        hypotheses = dict(A=hypoA, B=hypoB)

        def Likelihood(self, data, hypo):
            """Computes the likelihood of the data under the hypothesis.
            hypo: string hypothesis (A or B)
            data: tuple of string bag, string color
            """
            bag, color = data
            mix = self.hypotheses[hypo][bag]
            like = mix[color]
            return like

    def main():
        suite = M_and_M('AB')
        suite.Update(('bag1', 'yellow'))
        suite.Update(('bag2', 'green'))
        suite.Print()

### Chapter 03 Estimation

**The dice problem**

Suppose I have a box of dice that contains a 4-sided die, a 6-sided die, an
8-sided die, a 12-sided die, and a 20-sided die. Suppose I select a die from the box at random, roll it, and get a 6. What is the probability that I rolled each die?  

    class Dice(Suite):
    """Represents hypotheses about which die was rolled."""
        def Likelihood(self, data, hypo):
            """Computes the likelihood of the data under the hypothesis.
            hypo: integer number of sides on the die
            data: integer die roll
            """
            if hypo < data:
                return 0
            else:
                return 1.0/hypo

    def main():
        suite = Dice([4, 6, 8, 12, 20])

        suite.Update(6)
        print 'After one 6'
        suite.Print()
        """
        4 0.0
        6 0.392156862745
        8 0.294117647059
        12 0.196078431373
        20 0.117647058824
        """

        for roll in [4, 8, 7, 7, 2]:
            suite.Update(roll)
        print 'After more rolls'
        suite.Print()
        """
        4 0.0
        6 0.0
        8 0.943248453672
        12 0.0552061280613
        20 0.0015454182665
        """

**The locomotive problem**

A railroad numbers its locomotives in order 1..N. One day you
see a locomotive with the number 60. Estimate how many locomotives
the railroad has.  

幂次原理：PMF(x)和(1/x)**alpha成正比 (alpha一般接近于1.0)  

    class Train2(Dice):
        """Represents hypotheses about how many trains the company has."""

        def __init__(self, hypos, alpha=1.0):
            """Initializes the hypotheses with a power law distribution.
            hypos: sequence of hypotheses
            alpha: parameter of the power law prior
            """
            Pmf.__init__(self)
            for hypo in hypos:
                self.Set(hypo, hypo**(-alpha))
            self.Normalize()

    hypos = xrange(1, 1001)
    suite = Train2(hypos)

    dataset = [30, 60, 90]
    for data in dataset:
        suite.Update(data)

Among Bayesians, there are two approaches to choosing prior distributions: informative and uninformative.  
informative priors often seem subjective.

CDF vs PMF
We have seen two representations of distributions: Pmfs and Cdfs. These
representations are equivalent in the sense that they contain the same information,
so you can convert from one to the other. The primary difference
between them is performance: some operations are faster and easier with a
Pmf; others are faster with a Cdf.

### Chapter 04 More Estimation

**The Euro problem**

    class Euro2(thinkbayes.Suite):
        """Represents hypotheses about the probability of heads."""

        def Likelihood(self, data, hypo):
            """Computes the likelihood of the data under the hypothesis.
            hypo: integer value of x, the probability of heads (0-100)
            data: tuple of (number of heads, number of tails)
            """
            x = hypo / 100.0
            heads, tails = data
            like = x**heads * (1-x)**tails
            return like

    def TrianglePrior():
        """Makes a Suite with a triangular prior."""
        suite = Euro()
        for x in range(0, 51):
            suite.Set(x, x)
        for x in range(51, 101):
            suite.Set(x, 100-x) 
        suite.Normalize()
        return suite

    heads, tails = 140, 110
    suite.Update((heads, tails))
    print 'CI', thinkbayes.CredibleInterval(suite, 90) # The result is (51, 61).

**Cromwell’s rule**, which is the recommendation
that you should avoid giving a prior probability of 0 to any hypothesis
that is even remotely possible  

### Chapter 05 Odds and Addends

The odds form of Bayes's theorem

    o(A|D) = o(A) p(D|A)/p(D|B)

p(D|A)/p(D|B), is also called **Bayes factor**

**Oliver’s blood**

Two people have left traces of their own blood at the scene of
a crime. A suspect, Oliver, is tested and found to have type ‘O’
blood. The blood groups of the two traces are found to be of type
‘O’ (a common type in the local population, having frequency
60%) and of type ‘AB’ (a rare type, with frequency 1%). Do these
data [the traces found at the scene] give evidence in favor of the
proposition that Oliver was one of the people [who left blood at
the scene]?  

A: Oliver is one of the criminal
B: Oliver is not one of the criminal
D: the data (an O and an AB)

P(A|D)/P(B|D) = P(A)P(D|A)/P(B)P(D|B)

| hypo  | Prior | Likelihood |
| ----- | ----- | ---------- |
| A     | 1/2   | 0.01       |
| B     | 1/2   | 0.6*0.01*2 |

**Addends**: Sum of pmf

    def __add__(self, other):
        pmf = Pmf()
        for v1, p1 in self.Items():
            for v2, p2 in other.Items():
                pmf.Incr(v1+v2, p1*p2)
        return pmf

**Maxima**: max of cdf, for example, CDF(5) is the probability that a value
from this distribution is less than or equal to 5

    def Max(self, k):
        cdf = self.Copy()
        cdf.ps = [p**k for p in cdf.ps]
        return cdf

*k, 表示次数, 比如丢k次骰子，这k次中的最大值*

**Mixtures**: choose a die from the box and roll it. What is the distribution of the outcome?

    pmf_dice = thinkbayes.Pmf()
    pmf_dice.Set(Die(4), 2)
    pmf_dice.Set(Die(6), 3)
    pmf_dice.Set(Die(8), 2)
    pmf_dice.Set(Die(12), 1)
    pmf_dice.Set(Die(20), 1)
    pmf_dice.Normalize()
    mix = thinkbayes.Pmf()
    for die, weight in pmf_dice.Items():
        for outcome, prob in die.Items():
            mix.Incr(outcome, weight*prob)

### Chapter 06 Decision Analysis

**The Price is Right problem**

* Before seeing the prizes, what prior beliefs should the contestant have
about the price of the showcase?
* After seeing the prizes, how should the contestant update those beliefs?
* Based on the posterior distribution, what should the contestant bid?

The third question demonstrates a common use of Bayesian analysis: decision
analysis. Given a posterior distribution, we can choose the bid that
maximizes the contestant’s expected return.  

**Use PDF to estimate prior**

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

**建模**

对选手进行建模：

    class Player(object):
        def __init__(self, prices, bids, diffs):
            self.pdf_price = thinkbayes.EstimatedPdf(prices)
            self.cdf_diff = thinkbayes.MakeCdfFromList(diffs)
            mu = 0
            sigma = numpy.std(diffs)
            self.pdf_error = thinkbayes.GaussianPdf(mu, sigma)

Likelihood doesn’t need to compute a probability; it only has
to compute something proportional to a probability.

    def Likelihood(self, data, hypo):
        price = hypo
        guess = data
        error = price - guess
        like = self.player.ErrorDensity(error)
        return like 

通过历史数据更新模型：

    def MakeBeliefs(self, guess):
        pmf = self.PmfPrice()
        self.prior = Price(pmf, self)
        self.posterior = self.prior.Copy()
        self.posterior.Update(guess)

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

**The Boston Bruins problem**

In the 2010-11 National Hockey League (NHL) Finals, my beloved Boston
Bruins played a best-of-seven championship series against the despised
Vancouver Canucks. Boston lost the first two games 0-1 and 2-3, then won
the next two games 8-1 and 4-0. At this point in the series, what is the probability
that Boston will win the next game, and what is their probability of
winning the championship?

First, it is reasonable to believe that goal scoring in hockey is at least
approximately a Poisson process

* Use statistics from previous games to choose a prior distribution for lamda
* Use the score from the first four games to estimate lamda for each team
* Use the posterior distributions of lamda to compute distribution of goals
for each team, the distribution of the goal differential, and the probability
that each team wins the next game.
* Compute the probability that each team wins the series.

**The Prior**

根据网上的历史数据分析，The distribution is roughly Gaussian with mean 2.8 and standard
deviation 0.3.

    class Hockey(thinkbayes.Suite):
        def __init__(self):
            pmf = thinkbayes.MakeGaussianPmf(2.7, 0.3, 4)
            thinkbayes.Suite.__init__(self, pmf)

**Poisson processes**

In mathematical statistics, a **process** is a stochastic model of a physical system.

A Bernoulli process is a model of a sequence of events, called
trials, in which each trial has two possible outcomes, like success and failure.

A Poisson process is the continuous version of a Bernoulli process, where
an event can occur at any point in time with equal probability. Poisson
processes can be used to model customers arriving in a store, buses arriving
at a bus stop, or goals scored in a hockey game.

**The posteriors**

    def Likelihood(self, data, hypo):
        lam = hypo
        k = data
        like = thinkbayes.EvalPoissonPmf(lam, k)
        return like
        
    suite1 = Hockey('bruins')
    suite1.UpdateSet([0, 2, 8, 4])
    
    suite2 = Hockey('canucks')
    suite2.UpdateSet([1, 3, 1, 0])

根据更新后的概率模型，计算概率分布：

    def MakeGoalPmf(suite):
        metapmf = thinkbayes.Pmf()
        for lam, prob in suite.Items():
            pmf = thinkbayes.MakePoissonPmf(lam, 10)
            metapmf.Set(pmf, prob)
        mix = thinkbayes.MakeMixture(metapmf)
        return mix

对比赛结果进行建模：

    goal_dist1 = MakeGoalPmf(suite1)
    goal_dist2 = MakeGoalPmf(suite2)
    diff = goal_dist1 - goal_dist2
    
    p_win = diff.ProbGreater(0)
    p_loss = diff.ProbLess(0)
    p_tie = diff.Prob(0)

如果有突然死亡规则，需要对进球时间进行建模：

    def MakeGoalTimePmf(suite):
        metapmf = thinkbayes.Pmf()
        for lam, prob in suite.Items():
            pmf = thinkbayes.MakeExponentialPmf(lam, high=2, n=2001)
            metapmf.Set(pmf, prob)
        mix = thinkbayes.MakeMixture(metapmf)
        return mix

    time_dist1 = MakeGoalTimePmf(suite1)
    time_dist2 = MakeGoalTimePmf(suite2)
    
    p_tie = diff.Prob(0)
    p_overtime = thinkbayes.PmfProbLess(time_dist1, time_dist2)
    p_win = diff.ProbGreater(0) + p_tie * p_overtime

To win the series, the Bruins can either win the next two games or split the
next two and win the third. Again, we can compute the total probability:

    # win the next two
    p_series = p_win**2
    
    # split the next two, win the third
    p_series += 2 * p_win * (1-p_win) * p_win


Chapter 08 Observer Bias

**The Red Line problem**

When I arrived at the station, I could estimate the time until the next train
based on the number of passengers on the platform. If there were only a
few people, I inferred that I just missed a train and expected to wait about 7
minutes. If there were more passengers, I expected the train to arrive sooner.
But if there were a large number of passengers, I suspected that trains were
not running on schedule, so I would go back to the street level and get a
taxi. While I was waiting for trains, I thought about how Bayesian estimation
could help predict my wait time and decide when I should give up and
take a taxi.

**Observer Bias**

values from the actual distribution are oversampled in proportion to their value.
Students think that classes are bigger than they are because more of them are in the big classes.
Airline passengers think that planes are fuller than they are because more
of them are on full flights.

    def BiasPmf(pmf):
        new_pmf = pmf.Copy()
        for x, p in pmf.Items():
        new_pmf.Mult(x, x)
        new_pmf.Normalize()
        return new_pmf

**Wait Time**

Wait time, which I call y, is the time between the arrival of a passenger and
the next arrival of a train. Elapsed time, which I call x, is the time between
the arrival of the previous train and the arrival of a passenger. I chose these
definitions so that zb = x + y.  

Given the distribution of zb, we can compute the distribution of y.

    def PmfOfWaitTime(pmf_zb):
        metapmf = thinkbayes.Pmf()
        for gap, prob in pmf_zb.Items():
            uniform = MakeUniformPmf(0, gap)
            metapmf.Set(uniform, prob)
        pmf_y = thinkbayes.MakeMixture(metapmf)
        return pmf_y

    class WaitTimeCalculator(object):
        def __init__(self, pmf_z):
            self.pmf_z = pmf_z
            self.pmf_zb = BiasPmf(pmf_z)
            self.pmf_y = self.PmfOfWaitTime(self.pmf_zb)
            self.pmf_x = self.pmf_y

**Predicting Wait Time**

* Use the distribution of z to compute the prior distribution of zp, the
time between trains as seen by a passenger.
* Then we can use the number of passengers to estimate the distribution
of x, the elapsed time since the last train.
* Finally, we use the relation y = zp - x to get the distribution of y.


    wtc = WaitTimeCalculator(pmf_z)

    class ElapsedTimeEstimator(object):
        def __init__(self, wtc, lam, num_passengers):
            self.prior_x = Elapsed(wtc.pmf_x)
            self.post_x = self.prior_x.Copy()
            self.post_x.Update((lam, num_passengers))
            self.pmf_y = PredictWaitTime(wtc.pmf_zb, self.post_x)

    class Elapsed(thinkbayes.Suite):
        def Likelihood(self, data, hypo):
            x = hypo
            lam, k = data
            like = thinkbayes.EvalPoissonPmf(lam * x, k)
            return like

    def PredictWaitTime(pmf_zb, pmf_x):
        pmf_y = pmf_zb - pmf_x
        RemoveNegatives(pmf_y)
        return pmf_y
        
    def RemoveNegatives(pmf):
        for val in pmf.Values():
            if val < 0:
                pmf.Remove(val)
        pmf.Normalize()

**Estimating the arrival rate**




