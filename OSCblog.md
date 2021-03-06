Using `R` to Compute Effect Size Confidence Intervals
========================================================
       
This is a demonstration of using `R` in the context of hypothesis testing by means of Effect Size Confidence Intervals. In other words, we'll calculate confidence intervals based on the distribution of a test statistic under the assumption that $latex H_0$ is false, the noncentral distribution of a test statistic.
  
1. [Discrete distributions: Count data and proportions](#A1)
   * Discrete CI ≠ Approximately Continuous CI
   * Noncentral CIs around the Odds Ratio of Fisher's Exact Test
   * Example: _Time, Money, and Morality_  
2. [Continuous distributions: Standardised sample means](#A2)
   * Standardised, but on which measure of dispersion?
   * Example: _Neural reactivation links unconscious thought to decision-making performance_
3. [Learn From Reproducible Open Data](#A3)
   * Example: The [ManyLabs](https://openscienceframework.org/project/WX7Ck/) project    
  
> **Notes:**   
> * New to `R`? [Here are some pointers to get you started](#A4)  
> * The code chunk below defines a function `init` that checks whether the package names passed in the list argument `need` have to be installed. These packages are required to run the code in this demonstration. After installation (if necessary) the packages are loaded.


```r
init <- function(need) {
    ip <- .packages(all.available = T)
    if (any((need %in% ip) == F)) {
        install.packages(need[!(need %in% ip)])
    }
    ok <- sapply(1:length(need), function(p) require(need[[p]], character.only = T))
}

init(c("MBESS", "metafor", "plyr"))
```

   
****
<a name="A1"></a>1. Discrete distributions: Count data
========================================================
       
### Discrete CI ≠ Approximately Continuous CI
Many studies in the social sciences compare variables across the levels of a design factor that are discrete in nature. These discrete variables often represent countable units: Participants, behaviours, correctly answered items, etc. At the aggregate level a proportion or percentage is often the object of analysis. It is common practice to disregard the discrete nature of the variable under investigation and assign probabilities to observed events using a continuous distribution function given large enough sample sizes. For example, the test for equality of sample proportions uses the standardised normal distribution (Z-score). In `R` this test can be conducted using the function `prop.test`, by default, it performs Yates' continuity correction. Depending on the situation such corrections can be either too lenient or too strict, moreover, calculating a symmetrical, continuous CI around a discrete variable is almost always inaccurate.   
   
Let's review some properties of discrete probability distributions: A distribution function that assigns a probability to the value taken on by a random variable due to the occurrence of a discrete random event is called a *Probability Mass Function*. They come in several flavours, the [binomial distribution function](http://en.wikipedia.org/wiki/Binomial_distribution) assigns probabilities to events drawn from a finite population *with* replacement. The [hypergeometric distribution function](http://en.wikipedia.org/wiki/Hypergeometric_distribution) assigns probabilities to events drawn *without* replacing them, and the [poisson distribution](http://en.wikipedia.org/wiki/Poisson_distribution)  describes independent event probabilities based on an expected value of 'successes' denoted as $latex \lambda$. 
   
These probability mass functions and their cumulative distribution function are shown in the figure below, the value with highest mass is 3 in all cases (the code to generate these figures is available in the [.Rmarkdown file](https://osf.io/vn5dz/) that generated this page).   
![plot of chunk PMF](figure/PMF.png) 

 
It is evident from the figures above that for small sample sizes the CI based on a discrete distribution is not symmetrical: These distributions are zero inflated and depending on the context, can have fat tails when large samples are concerned.    
    
Another reason to look for a more accurate CI around a discrete ES is a less than ideal analysis strategy that is often used when there are more than 2 factor levels in the design. One often calculates a $latex \chi^2 $ statistic for the large design table and subsequently interprets $latex \chi^2 $ tests on 2X2 partitions of the full design as level or group comparisons, like in a linear model. In my opinion this is an incorrect strategy: A $latex \chi^2 $ test is a goodness-of-fit test and should not be used to test hypotheses about the effects of a linear predictor on a dependent variable. One should use a generalised linear model with a poisson link (poisson regression, log-linear analysis), or binomial link (logistic, probit regression) function to answer such questions.  
    
What options are available if one does not want to embark on a generalised mixed model fitting spree?   
    
### Noncentral CIs around the Odds Ratio of Fisher's Exact Test    
    
The solution is in fact quite simple when you are using `R`. **No extra packages are required**, because the `fisher.test()` function available in the core `stats` package will give you *exactly* the effect size you need: An *exact* Odds Ratio (OR) with CIs based on the noncentral hypergeomtric distribution for 2x2 tables. The function performs Fisher's exact test, which for 2x2 tables amounts to evaluating $latex H_0: OR = 1$. If the Odds Ratio is 1, the cell counts are conditionally independent of row and column variables and one would infer: "There is no effect".   
    
You've probably read somewhere that Fisher's Exact Test can take a long time to compute when the table is very large. That is correct, but when examples of large datasets with discrete variables in recent experimental studies are considered, I would not worry too much about the computation time. Moreover, `R` only gives the noncentral CIs for 2x2 tables, therefore this specific strategy can only be applied in the following common analysis context:   
 1. When dealing with tables larger than 2x2, check dependence of row and column variables using whatever test is convenient to do so.   
 2. When it is appropriate to analyse whether variation in cell counts depends on the levels a row or column factor, use Fisher's exact test on the appropriate 2x2 sub-tables to get an exact Odds Ratio (OR).   
 3. Adjust the confidence level of the of the confidence interval around the OR estimate to take into account the number of statistical tests that were conducted on a partition of the full table: $latex CL =  \frac{(1-\alpha)}{n_{comparisons}}$   
   
> **Note:**
> The $latex OR$ is usually expressed as $latex \log(OR)$    

### Example: _Time, Money, and Morality_  

This example is taken from:  [A Post-Publication Peer-Review (3PR) of _Time, Money, and Morality_](http://anti-ism-ism.blogspot.nl/2014/01/time-money-morality-of-being-accurate.html), based on: [Gino et al., 2013](#R1)   

This study examines unethical behaviour in participants as a function of induced states of *self-interest* or *self-reflection*. One of the dependent variables is the observation of cheating behaviour in participants, in order to receive a higher financial reward for completing the experiment. The two states are induced by subconscious priming of the concept of *Money* and *Time* respectively. In addition to subconscious priming, 2 of the 4 studies also used inductions of *self-reflection* by means of framing the task as a personality vs. intelligence assessment, or by placing a mirror in the experimentation room.    
    
**Setup the data**    
The R code below creates 4 tables based on the published results of the 4 experiments (however, see link to HIBAR above for details). The columns are Cheating=YES and Cheating=NO, and rows are the conditions in each experiment.     
After each table is assigned cell counts (and row and column names), the variable name is called. Normal R behaviour is to display the contents of the variable.    
    

```r
# Experiment 1 - Cheating(YES,NO) X Prime(Money,Time,No Prime):
table.1 <- as.table(cbind(c(28, 14, 22), c(4, 19, 11)))
dimnames(table.1) <- list(Test = c("Money", "Time", "Control"), Cheating = c("CheatYES", 
    "CheatNO"))
table.1
```

```
>          Cheating
> Test      CheatYES CheatNO
>   Money         28       4
>   Time          14      19
>   Control       22      11
```

     
**Setup the analysis**   
The statistical hypotheses concern comparisons of individual conditions, in ESCI terms, one needs an effect size with confidence interval for multiple 2x2 partitions of the full design table. In experiment 1 & 4 the number of 2x2 sub-tables that are possible is 3 and in experiment 2 & 3 there are 4 such partitions of the full table. In order to take into account that we are questioning a random sample more than once, a correction of the $latex \alpha$ level is appropriate and this implies a CI of 0.9833 for experiment 1.   
     
The 2x2 partitions are easy to create in R. For instance, to compare row 1 and 3 of table.1 created above one can us the following:

```r
# Subset a table by calling specific row numbers: [c(1,3),1:2] Omitting
# column IDs [c(1,3),] will return all columns
table.1[c(1, 3), ]
```

```
>          Cheating
> Test      CheatYES CheatNO
>   Money         28       4
>   Control       22      11
```

> Note: It often pays off to take the time to create sensible row and column labels; it is immediately clear which comparison is up for analysis.
    
The function `fisher.test()` can be called using this syntax and  we need just one extra argument which is the desired confidence level. The code below clearly contains repetitions, the test is conducted 14 times! Experienced R programmers will shake their head and maybe even shed a tear, but the purpose here is to give an idea of what it is we are calculating and how those results are converted into a plot.    


```r
# Create confidence levels adjusted for comparisons
CL <- (1 - 0.05/3)

# Perform Fisher's Exact Test on the appropriate 2x2 table
c1.1 <- fisher.test(table.1[c(1, 2), ], conf.level = CL)
c1.2 <- fisher.test(table.1[c(1, 3), ], conf.level = CL)
c1.3 <- fisher.test(table.1[c(2, 3), ], conf.level = CL)
```


The output of the `fisher.test()` function is a list with different kinds of values, the OR estimates, the CIs and the exact p-values. The code below creates a data frame called `ORcomp`, it will be used to plot the results.
    
**Setup the plot**    

```r
# Create a data frame
ORcomp <- data.frame(cbind(comparison = 1:3, log(rbind(c1.1$estimate, c1.2$estimate, 
    c1.3$estimate)), log(rbind(c1.1$conf.int, c1.2$conf.int, c1.3$conf.int)), 
    round(rbind(c1.1$p.value, c1.2$p.value, c1.3$p.value), digits = 4)))

# Some columns do not have a name yet...
names(ORcomp)[3] <- "lo"
names(ORcomp)[4] <- "hi"
names(ORcomp)[5] <- "p"

# Create a factor with labels to indicate which comparison we are dealing
# with
ORcomp$comparison <- factor(ORcomp$comparison, labels = c("EXP1: Money-Time", 
    "EXP1: Money-Control", "EXP1: Time-Control"))

# Show the data frame
ORcomp
```

```
>            comparison odds.ratio      lo     hi      p
> 1    EXP1: Money-Time     2.2124  0.6511 4.1468 0.0002
> 2 EXP1: Money-Control     1.2336 -0.3955 3.1833 0.0759
> 3  EXP1: Time-Control    -0.9827 -2.3550 0.3208 0.0828
```

   
**A forest (the cure to publication bias?)**    
All that is left is to create a plot of the results.
The function `forest()` from the package `metafor` is used to create a so called forest plot, the Bonferroni adjusted p-values of the exact test will be added to the plot.


```r
# Bonferroni adjustment of the exact p-value
adj <- c(rep(3, 3)) * ORcomp$p
adj[adj > 1] <- 1

# Plot the results stored in data frame ORcomp
forest(x = ORcomp$odds.ratio, ci.lb = ORcomp$lo, ci.ub = ORcomp$hi, slab = ORcomp$comparison, 
    ilab = adj, ilab.xpos = 5, alim = c(-3.5, 4.5), main = "Forest Plot", xlim = c(-7, 
        9), xlab = "Exact log Odds Ratio", efac = 1, cex = 0.9, mgp = c(1, 1, 
        0))

# Add some text
text(7.5, (nrow(ORcomp) + 1.5), "log OR [CI.9833]")
text(-4.4, (nrow(ORcomp) + 1.5), "Contrasts comparing Cheating frequency")
text(5, (nrow(ORcomp) + 1.5), "Adjusted p")
```

![plot of chunk ORCI_Forest](figure/ORCI_Forest.png) 

    
Only the first comparison does not include 0. In the original article, the number of significant contrasts reported in 14 comparisons of occurrence of cheating behaviour conducted in 4 samples, was 9. This was based on a continuous test of sample proportions, without correcting for multiple comparisons. Both in R and in SPSS (for 2x2 tables), this is a deliberate choice, because the default behaviour is to apply Yates' correction. If the continuous test is continuity corrected and a Bonferroni adjustment is applied, 2 significant results remain in the entire article. The ESCI hypothesis test conducted for all 4 experiments gives the same result: Just 2 intervals remain that do not include log(1)=0.
    
****
<a name="A2"></a>2. Continuous distributions: Standardised sample means
==========================================================

### Standardised, but on which measure of dispersion?
When continuous variables are compared across different independent samples, the effect size of interest is often the standardised mean difference, also known as $latex \text{Cohen's d} = \frac{\bar{X_1}-\bar{X_2}}{s}$. One source of confusion that can arise when comparing this effect size between studies, is that different dispersion measures may be used to standardise the mean difference. In the equation, $latex s$ can mean at least two things: A reference sample's standard deviation (e.g. a control group), or a pooled standard deviation.   

   
It is of course important to know which dispersion measure was used to calculate *Cohen's d* in order to build a CI around it. The R package [`MBESS`](http://nd.edu/~kkelley/site/MBESS.html) contains separate functions to deal with each situation:
 1. `ci.sm()` Confidence interval for the **s**tandardised **m**ean: $latex \frac{\bar{X}-\mu}{SD_X}$ 
 2. `ci.smd()` Confidence interval for the **s**tandardised **m**ean **d**ifference: $latex \frac{\bar{X_1} - \bar{X_2}}{SD_{pooled}}$   
 3. `ci.smd.c()` Confidence interval for the **c**ontrol **s**tandardised **m**ean **d**ifference: $latex \frac{\bar{X_C}-\bar{X_E}}{SD_C}$  
   
   
Most `MBESS` functions allow different sets of input arguments to calculate the CIs:
 1. Based on sample descriptives:
  * `ci.sm(Mean = , SD = , N = )`, or: `ci.sm(sm = , N = )` 
  * `ci.smd(smd = , n.1 = , n.2 = )`
  * `ci.smd.c(smd.c = , n.C = , n.E = )`
 2. Based on the estimated non-centrality parameter (value of the test statistic, most often *Student's t* assuming homogeneity of variance):
  * `ci.sm(ncp = , N = )`
  * `ci.smd(ncp = , n.1 = , n.2 = )`
  * `ci.smd.c(ncp = , n.C = , n.E = )`
   
The confidence interval coverage ($latex 1 - \alpha$) is .95 by default, this can be adjusted:
 1. Symmetrically, e.g., `ci.sm(sm = 2.1, N = 15, conf.level = .99)`
 2. Asymmetrically, separate $latex \alpha$ for lower and upper bound of the CI, e.g., `ci.smd(ncp = 3.3, n.1 = 20, n.2 = 19, alpha.lower = 0, alpha.upper = .05)`
 
 
### Example: _Neural reactivation links unconscious thought to decision-making performance_

A study by [Creswell et al. (2013)](#R2) examined whether it could find neural correlates of the so-called *Unconscious Thought Effect* (cf. [Dijksterhuis, 2013](#R3))    
The claim is that incubation effects, the unreflective emergence of meaning that is associated with creative processes and insight in problem solving, also occur in complex decision making. Participants who are distracted from thinking about a decision problem (UT) make a better choice than participants who had a chance to consciously think about the problem (CT), or who had to make an immediate decision after the problem was presented (ID).   
    
This paradigm was implemented as a within-subjects fMRI study in which participants went through all 3 conditions rating each one of three Cars, Apartments, Backpacks. The accuracy of the decision process was defined as the difference between the rating for the best item and the worst item. Of course, for the fMRI data to make any sense, it is important that the behavioural effects are replicated. From the article:
      
> Paired t-tests indicated that UT produced better decisions compared with the ID [t(26)=2.15, P=0.04] and CT [t(26)=2.10, P=0.04] conditions [the overall one-way ANOVA was marginally significant, F(2,52)=2.48, P=0.09].    
[...]   
> In this study, we did not find that a 2-min period of CT produced better decision making compared with the ID condition  [paired samples t(26)=0.20, P=0.84]   
    
Note that a one-way ANOVA is not the correct analysis for repeated measures data, oddly enough the df of the F-test, F(2,52) do not correspond to a one-way ANOVA at all. That would have been F(2,24) and for a repeated measures ANOVA F(2,50). In any case, df and p-value were apparently 'close enough' to warrant three post-hoc paired samples t-tests. Indeed, use of paired sample tests is correct here, but a correction for multiple comparisons should be used and at $latex \alpha = 0.017$ none of these post-hoc tests is significant. This is in accordance with the F-test for the main effect.   
  
In addition to the behavioural data, the authors report tests that show neural reactivation in several areas of the brain predicts decision performance (difference between judgements of the best and the worst item). The association between neural reactivation and performance was not tested in one model even though the contrasts are just linear combinations of the within-subject design factor and neural activity and rating differences are measured from one and the same random outcome, the randomly selected participant.   
   
> We observed that neural reactivation occurring in right dorsolateral PFC [$latex \beta$=0.39, t(26)=2.13, P=0.04] and left intermediate visual cortex [$latex \beta$=0.40, t(26)=2.20, P=0.04] predicted subsequent decision-making performance, such that more neural reactivation in these regions was associated with greater discrimination between the best and worst items on the decision-making task.   
> [...]    
> Although we observed clusters of neural reactivation during CT (Figure 5; Table 4), none of these clusters significantly predicted decision-making performance. Specifically, CT reactivation clusters observed in right cerebellum [$latex \beta$=0.27, t(26)=1.39, P=0.18], left supplementary motor area [$latex \beta$=0.21, t(26)=1.07, P=0.29], right ventrolateral PFC [$latex \beta$=0.18, t(26)=0.90, P=0.38] and right intraparietal lobule [$latex \beta$=0.13, t(26)=0.67, P=0.51] did not predict decision performance after CT.    
    
**T-tests galore!**   
Again, one should accommodate in some way for multiple hypothesis testing and assuming just 2 tests were conducted to evaluate the benefits of UT mode of thought at $latex \alpha = 0.025$, we might as well be looking at [a dead salmon](http://prefrontal.org/files/posters/Bennett-Salmon-2009.pdf), or at least a red herring.    
Granted, the neural activity measure used in these tests is itself the result of multiple comparison corrected analyses. This measure is the output from a conjunction analysis in which activity during several different conditions is examined conditional on a specific contrast or hypothesis (*UT*: (UT n-back task > independent n-back task) AND (encoding > fixation); *CT*: (CT fixation > fixation) AND (encoding > fixation)). The purpose is to identify clusters of voxels whose activity is connected and associated to the UT and CT modes of thought as specified in the contrast. That is, assuming brain physiology can be neatly decomposed into the architecture of cognitive components and processes posited to exist by these authors, but that is another story.   
    
How to deal with within-subjects effect sizes? There must be correlations between within-subject conditions, so is it at all possible to calculate ESCIs? Yes, provided one can be sure the t-value was obtained from a paired samples t-test, because then it is associated to a tests of the mean difference. An effect size in this case is a standardised mean, whose magnitude indicates how much the observed within-subject mean difference deviates from 0. The `MBESS` function `ci.sm(ncp= , N= )` can be used here, taking the paired samples t-value as the non-centrality parameter, ncp.  

**Setup the data**    
Compared to the OR example, the code below reflects a different approach to get the ESCIs.     
     
It is in general more efficient in R to use the so-called `apply` functions that ship with R when you need to *apply* the same function repeatedly to different data vectors. These functions take each element of a list object and pass those to a function, returning the results as another list object. A package that can make your `apply` coding a little easier is `plyr`, it contains functions that are sometimes just wrappers for the native `apply` functions, but the great advantage is that the input and output object types can be chosen.    

```r
# Create a list of confidence levels adjusted for the number of comparisons
# of each family of tests (3, 2 and 4 respectively)
CL <- c(rep(1 - 0.05/3, 3), rep(1 - 0.05/2, 2), rep(1 - 0.05/4, 4))
CL
```

```
> [1] 0.9833 0.9833 0.9833 0.9750 0.9750 0.9875 0.9875 0.9875 0.9875
```

```r

# This is a list object with t-values from the article as elements.  The
# elements are named after the contrast / paired observation that was
# tested.
NCP <- list(UTvsID = c(2.15), UTvsCT = c(2.1), CTvsID = c(0.2), UTpfc = c(2.13), 
    UTvzc = c(2.2), CTcbl = c(1.39), CTsma = c(1.07), CTpfc = c(0.9), CTlob = c(0.67))
```


**Setup the analysis**    
That's it, a single line of code!

```r
# The function `ldply` passes the elements in NCP (t-values) to an anonymous
# function as `p`.  This ensures the t-values are interpreted as the correct
# input argument for ci.sm: ncp=p `ldply` returns the data in a dataframe,
# native `apply` functions would return a list object.
PTcomp <- ldply(NCP, function(p) as.data.frame(ci.sm(ncp = p, N = 27, conf.level = CL)))
```

     
**Setup the plot**    

```r
# Plot the results stored in data frame PTcomp
forest(x = PTcomp$Standardized.Mean, ci.lb = PTcomp$Lower.Conf.Limit.Standardized.Mean, 
    ci.ub = PTcomp$Upper.Conf.Limit.Standardized.Mean, slab = PTcomp$.id, alim = c(-1.5, 
        2.5), ilab = round(CL, digits = 3), ilab.xpos = 2.2, main = "Forest Plot", 
    xlim = c(-3, 5), xlab = "Standardised Mean", efac = 1, cex = 0.9, mgp = c(1, 
        1, 0))

# Add some text
text(2.2, (nrow(PTcomp) + 1.5), "adjusted CL")
text(3.9, (nrow(PTcomp) + 1.5), "Standardised Mean [Low,High]")
text(-2.2, (nrow(PTcomp) + 1.5), "Dependent samples t-test")
```

![plot of chunk SMDCI_plot](figure/SMDCI_plot.png) 


**Neural correlates of... barely observable behavioural phenomena?**    
It is tempting to place more epistemic weight to psychological effects evidenced using expensive measurement equipment, especially if those measurements concern outcomes of observables at the level where biology meets (quantum) physics. In this case, it is especially important not to reverse the chain of evidence: There is no difference between conditions in the behavioural measures which led to doing the experiment in an fMRI scanner! The data should not be presented as a confirmation of the behavioural effect:   
     
> Using BOLD contrast fMRI, we observed neural activity during an UT period, which challenges existing accounts that have claimed that deliberation without attention (UT) does not occur during periods of distraction (Acker, 2008).   
   
The first part is partly correct, neural activity was observed in UT, but also in CT. The second part is incorrect:   
 * The deliberation without attention effect concerns an advantage in decision making and this was not observed.    
 * The fact that activity was observed reveals nothing about the processes that were going on, moreover, the CT condition does not exclude that UT is taking place as well.   
 * There was no association between neural activation and performance.
 * If anything, the full interpretation concerns an interaction between Mode of Thought and Neural reactivation networks to predict decision performance. This interaction was never tested.
     
*The correct interpretation of the results of the study is*: Within an individual, there are different clusters of jointly active (=connected?) voxels, whose level of activity are represented by a t-value resulting from a test of a contrast composed of a set of relational constraints on the BOLD signal measured in different experimental conditions. It was thus shown that combining brain activity from experimental conditions that differ both in task, cognitive load and visual complexity into statistical hypotheses, will reveal that different clusters of voxels are active.   
   
****
<a name="A3"></a>3. Learn From Reproducible Open Data
==========================================================
   
Yes, much more to learn about R, about ESCI, and meta-analysis!   

**Example: The [ManyLabs](https://openscienceframework.org/project/WX7Ck/) project**    
    
A good place to continue from here on is to have a look at the R scripts used in the ManyLabs project.    
There are three R scripts available from the [OSF ManyLabs project pages](https://openscienceframework.org/project/WX7Ck/) that will also show you how to read from and write to a spreadsheet and how to save graphics device output to a pdf file:   
 1.`Manylabs_OriginalStudiesESCI.R` - ESCI for the original studies (correlations, $latex \chi^2$, Cohen's d).   
 2.`Manylabs_ReplicationStudiesESCI.R` - ESCI for the replication data (correlations, $latex \chi^2$, Cohen's d).   
 3.`ManyLabs_Heterogeneity.R` - Meta-analysis (forest plot, funnel plot, influence plot, radial plot, heterogeneity measures).   
   
Good luck!

[Fred Hasselman](http://fredhasselman.com)   
   
****
<a name="A4"></a>New to `R`?
========================================================
You have probably heard many people say they should invest more time and effort to learn to use the `R` software environment for statistical computing... *and they were right*. However, what they probably meant to say is: "I tried it, but it's so damned complicated, I gave up"... *and they were right*. That is, they were right to note that this is not a point and click tool designed to accommodate any user. It was built for the niche market of scientists who use statistics, but in that segment it's actually the most useful tool I have encountered so far. Now that your struggles with getting a grip on `R` are fully acknowledged in advance, let's try to avoid the 'giving up' from happening. Try to follow these steps to get started:   

1. **Get `R` and add some user comfort:** Install the latest [`R` software](http://www.r-project.org) *and* install a user interface like [RStudio](http://www.rstudio.com)... *It's all free!* An R interface will make some things easier, e.g., searching and installing packages from repositories. RStudio will also add functionality, like git/svn version control, project management and more, like the tools to create html pages like this one (`knitr` and `Rmarkdown`). Another source of user comfort are the `packages`. `R` comes with some basic packages installed, but you'll soon need to fit generalised linear mixture models, or visualise social networks using graph theory and that means you'll be searching for packages that allow you to do such things. A good place to start *package hunting* are the [CRAN task view](http://cran.r-project.org/web/views/) pages.
2. **Learn by running example `code`:** Copy the commands in the `code` blocks you find on this page, or any other tutorial or help files (e.g., Rob Kabacoff's [Quick R](http://www.statmethods.net)). Paste them into an `.R` script file in the script (or, source) editor. In RStudio You can run code by pressing `cmd` + `enter` when the cursor is on a single single line, or you can run multiple lines at once by selecting them first. If you get stuck remember that there are expert `R` users who probably have answered your question already when it was posted on a forum. Search for example through the Stackoverflow site for [questions tagged with `R`](http://stackoverflow.com/questions/tagged/r))
3. **Examine what happens... when you tell `R` to make something happen:** `R` stores variables (anything from numeric data to functions) in an `Environment`. There are in fact many different environments, but we'll focus on the main workspace for the current `R` session. If you run the command `x <- 1+1`, a variable `x` will appear in the `Environment` with the value `2` assigned to it. Examining what happens in the `Environment` is not the same as examining the output of a statistical analysis. Output in `R` will appear in the `Console` window. Note that in a basic set-up each new `R` session starts with an empty `Environment`. If you need data in another session, you can save the entire `Environment`, or just some selected variables, to a file (`.RData`).
4. **Learn about the properties of `R` objects:** Think of objects as containers designed for specific content. One way to characterize the different objects in `R` is by how picky they are about the content you can assign it. There are objects that hold `character` and `numeric` type data, a `matrix` for numeric data organised in rows and columns, a `data.frame` is a matrix that allows different data types in columns, and least picky of all is the `list` object. It can carry any other object, you can have a `list` of which item 1 is an entire `data.frame` and item 2 is just a `character` vector of the letter `R`. The most difficult thing to master is how to efficiently work with these objects, how to assign values and query contents.
5. **Avoid repeating yourself:** The `R` language has some amazing properties that allow execution of many repetitive algorithmic operations using just a few lines of code at speeds up to warp 10. Naturally, you'll need to be at least half Vulcan to master these features properly and I catch myself copying code when I shouldn't on a daily basis. The first thing you will struggle with are the `apply` functions. These functions pass the contents of a `list` object to a function. Suppose we need to calculate the means of column variables in 40 different SPSS `.sav` files stored in the folder `DAT`. With the `foreign` package loaded we can execute the following commands:   
`data <- lapply(dir("/DAT/",pattern=".sav$"),read.spss)`        
`out  <- sapply(data,colMeans)`       
The first command applies read.spss to all files with a `.sav` extension found in the folder `/DAT`. It creates a dataframe for each file which are all stored as elements of the list `data`. The second line applies the function `colMeans` to each element of `data` and puts the combined results in a matrix with dataset ID as columns (1-40), dataset variables as rows and the calculated column means as cells. This is just the beginning of the `R` magic, wait 'till you learn how to write functions that can create functions.      
    
****    
**References**   

 * <a name="R2"></a> Creswell, J. D., Bursley, J. K., & Satpute, A. B. (2013). Neural reactivation links unconscious thought to decision-making performance. _Social Cognitive and Affective Neuroscience, 8(8)_, 863–869. doi:10.1093/scan/nst004    
 * <a name="R3"></a> Dijksterhuis, A. (2013). First neural evidence for the unconscious thought process. _Social Cognitive and Affective Neuroscience, 8(8)_, 845–846. doi:10.1093/scan/nst036    
 * <a name="R1"></a> Gino, F., & Mogilner, C. (online, 2013). Time, Money, and Morality. _Psychological Science_. doi: 10.1177/0956797613506438   
