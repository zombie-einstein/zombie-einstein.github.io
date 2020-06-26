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

I have to admit I find cellular automaton (CA) endlessly fascinating. They seem 
to have fallen out of favour, and have never seemed to have any killer 
real-world applications. But I find the questions they raise about emergent 
behaviour and self organization incredibly engaging. 

I'll not go into much depth of the theory on CA in this post, 
but there's a ton of stuff out there and as always 
[wikipedia](https://en.wikipedia.org/wiki/Cellular_automaton) is a good place
to start.

I'll mainly be discussing 1-dimensional CA. This can be thought
of as a 1-d array of cells, with each cell in a (usually discrete) state
$s\in S$. At each update of the model, the cell updates it's state based
on it's state and local neighbourhood. For example if each cell only considers
it's nearest neighbours then the update rule for the CA is
the mapping

 $$(s_{i-1}^{t}, s_{i-1}^{t}, s_{i+1}^{t}) \rightarrow s_{i}^{t+1}$$
 
 where $i$ indexes the cells position in the array. The array is usually 
 wrapped to form a closed loop, and the space time behaviour of the 
 CA shown as a 2-d array like where each row is one step of the model 

{:refdef: style="text-align: center;"}
![](https://upload.wikimedia.org/wikipedia/commons/9/9d/CA_rule30s.png)
{: refdef}

In many cases it's convenient index rules using the wolfram system where the
index is derived from the mapping representation in base $|S|$. For example
if $S=\\{0,1\\}$ then full describing the update rule requires a mapping for
the $2^3$ possible states resulting in $2^8=256$ possible rules see 
[here](https://en.wikipedia.org/wiki/Wolfram_code) for more details.

## Motivation

This project was motivated by a couple of broad questions about 1-d CA:

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
  
## Theory
 *Note: In the literature "Probabilistic CA" seems to refer to a model where
 the update rule is probabilistic but each cell still always has a fixed
 discrete state. So for now the name "Continuous 
 Probabilistic CA" seems like a good name to distinguish from this* 

We'll consider a model where each cell has a probability of being in a state
$s$ denoted as 

$$P(s_{i}^{t})=P(s_{i}^{t}=s)\quad\text{where}\quad s\in S$$

The state of the cell at the next step is then given by

$$
P\left(s_{i}^{t+1}\right)=\sum_{f=s_{i}^{t+1}}P\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t}\right)
$$

where the r.h.s is the probability of a state of the preceding triple and the
summation is performed over the triples will result in the state $s$ 

$$f\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t}\right)=s_{i}^{t+1}$$

where $f(\dots)$ is the CA update function.

To do this for an arbitrary number of steps we need the joint probability of 
two neighbouring cells at each site and it's neighbour to the right i.e.

$$P\left(s_{i}^{t}, s_{i+1}^{t}\right)$$

extending the approach for the update of a single cell gives

$$
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\right)=\sum_{f=s_{i}^{t+1}, f=s_{i+1}^{t+1}}P\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t},s_{i+2}^{t}\right)
$$

the summation now over the overlapping triple cell states that will create the 
joint state. Then using the chain rule of probabilities we can decompose this
to

$$\begin{align}
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\right) &=\sum_{f} P\left(s_{i}^{t},s_{i+1}^{t}\right)P\left(s_{i-1}^{t}|s_{i}^{t}\right)P\left(s_{i+2}^{t}|s_{i+1}^{t}\right)\\
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

In the above model, any uncertainty in cell states can only arise from 
uncertainty in the initial state (if all the cells are in only one state 
initially i.e. $P(s)\in\\{0,1\\}$ it behaves like a regular CA), the update rule
still maps states to discrete states.

We can extend the model to include probabilistic updates, represented 
by the conditional distribution

$$
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:| \: q_{i}^{t}\right)
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

summing over the possible preceding quadruples. 

If 

$$
P\left(s_{i}^{t+1}, s_{i+1}^{t+1}\:| \: q_{i}^{t}\right) \in \{0,1\}
$$

then a deterministic CA rule is obtained.

## Implementation

- Generating an array of mappings from the quadruple of cells to the following tuples, i.e.

  $$\left(s_{i-1}^{t},s_{i}^{t},s_{i+1}^{t},s_{i+2}^{t}\right)\rightarrow\left(s_{i}^{t+1},s_{i+1}^{t+1}\right)$$   

  based on the update rule
- Create an empty array for joint probabilities in the shape 

  $$\text{steps}\times \text{width}\times \|S\|\times\|S\|$$
  
- Set the initial joint probability row from an initial state, using the independence of the initial states
- At each step calculate the marginal probabilities required to calculate $P\left(s_{i}^{t+1},s_{i+1}^{t+1}\right)$ along with left and right shifted arrays
- Using the mapping to sum the contributions from the previous row (and marginals)