---
title: "A guided tour in targeted learning territory"
author: "David Benkeser, Antoine Chambaz, Nima Hejazi"
date: "08/13/2018"
encoding: "UTF-8"
output:
  bookdown::pdf_document2:
    toc: true
    toc_depth: 2
    number_sections: true
    includes:
      in_header: dan_header.tex
  latex_engine: pdflatex
  citation_package: natbib
---

<!-- to compile the file, run

R -e "rmarkdown::render('dan.Rmd', encoding='UTF-8')"

or 

R> bookdown::render_book("dan.Rmd")

-->




\section{Introduction}

\tcg{This is a very first draft  of our article. The current *tentative* title
  is "A guided tour in targeted learning territory".}

\tcg{Explain our objectives and how we will meet them. Explain that the symbol
\textdbend indicates more delicate material.}

\tcg{Use sectioning a lot to ease cross-referencing.}

\tcg{Do we include  exercises? I propose we do, and  to flag the corresponding
subsections with symbol $\gear$.}


```r
set.seed(54321) ## because reproducibility matters...
suppressMessages(library(R.utils)) ## make sure it is installed
suppressMessages(library(tidyverse)) ## make sure it is installed
suppressMessages(library(ggplot2)) ## make sure it is installed
suppressMessages(library(caret)) ## make sure it is installed
expit <- plogis
logit <- qlogis
```

Function `expit` implements the link function  $\expit : \bbR \to ]0,1[$ given
by  $\expit(x) \equiv  (1 +  e^{-x})^{-1}$.  Function  `logit` implements  its
inverse function  $\logit : ]0,1[  \to \bbR$  given by $\logit(p)  \equiv \log
[p/(1-p)]$.

\section{A simulation study}
\label{sec:simulation:study}

blabla

\subsection{Reproducible experiment as a law.}
\label{subsec:as:a:law}

We are interested in a reproducible experiment. The generic summary of how one
realization of the experiment unfolds, our observation, is called $O$. We view
$O$  as a  random variable  drawn from  what we  call the  law $P_{0}$  of the
experiment.  The  law $P_{0}$  is viewed  as an  element of  what we  call the
model. Denoted  by $\calM$, the model  is the collection of  \textit{all} laws
from which  $O$ can be drawn  and that meet some  constraints. The constraints
translate the knowledge  we have about the experiment. The  more we know about
the experiment,  the smaller is  $\calM$. In  all our examples,  model $\calM$
will put very few restrictions on the candidate laws.

Consider the following chunk of code:


```r
draw_from_experiment <- function(n, ideal = FALSE) {
  ## preliminary
  n <- Arguments$getInteger(n, c(1, Inf))
  ideal <- Arguments$getLogical(ideal)
  ## ## 'Gbar' and 'Qbar' factors
  Gbar <- function(W) {
    expit(-0.2 + 3 * sqrt(W) - 1.5 * W)
  }
  Qbar <- function(AW) {
    A <- AW[, 1]
    W <- AW[, 2]
    ## A * cos((1 + W) * pi / 5) + (1 - A) * sin((1 + W^2) * pi / 4)
    A * (cos((1 + W) * pi / 5) + (1/3 <= W & W <= 1/2) / 10) +
      (1 - A) * (sin(4 * W^2 * pi) / 4 + 1/2) 
  }
  ## sampling
  ## ## context
  W <- runif(n)
  ## ## counterfactual rewards
  zeroW <- cbind(A = 0, W)
  oneW <- cbind(A = 1, W)
  Qbar.zeroW <- Qbar(zeroW)
  Qbar.oneW <- Qbar(oneW)
  Yzero <- rbeta(n, shape1 = 2, shape2 = 2 * (1 - Qbar.zeroW) / Qbar.zeroW)
  Yone <- rbeta(n, shape1 = 3, shape2 = 3 * (1 - Qbar.oneW) / Qbar.oneW)
  ## ## action undertaken
  A <- rbinom(n, size = 1, prob = Gbar(W))
  ## ## actual reward
  Y <- A * Yone + (1 - A) * Yzero
  ## ## observation
  if (ideal) {
    obs <- cbind(W = W, Yzero = Yzero, Yone = Yone, A = A, Y = Y)
  } else {
    obs <- cbind(W = W, A = A, Y = Y)
  }
  attr(obs, "Gbar") <- Gbar
  attr(obs, "Qbar") <- Qbar
  attr(obs, "QW") <- dunif
  attr(obs, "qY") <- function(AW, Y, Qbar){
    A <- AW[, 1]
    W <- AW[, 2]
    Qbar.AW <- do.call(Qbar, list(AW))
    shape1 <- ifelse(A == 0, 2, 3)
    dbeta(Y, shape1 = shape1, shape2 = shape1 * (1 - Qbar.AW) / Qbar.AW)
  }
  ##
  return(obs)
}
```

We can interpret `draw_from_experiment` as a  law $P_{0}$ since we can use the
function to sample observations  from a common law.  It is  even a little more
than that, because we can tweak the experiment, by setting its `ideal` argument
to  `TRUE`, in  order  to  get what  appear  as intermediary  (counterfactual)
variables  in  the regular  experiment.   The  next  chunk  of code  runs  the
(regular)  experiment  five  times  independently and  outputs  the  resulting
observations: 


```r
(five_obs <- draw_from_experiment(5))
```

```
##              W A         Y
## [1,] 0.4290078 0 0.9426242
## [2,] 0.4984304 1 0.7202482
## [3,] 0.1766923 1 0.8768885
## [4,] 0.2743935 0 0.8494665
## [5,] 0.2165102 1 0.3849406
## attr(,"Gbar")
## function (W) 
## {
##     expit(-0.2 + 3 * sqrt(W) - 1.5 * W)
## }
## <bytecode: 0x7fa6587dae00>
## <environment: 0x7fa6582c2e40>
## attr(,"Qbar")
## function (AW) 
## {
##     A <- AW[, 1]
##     W <- AW[, 2]
##     A * (cos((1 + W) * pi/5) + (1/3 <= W & W <= 1/2)/10) + (1 - 
##         A) * (sin(4 * W^2 * pi)/4 + 1/2)
## }
## <bytecode: 0x7fa6596edc60>
## <environment: 0x7fa6582c2e40>
## attr(,"QW")
## function (x, min = 0, max = 1, log = FALSE) 
## .Call(C_dunif, x, min, max, log)
## <bytecode: 0x7fa658ece108>
## <environment: namespace:stats>
## attr(,"qY")
## function (AW, Y, Qbar) 
## {
##     A <- AW[, 1]
##     W <- AW[, 2]
##     Qbar.AW <- do.call(Qbar, list(AW))
##     shape1 <- ifelse(A == 0, 2, 3)
##     dbeta(Y, shape1 = shape1, shape2 = shape1 * (1 - Qbar.AW)/Qbar.AW)
## }
## <bytecode: 0x7fa658d5ba40>
## <environment: 0x7fa6582c2e40>
```

We can view the `attributes` of object `five_obs` because, in this section, we
act  as  oracles,  \textit{i.e.},  we   know  completely  the  nature  of  the
experiment. In  particular, we  have included several  features of  $P_0$ that
play an important  role in our developments. The attribute  `QW` describes the
density  of $W$,  that  of  a uniform  distribution  over  $[0.1, 0.9]$.   The
attribute `Gbar` describes the conditional probability of action $A = 1$ given
$W$, and  for $a  = 0,1$  and each  $w$ in the  support of  $W$, we  denote by
$\Gbar_0(a,w) = pr_{P_0}(A = a \mid W  = w)$. The attribute `qY` describes the
conditional   density  of   $Y$   given   $A$  and   $W$.    For  each   $y\in
\interval[open]{0}{1}$,  we  denote  by  $q_{0,Y}(y, a,  w)$  the  conditional
density of $Y$ given $A = a, W = w$ evaluated at $y$. Similarly, the attribute
`Qbar` describes the conditional mean of $Y$  given $A$ and $W$, and we denote
by $\Qbar_0(a,w)$ the conditional mean of $Y$ given $A = a, W = w$.

*[David  note: I  revised  this  paragraph to  include  a  description of  the
conditional  density  of  $Y$,  which  is  needed  to  describe  the  quantile
exercises.  Also, should  we consider  adopting the  notational convention  of
lower case  $q$ for density  (e.g., $q_W$) and  upper case $q$  for cumulative
distribution? I need both quantities for the quantile exercises.]*

\subsection{The parameter of interest, first pass.}
\label{subsec:parameter:first}

It happens that we especially care for a finite-dimensional feature of $P_{0}$
that  we   denote  by  $\psi_{0}$.    Its  definition  involves  two   of  the
aforementioned  infinite-dimensional  features: \begin{align}  \label{eq:psi0}
\psi_{0}  &\equiv   \int  \left(\Qbar_{0}(1,   w)  -   \Qbar_{0}(0,  w)\right)
dQ_{0,W}(w)\\  \notag  &=  E_{P_{0}}   \left(\Qbar_{0}(1,  W)  -  \Qbar_{0}(0,
W)\right).   \end{align} Acting  as  oracles, we  can  compute explicitly  the
numerical value of  $\psi_{0}$.  *[David note: Is the  first equality helpful?
The preceding  sentence might cause  a reader to  expect to see  $\Qbar_0$ and
$Q_{0,W}$ in the equation.   So maybe we flip the order?   Or remove the first
equality altogether?]*


```r
integrand <- function(w) {
  Qbar <- attr(five_obs, "Qbar")
  QW <- attr(five_obs, "QW")
  ( Qbar(cbind(1, w)) - Qbar(cbind(0, w)) ) * QW(w)
}
(psi_zero <- integrate(integrand, lower = 0, upper = 1)$val)
```

```
## [1] 0.0605389
```

Our  interest in  $\psi_{0}$ is  of  causal nature.  Taking a  closer look  at
`drawFromExperiment` reveals indeed  that the random making  of an observation
$O$ drawn  from $P_{0}$ can  be summarized by  the following causal  graph and
nonparametric system of structural equations:


```r
## plot the causal diagram
```

and, for some deterministic functions $f_w$, $f_a$, $f_y$ and independent
sources of randomness $U_w$, $U_a$, $U_y$,
\begin{enumerate}
\item sample  the context where the  rest of the experiment
  will take place, $W = f_{w}(U_w)$;
\item  sample the  two counterfactual  rewards of  the two
  actions  that   can  be   undertaken,  $Y_{0}  =   f_{y}(0,  W,   U_y)$  and
  $Y_{1} = f_{y}(1, W, U_y)$;
\item\label{item:A:equals} sample   which  action is carried
  out in the given context, $A = f_{a} (W, U_a)$;
\item    define        the    corresponding    reward,
  $Y = A Y_{1} + (1-A) Y_{0}$;
\item summarize the course of the experiment  with the observation $O = (W, A,
  Y)$, thus concealing $Y_{0}$ and $Y_{1}$. 
\end{enumerate}

The above  description of the  experiment `draw_from_experiment` is  useful to
reinforce what it means to run the "ideal" experiment by setting argument `ideal`
to  `TRUE`  in   a  call  to  `draw_from_experiment`.  Doing   so  triggers  a
modification   of  the   nature  of   the  experiment,   enforcing  that   the
counterfactual  rewards $Y_{0}$  and $Y_{1}$  be part  of the  summary of  the
experiment eventually.  In light  of the above  enumeration, $\bbO  \equiv (W,
Y_{0}, Y_{1}, A,  Y)$ is output, as  opposed to its summary  measure $O$. This
defines another experiment and its law, that we denote $\bbP_{0}$.

*[David note: Would 'ideal' or 'perfect' experiment be better than 'full'?]*

It is well known \tcg{(do we give the proof or refer to other articles?)} that
\begin{equation} \label{eq:psi:zero}    
\psi_{0}  =  E_{\bbP_{0}} \left(Y_{1}  -  Y_{0}\right)  = E_{\bbP_{0}}(Y_1)  -
E_{\bbP_{0}}(Y_0).   \end{equation}  Thus,  $\psi_{0}$ describes  the  average
difference in of  the two counterfactual rewards.  In  other words, $\psi_{0}$
quantifies the difference  in average of the  reward one would get  in a world
where one would always enforce action $a=1$ with the reward one would get in a
world where  one would always  enforce action $a=0$.   This said, it  is worth
emphasizing  that $\psi_{0}$  is a  well defined  parameter beyond  its causal
interpretation.

*[David   note:   Is  it   worth   writing   this  as   $E_{\bbP_{0}}(Y_1)   -
E_{\bbP_{0}}(Y_0)$ as well (or instead)? Maybe I am thinking too much, but for
the quantile example,  we compare a quantile  of $Y_1$ to a  quantile of $Y_0$
rather than describe a quantile of the  difference $Y_1 - Y_0$; the latter, of
course,  involves   cross-world  distributions  (i.e.,  the   joint  dist.  of
$(Y_1,Y_0)$)]*

To conclude  this subsection, we  take advantage of  our status as  oracles to
sample observations from the  ideal experiment. We call `draw_from_experiment`
with its  argument `ideal` set to  `TRUE` in order to  numerically approximate
$\psi_{0}$.   By  the law  of  large  numbers,  the  following chunk  of  code
approximates $\psi_{0}$ and shows it approximate value:


```r
B <- 1e5 ## Antoine: 1e6 eventually
ideal_obs <- draw_from_experiment(B, ideal = TRUE)
(psi_hat <- mean(ideal_obs[, "Yone"] - ideal_obs[, "Yzero"]))
```

```
## [1] 0.06062719
```

In fact, the central limit theorem and Slutsky's lemma allow us to build a
confidence interval with asymptotic level 95\% for $\psi_{0}$:


```r
sd_hat <- sd(ideal_obs[, "Yone"] - ideal_obs[, "Yzero"])
alpha <- 0.05
(psi_CI <- psi_hat + c(-1, 1) * qnorm(1 - alpha / 2) * sd_hat / sqrt(B))
```

```
## [1] 0.05858054 0.06267383
```

*[David note: Could remove the confidence interval bit?]*

\subsection{\gear Difference in covariate-adjusted quantile rewards, first
pass.}  
\label{subsec:exo:dave:one}

The questions are asked in the context of Sections \ref{subsec:parameter:first}.

As discussed  above, parameter $\psi_0$ \eqref{eq:psi:zero}  is the difference
in average  rewards if  we enforce  action $a  = 1$  rather than  $a =  0$. An
alternative  way to  describe  the rewards  under  different actions  involves
quantiles as opposed to averages.  

Let $Q_{0,Y}(y,  a, w) =  \int_{0}^y q_{0,Y}(u, a,  w) du$ be  the conditional
cumulative distribution of  reward $Y$ given $A=a$ and $W=w$,  evaluated at $y
\in \interval[open]{0}{1}$, that is implied by  $P_0$.  For each action $a \in
\{0,1\}$  and   $c  \in  \interval[open]{0}{1}$,   introduce  \begin{equation}
\label{def_quantile}     \gamma_{0,a,c}    \equiv     \inf    \left\{y     \in
\interval[open]{0}{1}  : \int  Q_{0,Y}(y, a,  w) dQ_{0,W}(w)  \ge c  \right\}.
\end{equation}

It is not difficult to check that \tcg{do  we give the proof or refer to other
articles?}      \begin{equation*}\gamma_{0,a,c}     =     \inf\left\{y     \in
\interval[open]{0}{1}      :      pr_{\bbP_{0}}(Y_a     \leq      y)      \geq
c\right\}.\end{equation*}  Thus,  $\gamma_{0,a,c}$  can be  interpreted  as  a
covariate-adjusted $c$-th  quantile reward when  action $a$ is  enforced.  The
difference     \begin{equation*}\delta_{0,c}    \equiv     \gamma_{0,1,c}    -
\gamma_{0,0,c}\end{equation*} is the $c$-th  quantile counterpart to parameter
$\psi_{0}$ \eqref{eq:psi:zero}.

1. \textdbend Compute the numerical  value of $\gamma_{0,a,c}$ for each $(a,c)
   \in \{0,1\} \times  \{1/4, 1/2, 3/4\}$ using the  appropriate attributes of
   `five_obs`.   Based  on  these  results,  report  the  numerical  value  of
   $\delta_{0,c}$ for each $c \in \{1/4, 1/2, 3/4\}$.
   
2. Approximate  the numerical values  of $\gamma_{0,a,c}$ for each  $(a,c) \in
   \{0,1\}  \times \{1/4,  1/2,  3/4\}$ by  drawing a  large  sample from  the
   "ideal"  data experiment  and using  empirical quantile  estimates.  Deduce
   from these results  a numerical approximation to $\delta_{0,c}$  for $c \in
   \{1/4, 1/2, 3/4\}$. Confirm that  your results closely match those obtained
   in the previous problem.


\subsection{The parameter of interest, second pass.}
\label{subsec:parameter:second}

Suppose we  know beforehand that  $O$ drawn from  $P_{0}$ takes its  values in
$\calO \equiv [0,1] \times \{0,1\}  \times \interval{0}{1}$ and that $\Gbar(W)
=  P_{0}(A=1|W)$ is  bounded away  from zero  and one  $Q_{0,W}$-almost surely
(this is the case indeed).  Then we can define model $\calM$ as the set of all
laws $P$ on $\calO$ such that  $\Gbar(W) \equiv P(A=1|W)$ is bounded away from
zero and one $Q_{W}$-almost surely, where $Q_{W}$ is the marginal law of $W$
under $P$.  

Let us also define generically $\Qbar$ as \begin{equation*} \Qbar (A,W) \equiv
E_{P} (Y|A, W). \end{equation*} Central  to our approach is viewing $\psi_{0}$
as the  value at  $P_{0}$ of  the statistical mapping  $\Psi$ from  $\calM$ to
$[0,1]$ characterized  by \begin{align*}  \Psi(P) &\equiv  \int \left(\Qbar(1,
w) -  \Qbar(0, w)\right) dQ_{W}(w)  \\ &=  E_{P} \left(\Qbar(1, W)  - \Qbar(0,
W)\right), \end{align*}  a clear extension of  \eqref{eq:psi0}.  For instance,
although the law  $\Pi_{0} \in \calM$ encoded by  default (\textit{i.e.}, with
`h=0`)  in  `drawFromAnotherExperiment`  defined below  differs  starkly  from
$P_{0}$,


```r
draw_from_another_experiment <- function(n, h = 0) {
  ## preliminary
  n <- Arguments$getInteger(n, c(1, Inf))
  h <- Arguments$getNumeric(h)
  ## ## 'Gbar' and 'Qbar' factors
  Gbar <- function(W) {
    sin((1 + W) * pi / 6)
  }
  Qbar <- function(AW, hh = h) {
    A <- AW[, 1]
    W <- AW[, 2]
    expit( logit( A *  W + (1 - A) * W^2 ) +
           hh * 10 * sqrt(W) * A )
  }
  ## sampling
  ## ## context
  W <- runif(n, min = 1/10, max = 9/10)
  ## ## action undertaken
  A <- rbinom(n, size = 1, prob = Gbar(W))
  ## ## reward
  shape1 <- 4
  QAW <- Qbar(cbind(A, W))
  Y <- rbeta(n, shape1 = shape1, shape2 = shape1 * (1 - QAW) / QAW)
  ## ## observation
  obs <- cbind(W = W, A = A, Y = Y)
  attr(obs, "Gbar") <- Gbar
  attr(obs, "Qbar") <- Qbar
  attr(obs, "QW") <- function(x){dunif(x, min = 1/10, max = 9/10)}
  attr(obs, "shape1") <- shape1
  attr(obs, "qY") <- function(AW, Y, Qbar, shape1){
    A <- AW[,1]; W <- AW[,2]
    Qbar.AW <- do.call(Qbar, list(AW))
    dbeta(Y, shape1 = shape1, shape2 = shape1 * (1 - Qbar.AW) / Qbar.AW)
  }
  ##
  return(obs)
}
```

the parameter $\Psi(\Pi_{0})$ is well defined, and numerically approximated by
`psi_Pi_zero` as follows. 


```r
five_obs_from_another_experiment <- draw_from_another_experiment(5)
another_integrand <- function(w) {
  Qbar <- attr(five_obs_from_another_experiment, "Qbar")
  QW <- attr(five_obs_from_another_experiment, "QW")
  ( Qbar(cbind(1, w)) - Qbar(cbind(0, w)) ) * QW(w)
}
(psi_Pi_zero <- integrate(another_integrand, lower = 0, upper = 1)$val)
```

```
## [1] 0.1966687
```

Straightforward algebra confirms that indeed $\Psi(\Pi_{0}) = 59/300$.

\subsection{\gear  Difference in  covariate-adjusted quantile  rewards, second
pass.}  
\label{subsec:exo:dave:two}

We  continue with  the exercise  from Section  \ref{subsec:exo:dave:one}.  The
questions are asked in the context of Section \ref{subsec:parameter:first}.

As above,  we define $q_{Y}(y,a,w)$  to be the $(A,W)$-conditional  density of
$Y$ given $A=a$ and  $W=w$, evaluated at $y$, that is implied  by a generic $P
\in \calM$.  Similarly, we use  $Q_{Y}$ to denote the corresponding cumulative
distribution  function.  The  covariate-adjusted  $c$-th  quantile reward  for
action $a \in \{0,1\}$ may be  viewed as a mapping $\Gamma_{a,c}$ from $\calM$
to $[0,1]$  characterized by \begin{equation*} \Gamma_{a,c}(P)  = \inf\left\{y
\in  \interval[open]{0}{1}  :  \int   Q_{Y}(y,a,w)  dQ_W(w)  \ge  c  \right\}.
\end{equation*} The  difference in  $c$-th quantile  rewards may  similarly be
viewed  as a  mapping $\Delta_c$  from  $\calM$ to  $[0,1]$, characterized  by
$\Delta_c(P) \equiv \Gamma_{1,c}(P) - \Gamma_{0,c}(P)$.

1. Compute the numerical value of $\Gamma_{a,c}(\Pi_0)$ for $(a,c) \in \{0,1\}
   \times   \{1/4,   1/2,  3/4\}$   using   the   appropriate  attributes   of
   `five_obs_from_another_experiment`.   Based on  these  results, report  the
   numerical value of $\Delta_c(\Pi_0)$ for each $c \in \{1/4, 1/2, 3/4\}$.

2. Approximate the  value of $\Gamma_{0,a,c}(\Pi_{0})$ for  $(a,c) \in \{0,1\}
   \times \{1/4, 1/2,  3/4\}$ by drawing a large sample  from the "ideal" data
   experiment  and  using empirical  quantile  estimates.   Deduce from  these
   results a numerical  approximation to $\Delta_{0,c} (\Pi_{0})$  for each $c
   \in  \{1/4, 1/2,  3/4\}$.  Confirm  that your  results closely  match those
   obtained in the previous problem.
   
3. Building upon the code you wrote to solve the previous problem, construct a
   confidence  interval   with  asymptotic  level  $95\%$   for  $\Delta_{0,c}
   (\Pi_{0})$, with  $c \in  \{1/4, 1/2, 3/4\}$.\footnote{Let  $X_{1}, \ldots,
   X_{n}$  be  independently drawn  from  a  continuous distribution  function
   $F$. Set  $p \in  \interval[open]{0}{1}$ and, assuming  that $n$  is large,
   find   $k\geq  1$   and  $l   \geq   1$  such   that  $k/n   \approx  p   -
   \Phi^{-1}(1-\alpha)    \sqrt{p(1-p)/n}$    and    $l/n    \approx    p    +
   \Phi^{-1}(1-\alpha) \sqrt{p(1-p)/n}$,  where $\Phi$ is the  standard normal
   distribution function.  Then  $\interval{X_{(k)}}{X_{(l)}}$ is a confidence
   interval for $F^{-1}(p)$ with asymptotic level $1 - 2\alpha$.}

\subsection{Being smooth, first pass.}
\label{subsec:being:smooth:one}


Luckily, the statistical mapping $\Psi$ is well behaved, or smooth.  Here,
this colloquial expression refers to the fact that, for each $P \in \calM$, if
$P_{h} \to_{h} P$ in  $\calM$ from a direction $s$ when  the real parameter $h
\to 0$,  then not  only $\Psi(P_{h}) \to_{h}  \Psi(P)$ (continuity),  but also
$h^{-1} [\Psi(P_{h}) - \Psi(P)] \to_{h} c$,  where the real number $c$ depends
on $P$ and $s$ (differentiability).


For   instance,   let   $\Pi_{h}   \in   \calM$  be   the   law   encoded   in
`draw_from_another_experiment` with  `h` ranging over  $\interval{-1}{1}$.  We
will argue shortly that $\Pi_{h} \to_{h}  \Pi_{0}$ in $\calM$ from a direction
$s$ when  $h \to  0$.  The  following chunk of  code evaluates  and represents
$\Psi(\Pi_{h})$   for   $h$   ranging   in   a   discrete   approximation   of
$\interval{-1}{1}$:


```r
approx <- seq(-1, 1, length.out = 1e2)
psi_Pi_h <- sapply(approx, function(t) {
  obs_from_another_experiment <- draw_from_another_experiment(1, h = t)
  integrand <- function(w) {
    Qbar <- attr(obs_from_another_experiment, "Qbar")
    QW <- attr(obs_from_another_experiment, "QW")
    ( Qbar(cbind(1, w)) - Qbar(cbind(0, w)) ) * QW(w)
  }
  integrate(integrand, lower = 0, upper = 1)$val  
})
slope_approx <- (psi_Pi_h - psi_Pi_zero) / approx
slope_approx <- slope_approx[min(which(approx > 0))]
ggplot() +
  geom_point(data = data.frame(x = approx, y = psi_Pi_h), aes(x, y),
             color = "#CC6666") +
  geom_segment(aes(x = -1, y = psi_Pi_zero - slope_approx,
                   xend = 1, yend = psi_Pi_zero + slope_approx),
               arrow = arrow(length = unit(0.03, "npc")),
               color = "#9999CC") +
  geom_vline(xintercept = 0, color = "#66CC99") +
  geom_hline(yintercept = psi_Pi_zero, color = "#66CC99") +
  labs(x = "h", y = expression(Psi(Pi[h]))) 
```

![Evolution of statistical parameter $\Psi$ along fluctuation $\{\Pi_{h} : h \in H\}$.](img/psi-approx-psi-one-1.png)

The dotted curve  represents the function $h \mapsto  \Psi(\Pi_{h})$. The blue
line represents  the tangent to the  previous curve at $h=0$,  which is indeed
differentiable around $h=0$.  It is  derived by simple geometric arguments. In
the  next  subsection,  we formalize  what  it  means  to  be smooth  for  the
statistical mapping $\Psi$. Once the presentation is complete, we will be able
to derive a  closed-form expression for the  slope of the blue  curve from the
chunk of code where `draw_from_another_experiment` is defined.

\subsection{\textdbend Being smooth, second pass.}
\label{subsec:being:smooth:two}

Let us now describe what it means  for statistical mapping $\Psi$ to be smooth
at  every $P  \in \calM$.   The description  necessitates the  introduction of
fluctuations.

For every direction\footnote{A direction is a measurable function.} $s : \calO
\to \bbR$ such that  $s \neq 0$\footnote{That is, $s(O)$ is  not equal to zero
$P$-almost surely.},  $E_{P} (s(O))  = 0$  and $s$ bounded  by, say,  $M$, for
every $h \in  H \equiv \interval[open]{-M^{-1}}{M^{-1}}$, we can  define a law
$P_{h}  \in \calM$  by  setting  $P_{h} \ll  P$\footnote{That  is, $P_{h}$  is
dominated  by $P$:  if an  event $A$  satisfies $P(A)  = 0$,  then necessarily
$P_{h} (A) = 0$ too.}  and

\begin{equation}\label{eq:fluct}\frac{dP_{h}}{dP}(O)    \equiv     1    +    h
s(O),\end{equation}

that is, $P_{h}$ has density $(1 + h s)$ with respect to (w.r.t.) $P$. We call
$\{P_{h}  :  h  \in  H\}$  a  fluctuation of  $P$  in  direction  $s$  because

\begin{equation}\label{eq:score}(i)  \;  P_{h}|_{h=0}  =   P,  \quad  (ii)  \;
\left.\frac{d}{dh}       \log        \frac{dP_{h}}{dP}(O)\right|_{h=0}       =
s(O).\end{equation} 

The fluctuation is a one-dimensional parametric submodel of $\calM$. 

Statistical mapping $\Psi$ is smooth at  every $P \in \calM$ because, for each
$P \in \calM$, there exists  a so called efficient influence curve\footnote{It
is  a   measurable  function.}   $D^{*}(P)   :  \calO  \to  \bbR$   such  that
$E_{P}(D^{*}(P)(O)) = 0$ and, for any direction  $s$ as above, if $\{P_{h} : h
\in H\}$  is defined as in  \eqref{eq:fluct}, then the real-valued  mapping $h
\mapsto \Psi(P_{h})$ is differentiable at $h=0$, with a derivative equal to

\begin{equation}\label{eq:derivative}E_{P} \left(D^{*}(P)(O) s(O)\right).\end{equation}

Interestingly,   if  a   fluctuation   $\{P_{h}  :   h   \in  H\}$   satisfies
\eqref{eq:score} for  a direction $s$ such  that $s\neq 0$, $E_{P}(s(O))  = 0$
and  $\Var_{P}  (s(O))  <  \infty$,  then $h \mapsto  \Psi(P_{h})$  is  still
differentiable  at  $h=0$ with  a  derivative  equal to  \eqref{eq:derivative}
(beyond fluctuations of the form \eqref{eq:fluct}).

The influence curves $D^{*}(P)$ convey valuable information about $\Psi$. For
instance,  an  important  result  from   the  theory  of  inference  based  on
semiparametric models  guarantees that if $\psi_{n}$  is a regular\footnote{We
can view  $\psi_{n}$ as the  by product of  an algorithm $\Psihat$  trained on
independent observations $O_{1}$, \ldots, $O_{n}$ drawn from $P$.  
% or, equivalently, trained on the empirical measure $P_{n} = n^{-1}
% \sum_{i=1}^{n} \Dirac(O_{i})$: $\psi_{n} = \Psihat(P_{n})$.  
The estimator is regular at $P$ (w.r.t. the maximal tangent space) if, for any
direction  $s\neq 0$  such that  $E_{P}  (s(O)) =  0$ and  $\Var_{P} (s(O))  <
\infty$ and fluctuation $\{P_{h} : h \in H\}$ satisfying \eqref{eq:score}, the
estimator $\psi_{n,1/\sqrt{n}}$ of $\Psi(P_{1/\sqrt{n}})$ obtained by training
$\Psihat$  on independent  observations  $O_{1}$, \ldots,  $O_{n}$ drawn  from
$P_{1/\sqrt{n}}$    is   such    that    $\sqrt{n}   (\psi_{n,1/\sqrt{n}}    -
\Psi(P_{1/\sqrt{n}}))$ converges  in law to  a limit  that does not  depend on
$s$.} estimator  of $\Psi(P)$  built from  $n$ independent  observations drawn
from $P$, then the asymptotic variance  of the centered and rescaled $\sqrt{n}
(\psi_{n} - \Psi(P))$ cannot be smaller  than the variance of the $P$-specific
efficient influence curve, that is,

\begin{equation}\label{eq:CR}\Var_{P}(D^{*}(P)(O)).\end{equation}

In   this   light,   an   estimator    $\psi_{n}$   of   $\Psi(P)$   is   said
\textit{asymptotically efficient} at $P$ if it is regular at $P$ and such that
$\sqrt{n} (\psi_{n} - \Psi(P))$ converges in  law to the centered Gaussian law
with variance \eqref{eq:CR}, which is called the Cramér-Rao bound.

\subsection{The efficient influence curve.}
\label{subsec:parameter:third}

It is not difficult to check \tcg{(do  we give the proof?)} that the efficient
influence curve  $D^{*}(P)$ of  $\Psi$ at  $P \in  \calM$ writes  as $D^{*}(P)
\equiv D_{1}^{*}  (P) +  D_{2}^{*} (P)$ where  $D_{1}^{*} (P)$  and $D_{2}^{*}
(P)$ are given by

\begin{align*}D_{1}^{*}(P) (O)  &\equiv \Qbar(1,W)  - \Qbar(0,W)  - \Psi(P),\\
D_{2}^{*}(P)     (O)     &\equiv      \frac{2A-1}{\ell\Gbar(A,W)}     (Y     -
\Qbar(A,W)),\end{align*}

with   shorthand   notation   $\ell\Gbar(A,W)   \equiv   A\Gbar(W)   +   (1-A)
(1-\Gbar(W))$.  The  following chunk  of code enables  the computation  of the
values of the efficient influence  curve $D^{*}(P)$ at observations drawn from
$P$  (note that  it is  necessary  to provide  the  value of  $\Psi(P)$, or  a
numerical approximation thereof, through argument `psi`).


```r
eic <- function(obs, psi) {
  Qbar <- attr(obs, "Qbar")
  Gbar <- attr(obs, "Gbar")
  QAW <- Qbar(obs[, c("A", "W")])
  gW <- Gbar(obs[, "W"])
  lgAW <- obs[, "A"] * gW + (1 - obs[, "A"]) * (1 - gW)
  ( Qbar(cbind(1, obs[, "W"])) - Qbar(cbind(0, obs[, "W"])) - psi ) +
    (2 * obs[, "A"] - 1) / lgAW * (obs[, "Y"] - QAW)
}

(eic(five_obs, psi = psi_hat))
```

```
## [1] -1.0729204  0.1645226  0.2829207 -0.5969342 -0.4555602
```

```r
(eic(five_obs_from_another_experiment, psi = psi_Pi_zero))
```

```
## [1]  0.17717086  0.18409808 -0.07018161  0.36266406  0.15090865
```

\subsection{Computing and comparing Cramér-Rao bounds.}

We can use `eic` to numerically approximate the Cramér-Rao bound at $P_{0}$:


```r
obs <- draw_from_experiment(B)
(cramer_rao_hat <- var(eic(obs, psi = psi_hat)))
```

```
## [1] 0.2553555
```

and the Cramér-Rao bound at $\Pi_{0}$:


```r
obs_from_another_experiment <- draw_from_another_experiment(B)
(cramer_rao_Pi_zero_hat <- var(eic(obs_from_another_experiment, psi = 59/300)))
```

```
## [1] 0.09414341
```

```r
(ratio <- sqrt(cramer_rao_Pi_zero_hat/cramer_rao_hat))
```

```
## [1] 0.6071868
```

We  thus  discover  that  of  the  statistical  parameters  $\Psi(P_{0})$  and
$\Psi(\Pi_{0})$,   the  latter   is  easier   to  target   than  the   former.
Heuristically, for  large sample  sizes, the narrowest  (efficient) confidence
intervals for  $\Psi(\Pi_{0})$ are approximately 0.61 (rounded
to two decimal places) smaller than their counterparts for $\Psi(P_{0})$.

\subsection{Revisiting Section~\ref{subsec:being:smooth:one}.}

It is not difficult either (though a little cumbersome) \tcg{(do we give the
proof? I'd rather not)} to verify  that $\{\Pi_{h} : h \in \interval{-1}{1}\}$
is a fluctuation  of $\Pi_{0}$ in the direction of  $\sigma_{0}$ (in the sense
of \eqref{eq:fluct}) given, up to a constant, by

\begin{align*}\sigma_{0}(O)  &\equiv -  10  \sqrt{W}  A \times  \beta_{0}(A,W)
\left(\log(1    -     Y)    +     \sum_{k=0}^{3}    \left(k     +    \beta_{0}
(A,W)\right)^{-1}\right)     +      \text{constant},\\     \text{where}     \;
\beta_{0}(A,W)&\equiv                                                  \frac{1
-\Qbar_{\Pi_{0}}(A,W)}{\Qbar_{\Pi_{0}}(A,W)}.\end{align*}

Consequently,    the    slope    of     the    dotted    curve    in    Figure
\@ref(fig:psi-approx-psi-one) is equal to 

\begin{equation}\label{eq:slope:Pi}E_{\Pi_{0}}       (D^{*}(\Pi_{0})       (O)
\sigma_{0}(O))\end{equation}

(since $D^{*}(\Pi_{0})$  is centered under $\Pi_{0}$,  knowing $\sigma_{0}$ up
to a constant is not problematic). 

Let  us check  this numerically.   In  the next  chunk of  code, we  implement
direction  $\sigma_{0}$  with `sigma0_draw_from_another_experiment`,  then  we
numerically approximate  \eqref{eq:slope:Pi} (pointwise and with  a confidence
interval of asymptotic level 95\%):


```r
sigma0_draw_from_another_experiment <- function(obs) { 
  ## preliminary
  Qbar <- attr(obs, "Qbar")
  QAW <- Qbar(obs[, c("A", "W")])
  shape1 <- Arguments$getInteger(attr(obs, "shape1"), c(1, Inf))
  ## computations
  betaAW <- shape1 * (1 - QAW) / QAW
  out <- log(1 - obs[, "Y"])
  for (int in 1:shape1) {
    out <- out + 1/(int - 1 + betaAW)
  }
  out <- - out * shape1 * (1 - QAW) / QAW * 10 * sqrt(obs[, "W"]) * obs[, "A"]
  ## no need to center given how we will use it
  return(out)
}

vars <- eic(obs_from_another_experiment, psi = 59/300) *
  sigma0_draw_from_another_experiment(obs_from_another_experiment)
sd_hat <- sd(vars)
(slope_hat <- mean(vars))
```

```
## [1] 1.357245
```

```r
(slope_CI <- slope_hat + c(-1, 1) * qnorm(1 - alpha / 2) * sd_hat / sqrt(B))
```

```
## [1] 1.340548 1.373941
```

Equal to  1.349 (rounded to  three decimal  places), the
first numerical approximation `slope_approx` is not too off.

\subsection{Double-robustness}
\label{subsec:double:robustness}

The  efficient influence  curve $D^{*}(P)$  at  $P \in  \calM$ enjoys  another
remarkable property: it is double-robust.   Specifically, if we define for all
$P' \in \calM$

\begin{equation}\label{eq:rem:one} \Rem_{P} (\Qbar',  \Gbar')\equiv \Psi(P') -
\Psi(P) + E_{P} (D^{*}(P') (O)), \end{equation}

then   the   so   called    remainder   term   $\Rem_{P}   (\Qbar',   \Gbar')$
satisfies\footnote{For  any   (measurable)  $f:\calO  \to  \bbR$,   we  denote
$\|f\|_{P} = E_{P} (f(O)^{2})^{1/2}$.}

\begin{equation}\label{eq:rem:two}   \Rem_{P}    (\Qbar',   \Gbar')^{2}   \leq
\|\Qbar'  - \Qbar\|_{P}^{2}  \times  \|(\Gbar' -  \Gbar)/\ell\Gbar'\|_{P}^{2}.
\end{equation}

In particular, if

\begin{equation}\label{eq:solves:eic} E_{P} (D^{*}(P') (O)) =
0,\end{equation}

and  \textit{either}  $\Qbar' =  \Qbar$  \textit{or}  $\Gbar' =  \Gbar$,  then
$\Rem_{P} (\Qbar', \Gbar') = 0$ hence $\Psi(P') = \Psi(P)$.  In words, if $P'$
solves  the   so  called  $P$-specific  efficient   influence  curve  equation
\eqref{eq:solves:eic} and if, in addition, $P'$ has the same $\Qbar$-component
or $\Gbar$-component as $P$, then $\Psi(P')  = \Psi(P)$ no matter how $P'$ may
differ  from  $P$ otherwise.  This  property  is  useful to  build  consistent
estimators of $\Psi(P)$.

However,   there  is   much   more  to   double-robustness   than  the   above
straightforward  implication. Indeed,  \ref{eq:rem:one} is  useful to  build a
consistent etimator of $\Psi(P)$ that,  in addition, satisfies a central limit
theorem and thus lends itsef to the construction of confidence intervals.

Let $P_{n}^{0} \in \calM$  be an element of model $\calM$  of which the choice
is data-driven, based  on observing $n$ independent draws  from $P$.  Equality
\ref{eq:rem:one} reveals  that the  statistical behavior of  the corresponding
\textit{substitution}  estimator  $\psi_{n}^{0}   \equiv  \Psi(P_{n}^{0})$  is
easier  to   analyze  when   the  remainder  term   $\Rem_{P}  (\Qbar_{n}^{0},
\Gbar_{n}^{0})$ goes  to zero  at a  fast (relative to  $n$) enough  rate.  In
light of  \ref{eq:rem:two}, this happens  if the features  $\Qbar_{n}^{0}$ and
$\Gbar_{n}^{0}$ of  $P_{n}^{0}$ converge  to their  counterparts under  $P$ at
rates of which \textit{the product} is fast enough.

\subsection[Inference assuming $\Gbar_{0}$ known, or not, first pass.]{Inference assuming
$\boldsymbol{\Gbar_{0}}$ known, or not, first pass.} 
\label{subsec:known:gbar:one}

Let $O_{1}$,  \ldots, $O_{n}$  be a sample  of independent  observations drawn
from   $P_{0}$.   Let   $P_{n}$  be   the  corresponding   empirical  measure,
\textit{i.e.},  the  law consisting  in  drawing  one among  $O_{1}$,  \ldots,
$O_{n}$ with equal probabilities $n^{-1}$. 

Let us  assume for a moment  that we know  $\Gbar_{0}$.  This may be  the case
indeed if  $P_{0}$ was a  controlled experiment.  Note that, on  the contrary,
assuming $\Qbar_{0}$ known would be difficult to justify. 


```r
Gbar <- attr(obs, "Gbar")

iter <- 1e3
```

Then, the alternative expression \begin{equation}\label{eq:psi0:b} \psi_{0} =
E_{P_{0}}     \left(\frac{2A-1}{\ell\Gbar_{0}(A,W)}Y\right)     \end{equation}
suggests            to           estimate            $\psi_{0}$           with
\begin{equation}\label{eq:psi:n:b}\psi_{n}^{b}         \equiv        E_{P_{n}}
\left(\frac{2A-1}{\ell\Gbar_{0}(A,W)}Y\right)  =   \frac{1}{n}  \sum_{i=1}^{n}
\left(\frac{2A_{i}-1}{\ell\Gbar_{0}(A_{i},W_{i})}Y_{i}\right).\end{equation}
Note how $P_{n}$ is substituted  for $P_{0}$ in \eqref{eq:psi:n:b} relative to
\eqref{eq:psi0:b}. 

It is easy to check that $\psi_{n}^{b}$ estimates $\psi_{0}$ consistently, but
this  is too  little  to request  from an  estimator  of $\psi_{0}$.   Better,
$\psi_{n}^{b}$   also   satisfies   a   central   limit   theorem:   $\sqrt{n}
(\psi_{n}^{b} -  \psi_{0})$ converges in law  to a centered Gaussian  law with
asymptotic     variance     \begin{equation*}v^{b}     \equiv     \Var_{P_{0}}
\left(\frac{2A-1}{\ell\Gbar_{0}(A,W)}Y\right),\end{equation*}   where  $v^{b}$
can    be    consistently    estimated   by    its    empirical    counterpart
\begin{equation}\label{eq:v:n:b}      v_{n}^{b}       \equiv      \Var_{P_{n}}
\left(\frac{2A-1}{\ell\Gbar_{0}(A,W)}Y\right)  =   \frac{1}{n}  \sum_{i=1}^{n}
\left(\frac{2A_{i}-1}{\ell\Gbar_{0}(A_{i},W_{i})}Y_{i}                       -
\psi_{n}^{b}\right)^{2}.\end{equation}

Let us investigate how $\psi_{n}^{b}$ behaves  based on `obs`.  Because we are
interested  in the  \textit{law} of  $\psi_{n}^{b}$,  the next  chunk of  code
constitutes `iter =` 1000  independent samples of independent observations
drawn from $P_{0}$, each consisting of $n$ equal to `nrow(obs)/iter =` 
100 data points, and computes the realization of $\psi_{n}^{b}$
on all samples.

Before  proceeding,   let  us  introduce   \begin{align*}\psi_{n}^{a}  &\equiv
E_{P_{n}}  \left(Y  |  A=1\right)  -  E_{P_{n}} \left(Y  |  A=0\right)  \\  &=
\frac{1}{n_{1}}   \sum_{i=1}^{n}  \one\{A_{i}=1\}   Y_{i}  -   \frac{1}{n_{0}}
\sum_{i=1}^{n} \one\{A_{i}=0\} Y_{i}  \\&=\frac{1}{n_{1}} \sum_{i=1}^{n} A_{i}
Y_{i} - \frac{1}{n_{0}}  \sum_{i=1}^{n} (1 - A_{i})  Y_{i}, \end{align*} where
$n_{1}  = \sum_{i=1}^{n}  A_{i} =  n -  n_{0}$ is  the number  of observations
$O_{i}$    such   that    $A_{i}   =    1$.    It   is    an   estimator    of
\begin{equation*}E_{P_{0}} (Y | A=1) -  E_{P_{0}} (Y | A=0).\end{equation*} We
seize  this  opportunity to  demonstrate  numerically  the obvious  fact  that
$\psi_{n}^{a}$ does not estimate $\psi_{0}$. 



```r
psi_hat_ab <- obs %>% as_tibble() %>% mutate(id = 1:n() %% iter) %>%
  mutate(lgAW = A * Gbar(W) + (1 - A) * (1 - Gbar(W))) %>% group_by(id) %>%
  summarize(est_a = mean(Y[A==1]) - mean(Y[A==0]),
            est_b = mean(Y * (2 * A - 1) / lgAW),
            std_b = sd(Y * (2 * A - 1) / lgAW) / sqrt(n()),
            clt_b = (est_b - psi_hat) / std_b)
std_a <- sd(psi_hat_ab$est_a)
psi_hat_ab <- psi_hat_ab %>%
  mutate(std_a = std_a,
         clt_a = (est_a - psi_hat) / std_a) %>% 
  gather(key, value, -id) %>%
  extract(key, c("what", "type"), "([^_]+)_([ab])") %>%
  spread(what, value)

(bias_ab <- psi_hat_ab %>% group_by(type) %>% summarise(bias = mean(clt)))
```

```
## # A tibble: 2 x 2
##   type     bias
##   <chr>   <dbl>
## 1 a     -0.258 
## 2 b      0.0961
```

```r
debug(ggplot2::stat_density)
fig <- ggplot() +
  geom_line(aes(x = x, y = y), 
            data = tibble(x = seq(-3, 3, length.out = 1e3),
                          y = dnorm(x)),
            linetype = 1, alpha = 0.5) +
  geom_density(aes(clt, fill = type, colour = type),
               psi_hat_ab, alpha = 0.1) +
  geom_vline(aes(xintercept = bias, colour = type),
             bias_ab, size = 1.5, alpha = 0.5)
  
fig +
  labs(x = expression(paste(sqrt(n/v[n]^{list(a, b)})*(psi[n]^{list(a, b)} - psi[0]))))
```

![Kernel density estimators of the law of two estimators of $\psi_{0}$ (recentered and renormalized), one of them misconceived (a), the other assuming that $\Gbar_{0}$ is known (b). Built based on `iter` independent realizations of each estimator.](img/known-Gbar-one-b-1.png)

Let $v_{n}^{a}$ be $n$ times the empirical variance of the `iter` realizations
of   $\psi_{n}^{a}$.   By   the  above   chunk  of   code,  the   averages  of
$\sqrt{n/v_{n}^{a}}   (\psi_{n}^{a}  -   \psi_{0})$  and   $\sqrt{n/v_{n}^{b}}
(\psi_{n}^{b}  -  \psi_{0})$  computed  across the  realizations  of  the  two
estimators are respectively equal to 
-0.258 and 
0.096 (both rounded to three decimal
places  ---  see  `bias_ab`).   Interpreted  as amounts  of  bias,  those  two
quantities    are     represented    by     vertical    lines     in    Figure
\@ref(fig:known-Gbar-one-b). The red and blue bell-shaped curves represent the
empirical   laws  of   $\psi_{n}^{a}$  and   $\psi_{n}^{b}$  (recentered   and
renormalized) as estimated  by kernel density estimation. The  latter is close
to the black curve, which represents the standard normal density.

\subsection[Inference assuming $\Gbar_{0}$ known, or not, second pass.]{Inference assuming
$\boldsymbol{\Gbar_{0}}$ known, or not, second pass.} 
\label{subsec:known:gbar:two}


At the beginning of Section \ref{subsec:known:gbar:one}, we assumed that $\Gbar_{0}$ was
known. Let us suppose now that it is not. The definition of $\psi_{n}^{b}$ can
be  adapted to  overcome  this  difficulty, by  substituting  an estimator  of
$\ell\Gbar_{0}$ for $\ell\Gbar_{0}$ in \eqref{eq:psi:n:b}. 

For  simplicity,  we  consider  the  case that  $\Gbar_{0}$  is  estimated  by
minimizing   a   loss    function   on   a   single    working   model,   both
fine-tune-parameter-free.   By adopting  this  stance,  we exclude  estimating
procedures that involve penalization  (\textit{e.g.} the LASSO) or aggregation
of competing estimators (\textit{via}  stacking/super learning) -- see Section
\ref{subsec:exo:one}.  Defined in the next chunk of code, the generic function
`estimate_G` fits a  user-specified working model by  minimizing the empirical
risk associated to the user-specified loss function and provided data, and the
generic  function  `predict_lGAW`  (merely  a  convenient  wrapper)  estimates
$\ell\Gbar_{0}(A,W)$ for any $(A,W)$ based on the output of `estimate_G`.



```r
estimate_G <- function(dat, algorithm, ...) {
  if (!attr(algorithm, "ML")) {
    fit <- algorithm[[1]](formula = algorithm[[2]], data = dat)
    Ghat <- function(newdata) {
      predict(fit, newdata, type = "response")
    }
  } else {
    fit <- algorithm(dat, ...)
    Qhat <- function(newdata) {
      caret::predict.train(fit, newdata)
    }
  }
  return(Ghat)
}

predict_lGAW <- function(A, W, algorithm, threshold = 0.05, ...) {
  ## a wrapper to use in a call to 'mutate'
  ## (a) fit the working model
  dat <- data.frame(A = A, W = W)
  Ghat <- estimate_G(dat, algorithm, ...)
  ## (b) make predictions based on the fit
  Ghat_W <- Ghat(dat)
  lGAW <- A * Ghat_W + (1 - A) * (1 - Ghat_W)
  pmin(1 - threshold, pmax(lGAW, threshold))
}
```

\tcg{Comment on new structure of} `estimate_G`.

Note how the prediction of any $\ell\Gbar_{0}(A,W)$ is manually bounded away
from 0 and 1 at the last line of `predict_lGAW`. This is desirable because the
\textit{inverse}   of  each   $\ell\Gbar_{0}(A_{i},W_{i})$   appears  in   the
definition of $\psi_{n}^{b}$ \eqref{eq:psi:n:b}.

For sake of illustration, we choose argument `working_model_G_one` of function
`estimate_G` as follows:


```r
working_model_G_one <- list(
  model = function(...) {glm(family = binomial(), ...)},
  formula = as.formula(
    paste("A ~",
          paste("I(W^", seq(1/2, 2, by = 1/2), sep = "", collapse = ") + "),
          ")")
  ))
attr(working_model_G_one, "ML") <- FALSE
working_model_G_one$formula
```

```
## A ~ I(W^0.5) + I(W^1) + I(W^1.5) + I(W^2)
```

In  words, we  choose  the  so called  logistic  (or  negative binomial)  loss
function    $L_{a}$    given   by    \begin{equation}    \label{eq:logis:loss}
-L_{a}(f)(A,W) \equiv A \log f(W) + (1 - A) \log (1 - f(W)) \end{equation} for
any function $f : [0,1] \to [0,1]$ paired with the working model $\calF \equiv
\left\{f_{\theta} :  \theta \in \bbR^{5}\right\}$  where, for any  $\theta \in
\bbR^{5}$,  $\logit   f_{\theta}  (W)   \equiv  \theta_{0}   +  \sum_{j=1}^{4}
\theta_{j}  W^{j/2}$. The  working model  is well  specified: it  happens that
$\Gbar_{0}$  is the  unique minimizer  of the  risk entailed  by $L_{a}$  over
$\calF$: \begin{equation*}\Gbar_{0} = \mathop{\arg\min}_{f_{\theta} \in \calF}
E_{P_{0}}  \left(L_{a}(f_{\theta})(A,W)\right).\end{equation*} Therefore,  the
estimator $\Gbar_{n}$  output by `estimate_G`  and obtained by  minimizing the
empirical risk \begin{equation*} E_{P_{n}} \left(L_{a}(f_{\theta})(A,W)\right)
=  \frac{1}{n}   \sum_{i=1}^{n}  L_{a}(f_{\theta})(A_{i},W_{i})\end{equation*}
over $\calF$ consistently estimates $\Gbar_{0}$.

In light of  \eqref{eq:psi:n:b}, introduce \begin{equation}\psi_{n}^{c} \equiv
\frac{1}{n}   \sum_{i=1}^{n}   \left(\frac{2A_{i}  -   1}{\ell\Gbar_{n}(A_{i},
W_{i})}   Y_{i}\right).\end{equation}   Because  $\Gbar_{n}$   minimizes   the
empirical  risk over  a finite-dimensional  and well-specified  working model,
$\sqrt{n} (\psi_{n}^{c} -  \psi_{0})$ converges in law to  a centered Gaussian
law. Let us compute $\psi_{n}^{c}$ on the  same `iter = ` 1000 independent
samples  of  independent  observations  drawn   from  $P_{0}$  as  in  Section
\ref{subsec:known:gbar:one}:


```r
psi_hat_c <- obs %>% as_tibble() %>% mutate(id = 1:n() %% iter) %>%
  group_by(id) %>%
  mutate(lgAW = predict_lGAW(A, W, working_model_G_one)) %>%
  summarize(est = mean(Y * (2 * A - 1) / lgAW),
            try = sd(Y * (2 * A - 1) / lgAW) / sqrt(n()))
std_c <- sd(psi_hat_c$est)
psi_hat_abc <- psi_hat_c %>%
  mutate(std = std_c,
         try = try,
         clt = (est - psi_hat) / std,
         type = "c") %>%
  full_join(psi_hat_ab)

(bias_abc <- psi_hat_abc %>% group_by(type) %>% summarise(bias = mean(clt)))
```

```
## # A tibble: 3 x 2
##   type      bias
##   <chr>    <dbl>
## 1 a     -0.258  
## 2 b      0.0961 
## 3 c      0.00724
```

Note how we exploit the independent realizations of $\psi_{n}^{c}$ to estimate
the  asymptotic variance  of the  estimator with  $v_{n}^{c}/n$. By  the above
chunk of code,  the average of $\sqrt{n/v_{n}^{c}}  (\psi_{n}^{c} - \psi_{0})$
computed across the realizations is equal to 
0.007 (rounded to three decimal places
--- see  `bias_abc`). We represent  the empirical  laws of the  recentered and
renormalized  $\psi_{n}^{a}$,  $\psi_{n}^{b}$  and $\psi_{n}^{c}$  in  Figures
\@ref(fig:unknown-Gbar-three)     (kernel      density     estimators)     and
\@ref(fig:unknown-Gbar-four) (quantile-quantile plots).


```r
fig +
  geom_density(aes(clt, fill = type, colour = type), psi_hat_abc, alpha = 0.1) +
  geom_vline(aes(xintercept = bias, colour = type),
             bias_abc, size = 1.5, alpha = 0.5) +
  xlim(-3, 3) + 
  labs(x = expression(paste(sqrt(n/v[n]^{list(a, b, c)})*
                            (psi[n]^{list(a, b, c)} - psi[0]))))
```

![Kernel density estimators of the law of three estimators of $\psi_{0}$  (recentered and renormalized), one of them misconceived (a), one assuming that $\Gbar_{0}$ is known (b) and one that hinges on the estimation of $\Gbar_{0}$ (c). The present figure includes Figure \@ref(fig:known-Gbar-one-b) (but the colors differ). Built based on `iter` independent realizations of each estimator.](img/unknown-Gbar-three-1.png)


```r
ggplot(psi_hat_abc, aes(sample = clt, fill = type, colour = type)) +
  geom_abline(intercept = 0, slope = 1, alpha = 0.5) +
  geom_qq(alpha = 1)
```

![Quantile-quantile plot of the standard normal law against the empirical laws  of three estimators of $\psi_{0}$, one of them misconceived (a), one assuming that $\Gbar_{0}$ is known (b) and one that hinges on the estimation of $\Gbar_{0}$ (c). Built based on `iter` independent realizations of each estimator.](img/unknown-Gbar-four-1.png)

Figures \@ref(fig:unknown-Gbar-three)  and \@ref(fig:unknown-Gbar-four) reveal
that $\psi_{n}^{c}$ behaves as well as $\psi_{n}^{b}$ --- but remember that we
did not discuss how to estimate its asymptotic variance.

\subsection{\gear Exercises.}
\label{subsec:exo:one}

The questions are asked in the context of Sections \ref{subsec:known:gbar:one}
and \ref{subsec:known:gbar:two}.

1. Building  upon the  piece of  code devoted to  the repeated  computation of
$\psi_{n}^{b}$ and  its companion  quantities, construct  confidence intervals
for  $\psi_{0}$ of  (asymptotic)  level  $95\%$, and  check  if the  empirical
coverage is satisfactory.  Note that if  the coverage was exactly $95\%$, then
the number of confidence intervals  that would contain $\psi_{0}$ would follow
a binomial  law with parameters  `iter` and  `0.95`, and recall  that function
`binom.test` performs  an exact  test of  a simple  null hypothesis  about the
probability of success  in a Bernoulli experiment against  its three one-sided
and two-sided alternatives.

2.  The wrapper `predict_lGAW` makes predictions by fitting a working model on
the same data points as those for which predictions are sought. Why could that
be problematic? Can you think of a simple workaround, implement and test it?

3.  Discuss what happens when the dimension of the (still well-specified)
working model grows. You could use the following chunk of code

```r
powers <- ## make sure '1/2' and '1' belong to 'powers', eg
  seq(1/4, 3, by = 1/4)
working_model_G_two <- list(
  model = function(...) {glm(family = binomial(), ...)},
  formula = as.formula(
    paste("A ~",
          paste("I(W^", powers, sep = "", collapse = ") + "),
          ")")
  ))
attr(working_model_G_two, "ML") <- FALSE
```
play around with  argument `powers` (making sure that `1/2`  and `1` belong to
it),   and   plot   graphics   similar   to   those   presented   in   Figures
\@ref(fig:unknown-Gbar-three) and \@ref(fig:unknown-Gbar-four). 

4. Discuss  what happens when the  working model is mis-specified.   You could
use the following chunk of code:

```r
transform <- c("cos", "sin", "sqrt", "log", "exp")
working_model_G_three <- list(
  model = function(...) {glm(family = binomial(), ...)},
  formula = as.formula(
    paste("A ~",
          paste("I(", transform, sep = "", collapse = "(W)) + "),
          "(W))")
  ))
attr(working_model_G_three, "ML") <- FALSE
(working_model_G_three$formula)
```

```
## A ~ I(cos(W)) + I(sin(W)) + I(sqrt(W)) + I(log(W)) + I(exp(W))
```

5.   \textdbend  Drawing inspiration  from \eqref{eq:v:n:b}, one  may consider
estimating the asymptotic  variance of $\psi_{n}^{c}$ with  the counterpart of
$v_{n}^{b}$ obtained  by substituting  $\ell\Gbar_{n}$ for  $\ell\Gbar_{0}$ in
\eqref{eq:v:n:b}.   By adapting  the piece  of  code devoted  to the  repeated
computation of  $\psi_{n}^{b}$ and its  companion quantities, discuss  if that
would be legitimate.



\subsection{Targeted inference.}
\label{subsec:tmle}


```r
estimate_Q <- function(dat, algorithm, ...) {
  if (!attr(algorithm, "ML")) {
    fit <- algorithm[[1]](formula = algorithm[[2]], data = dat)
    Qhat <- function(newdata) {
      predict(fit, newdata, type = "response")
    }
  } else {
    fit <- algorithm(dat, ...)
    Qhat <- function(newdata) {
      caret::predict.train(fit, newdata)
    }    
  }
  return(Qhat)
}

predict_QAW <- function(Y, A, W, algorithm, blip = FALSE, ...) {
  ## a wrapper to use in a call to 'mutate'
  ## (a) carry out the estimation based on 'algorithm'
  dat <- data.frame(Y = Y, A = A, W = W)
  Qhat <- estimate_Q(dat, algorithm, ...)
  ## (b) make predictions based on the fit
  if (!blip) {
    pred <- Qhat(dat)
  } else {
    pred <- Qhat(data.frame(A = 1, W = W)) - Qhat(data.frame(A = 0, W = W))
  }
  return(pred)
}

working_model_Q_one <- list(
  model = function(...) {glm(family = binomial(), ...)},
  formula = as.formula(
    paste("Y ~ A * (",
          paste("I(W^", seq(1/2, 2, by = 1/2), sep = "", collapse = ") + "),
          "))")
  ))
attr(working_model_Q_one, "ML") <- FALSE
working_model_Q_one$formula

## k-NN
kknn_algo <- function(dat, ...) {
  args <- list(...)
  if ("Subsample" %in% names(args)) {
    keep <- sample.int(nrow(dat), args$Subsample)
    dat <- dat[keep, ]
  }
  caret::train(Y ~ I(10*A) + W, ## a tweak
               data = dat,
               method = "kknn",
               verbose = FALSE,
               ...)
}
attr(kknn_algo, "ML") <- TRUE
kknn_grid <- expand.grid(kmax = c(3, 5), distance = 2, kernel = "gaussian")
control <- trainControl(method = "cv", number = 2,
                        predictionBounds = c(0, 1),
                        allowParallel = TRUE)

psi_hat_de <- obs %>% as_tibble() %>% mutate(id = 1:n() %% iter) %>%
  group_by(id) %>%
  mutate(blipQW_d = predict_QAW(Y, A, W, working_model_Q_one, blip = TRUE),
         blipQW_e = predict_QAW(Y, A, W, kknn_algo, blip = TRUE,
                                trControl = control,
                                tuneGrid = kknn_grid,
                                Subsample = 100)) %>%
  summarize(est_d = mean(blipQW_d),
            est_e = mean(blipQW_e))

std_d <- sd(psi_hat_de$est_d)
std_e <- sd(psi_hat_de$est_e)
psi_hat_de <- psi_hat_de %>%
  mutate(std_d = std_d,
         clt_d = (est_d - psi_hat) / std_d,
         std_e = std_e,
         clt_e = (est_e - psi_hat) / std_e) %>% 
  gather(key, value, -id) %>%
  extract(key, c("what", "type"), "([^_]+)_([de])") %>%
  spread(what, value)

(bias_de <- psi_hat_de %>% group_by(type) %>% summarize(bias = mean(clt)))

fig <- ggplot() +
  geom_line(aes(x = x, y = y), 
            data = tibble(x = seq(-3, 3, length.out = 1e3),
                          y = dnorm(x)),
            linetype = 1, alpha = 0.5) +
  geom_density(aes(clt, fill = type, colour = type),
               psi_hat_de, alpha = 0.1) +
  geom_vline(aes(xintercept = bias, colour = type),
             bias_de, size = 1.5, alpha = 0.5)
  
fig +
  labs(x = expression(paste(sqrt(n/v[n]^{list(d, e)})*(psi[n]^{list(d, e)} - psi[0]))))
```

For later$\ldots{}$


```r
working_model_Q_two <- list(
  model = function(...) {glm(family = binomial(), ...)},
  formula = as.formula(
    paste("Y ~ A * (",
          paste("I(W^", seq(1/2, 3, by = 1/2), sep = "", collapse = ") + "),
          "))")
  ))
attr(working_model_Q_two, "ML") <- FALSE

## xgboost based on trees
xgb_tree_algo <- function(dat, ...) {
  caret::train(Y ~ I(10*A) + W,
               data = dat,
               method = "xgbTree",
               trControl = control,
               tuneGrid = grid,
               verbose = FALSE)  
}
attr(xgb_tree_algo, "ML") <- TRUE
xgb_tree_grid <- expand.grid(nrounds = 350,
                             max_depth = c(4, 6),
                             eta = c(0.05, 0.1),
                             gamma = 0.01,
                             colsample_bytree = 0.75,
                             subsample = 0.5,
                             min_child_weight = 0)

## nonparametric kernel smoothing regression
npreg <- list(
  label = "Kernel regression",
  type = "Regression",
  library = "np",
  parameters = data.frame(parameter =
                            c("subsample", "regtype",
                              "ckertype", "ckerorder"), 
                          class = c("integer", "character",
                                    "character", "integer"), 
                          label = c("#subsample", "regtype",
                                    "ckertype", "ckerorder")),
  grid = function(x, y, len = NULL, search = "grid") {
    if (!identical(search, "grid")) {
      stop("No random search implemented.\n")
    } else {
      out <- expand.grid(subsample = c(50, 100),
                         regtype = c("lc", "ll"),
                         ckertype =
                           c("gaussian",
                             "epanechnikov",
                             "uniform"),
                         ckerorder = seq(2, 8, 2))
    } 
    return(out)
  },
  fit = function(x, y, wts, param, lev, last, classProbs, ...) {
    ny <- length(y)
    if (ny > param$subsample) {
      ## otherwise far too slow for what we intend to do here...
      keep <- sample.int(ny, param$subsample)
      x <- x[keep, ]
      y <- y[keep]
    }
    bw <- np::npregbw(xdat = as.data.frame(x), ydat = y,
                      regtype = param$regtype,
                      ckertype = param$ckertype,
                      ckerorder = param$ckerorder,
                      remin = FALSE, ftol = 0.01, tol = 0.01,
                      ...)
    np::npreg(bw)
  },
  predict = function (modelFit, newdata, preProc = NULL, submodels = NULL) {
    if (!is.data.frame(newdata)) {
      newdata <- as.data.frame(newdata)
    }
    np:::predict.npregression(modelFit, se.fit = FALSE, newdata)
  },
  sort = function(x) {
    x[order(x$regtype, x$ckerorder), ]
  },
  loop = NULL, prob = NULL, levels = NULL
)

npreg_algo <- function(dat, ...) {
  caret::train(working_model_Q_one$formula,
               data = dat,
               method = npreg, # no quotes!
               verbose = FALSE,
               ...)
}
attr(npreg_algo, "ML") <- TRUE
npreg_grid <- data.frame(subsample = 100,
                         regtype = "lc",
                         ckertype = "gaussian",
                         ckerorder = 4,
                         stringsAsFactors = FALSE)
```
-->