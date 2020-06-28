---
layout: post
mathjax: true
title:  "Continuous Probabilistic Cellular Automata"
date:   2020-06-25 22:54:33 +0100
tags: cellular-automata python
---

*The code for this project can be found on it's 
[github repo](https://github.com/zombie-einstein/probabilistic_ca)*

## Introduction

I have to admit I find cellular automaton (CA) endlessly fascinating. They have 
never seemed to find that killer real-world applications or deep insight. But 
I find the questions they raise about emergent behaviour and self organization 
incredibly engaging. 

### 1D Cellular Automata (Briefly)
I'll not go into much depth of the theory on CA in this post, if you've not
encountered them before there's a ton of stuff out there and as always 
[wikipedia](https://en.wikipedia.org/wiki/Cellular_automaton) is a good place
to start.

This project focused on 1-dimensional CA. This can be thought
of as a 1d array of cells, with each cell in a (usually discrete) state
$s\in S$. At each update of the model, the cell updates its state based
on its own state and local neighbourhood. For example if each cell only 
considers it's nearest neighbours then the update rule for the CA is
the mapping

 $$(s_{i-1}^{t}, s_{i-1}^{t}, s_{i+1}^{t}) \rightarrow s_{i}^{t+1}$$
 
 where $i$ indexes the cells position in the array and $t$ the step. The array 
 is usually wrapped to form a closed loop, and the space time behaviour of the 
 CA visualized as a 2d array where each row is one step of the model 


![](https://upload.wikimedia.org/wikipedia/commons/9/9d/CA_rule30s.png){: .center-image }
> Space time diagram showing the evolution of rule 30 from an initial stats
> containing a single live cell. The evolution contains both structured and
> disordered chaotic regions

In many cases it's convenient to index rules using the wolfram system where the
index is derived from the mapping represented in base $\vert S\vert$. For 
example if $S=\\{0,1\\}$ then full specifying the update rule requires a 
mapping for the $\vert S\vert^{3}=2^3$ possible triple states resulting in 
$2^8=256$ possible rules. See [here](https://en.wikipedia.org/wiki/Wolfram_code) 
for more details.

## Motivation

This project was motivated by a couple of broad questions about 1d CA:

- Is there some way to classify the long term behaviour of CA? CA seem to 
  roughly fall into 4 categories from a random initial state
  
  - Evolve to a static homogeneous state
  - Periodic where cells oscillate between states at each step or stable 
    inhomogeneous structures 
  - Evolution to chaotic or seemingly noisy behaviour
  - Formation of localised persistent structures that can interact
  
  but how can these classifications be made rigorous and what forms the 
  boundary between these behaviours?
- In the case where structures form, how do they form and persist against
  the background of chaotic noise?
 
Interesting work has been done on how changes in the initial state change the
long term behaviour of the model (an analogue of the Lyapunov exponent) 
and how information propagates through the array. It would seem expedient then
to be able to express the initial state as a distribution across initial states
or as something like a random walk that evolves over time as the update rule
is applied to it.

This motivates being able to model a cellular automata where the state of a 
cell is probabilistic. As the update rule is applied to a neighbourhood of
of cells, this also requires that we are able to model the joint or conditional
probability of neighbouring cells. 
 
## Theory
> **Note:** *In the literature "Probabilistic CA" seems to refer to a model 
> where the update rule is probabilistic but each cell still always has a 
> fixed discrete state. So for now the name "Continuous 
> Probabilistic CA" seems like a good name to distinguish from this* 

### Probabilistic States
We'll consider a model where each cell has a probability of being in a state
$s$ denoted as 

$$P(s_{i}^{t})=P(s_{i}^{t}=s)\quad\text{where}\quad s\in S$$

The state of the cell at the next step (if we are only considering nearest 
neighbours) is then given by 

$$
P\left(s_{i}^{t+1}\right)=\sum_{f=s_{i}^{t+1}}P\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t}\right)
$$

where the r.h.s is the joint probability of a state of the preceding triple 
and the summation is performed over the triples will result in the 
state $s_{i}^{t+1}$, i.e. when

$$f\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t}\right)=s_{i}^{t+1}$$

where $f(\dots)$ is the CA update function.

This approach is ok for one step of the model but to repeatedly perform 
this update for an arbitrary number of steps we need the joint 
probability of two neighbouring cells at each site and it's neighbour to the 
right i.e.

$$P\left(s_{i}^{t}, s_{i+1}^{t}\right)$$

extending the approach for the update of a single cell gives

$$
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\right)=\sum_{f=s_{i}^{t+1}, f=s_{i+1}^{t+1}}P\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t},s_{i+2}^{t}\right)
$$

the summation now over the overlapping triple cell states that will create the 
joint state. Then using the chain rule of probabilities we can decompose this
to

$$\begin{align}
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\right) &=\sum_{f} P\left(s_{i}^{t},s_{i+1}^{t}\right)P\left(s_{i-1}^{t}\vert s_{i}^{t}\right)P\left(s_{i+2}^{t}\vert s_{i+1}^{t}\right)\\
&=\sum_{f} P\left(s_{i-1}^{t},s_{i}^{t}\right)P\left(s_{i+1}^{t},s_{i+2}^{t}\right)h\left(s_{i}^{t},s_{i+1}^{t}\right)
\end{align}$$

where

$$h\left(s_{i}^{t},s_{i+1}^{t}\right)=\frac{P\left(s_{i}^{t},s_{i+1}^{t}\right)}{P\left(s_{i}^{t}\right)P\left(s_{i+1}^{t}\right)}$$

This form is then useful as we need only store the joint probabilities
for each cell (and it's neighbour) and can obtain $h$ from this and
the marginal probabilities

For the starting state at $t=0$ we assume that the initial probabilities are 
independent such that the initial array of joint probabilities can be found 
using

$$P\left(s_{i}^{0},s_{i+1}^{0}\right)=P\left(s_{i}^{0}\right)P\left(s_{i+1}^{0}\right)$$

### Probabilistic Update Rules

In the above model, any uncertainty in the model can only arise from 
uncertainty in the initial state (if all the cells are in only one state 
initially i.e. $P(s)\in\\{0,1\\}$ it behaves like a regular CA), as the update 
rule still maps states to discrete states.

We can extend the model to include probabilistic updates, represented 
by the conditional distribution

$$
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:\vert\: q_{i}^{t}\right)
$$

where $q_{i}^{t}$ is the preceding quadruple of states that inform the
updated state of the neighbouring cells

$$
q_{i}^{t} = \left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t},s_{i+2}^{t}\right)
$$

this somewhat simplifies the summation to give

$$\begin{align}
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\right) &= \sum_{q_{i}^{t}}P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:|\:q_{i}^{t}\right)P\left(q_{i}^{t}\right)\\
&= \sum_{q_{i}^{t}}P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:|\:q_{i}^{t}\right)P\left(s_{i-1}^{t},s_{i}^{t}\right)P\left(s_{i+1}^{t},s_{i+2}^{t}\right)h\left(s_{i}^{t},s_{i+1}^{t}\right)
\end{align}$$

summing over all the possible preceding quadruples. 

In the case that

$$
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:\mid \: q_{i}^{t}\right) \in \{0,1\}
$$

then a deterministic CA rule is obtained.

## Implementation

This was straightforward enough to implement in numpy. As with most cellular 
automata models like this (where the update is done in parallel for all cells) 
the real trick is shifting the state array left and right to be able to 
vectorize the update step. 

The model is specified by 3 parameters (as in a regular CA):

- The initial state of the cells, in this case it has the dimensions
  $\text{steps}\times\vert S\vert$ specifying the initial probability 
  distribution of each cell
  
- The update rule, provided as a 2d array mapping the permutation of 
  triples to the probability of the update state:
  
  $$P\left(s_{i}^{t+1}\:\vert\: s_{i-1}^{t}, s_{i}^{t}, s_{i+1}^{t}\right)$$ 

- The number of steps to run the model for

In brief the modelling process follows the following steps:

- Use the rule array to generate an array representing the update of 
  joint probabilities conditioned on the preceding quadruples of cells

  $$P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:\vert\: q_{i}^{t}\right)$$   

- Initialize an empty array for joint probabilities with the shape 

  $$\text{steps}\times \text{width}\times \vert S \vert\times\vert S \vert$$
  
  (i.e. a joint probability distribution for each cell and for each step of
   the model)
  
- Set the initial joint probability row from the initial state, 
  using the independence of the initial states
- At each step calculate the marginal probabilities required to calculate 
  $h\left(s_{i}^{t+1},s_{i+1}^{t+1}\right)$ along with the left and right
  shifted rows for $P\left(s_{i-1}^{t},s_{i}^{t}\right)$ and
  $P\left(s_{i+1}^{t},s_{i+2}^{t}\right)$
- For each joint probability entry sum over the contributions from the 
  permutations for of the previous quadruple of cells
  
### Complexity and Numerical Underflow
 
 Computationally there are a couple of potential drawbacks
 
 - **Computational complexity:** Increasing the number of states increases both
   the storage space required and the complexity of the update calculation. 
   The number of states means scaling the array of joint probabilities like 
   $\vert S\vert^2$ for each cell. When then need to then perform the update
   for each of these new entities as well as including contributions for 
   the additional permutation of states which scale as $\vert S\vert^4$.
- **Numerical Underflow:** As with many models where probabilities are 
  repeatedly multiplied numerical underflow can occur. A common approach is
  to work with log probabilities and use techniques like the 
  [log-sum-exp trick](https://www.xarg.org/2016/06/the-log-sum-exp-trick-in-machine-learning/).
  In this case there a couple of things that make this tricky:
  - The lack of an well-defined 0 probability in log space prohibits using 
    binary states as inputs to the model
  - Calculating the marginal probabilities required in the update step still
    means moving between log and normal probabilities which does not aid in
    reducing numerical under/overflow
    
## Analysis/Plotting

Plotting the result of the model as a time-space diagram like a regular CA
requires aggregating the joint probability distribution of each cell in some 
manner, but I also looked to choose statistics that might reveal underlying
dynamics of the probabilistic CA:

- The probabilities for each cell are easily recovered as the marginals of
  the joint distribution
  
  $$P\left(s_{i}^{t}\right)=\sum_{s_{i+1}^{t}}P\left(s_{i}^{t}, s_{i+1}^{t}\right)$$

- The mutual information of the joint probabilities also seems like it should
  be a useful, giving something like mutual dependence between neighbouring
  cells. Here defined as
  
  $$ 
  I_{i}^{t} = \sum_{s_{i}^{t}, s_{i+1}^{t}}P\left(s_{i}^{t}, s_{i+1}^{t}\right)
  \text{log}\left(\frac{P\left(s_{i}^{t}, s_{i+1}^{t}\right)}{P\left(s_{i}^{t}\right)P\left(s_{i+1}^{t}\right)}\right)
  $$
  
An additional issue is that in many cases the relative difference between 
cells decreases over time, meaning plots can fail to adequately show 
features contained in resulting arrays. Applying min-max scaling across
rows of the array is one approach used to tackle this issue, though care should
be taken in how this is interpreted, given that small relative differences
could be a result of numerical precision.

## Results

Given the large number of potential parametrizations of the model, mixing
probabilistic update rules, and initial states I looked to focus on two simple
cases:

- Standard update rules (i.e. deterministic) with an randomly chosen discrete 
  initial state containing a single cell in a mixed state. Used to investigate 
  how the uncertainty from a single cell propagates through the state over 
  time.

- Update rules with the same perturbation applied to each mapping, and a 
  randomly chosen discrete initial state. For example if the undeterred rule
  maps $(0,0,0) \rightarrow 0$ then the perturbed probabilities are
  $P(0\vert 0,0,0)=1-\epsilon$ and $P(1\vert 0,0,0)=\epsilon$ (applied to all
  the permutations). This could be thought of as a probability of error when
  applying the update rule.

## Conclusion 