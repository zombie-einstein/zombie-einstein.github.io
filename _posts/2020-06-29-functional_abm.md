---
layout: post
mathjax: true
title:  "Functional ABM API"
date:   2020-06-29
tags: abm agent-based-modelling functional python
---

*The code for this project can be found on it's 
[github repo](https://github.com/zombie-einstein/functional_abm)*

## Introduction

Most agent based modeling (ABM) frameworks tend to make use of an 
object-oriented (OOP) principals, and probably with good reason. The agents  
in an ABM are generally stateful and thus natural candidates for 
representation by classes encapsulating state and functionality; and 
inheritance (or composition) allows common functionality to be reused.

This project was an experiment in creating a "functional" ABM API inspired
in part by computation graph building APIs. Generally speaking these allow
a developer to construct a computation graph from function definitions with
the work of linking everything together done in the background (something like
[dagster](https://github.com/dagster-io/dagster) is a good example). 

It was also motivated by a couple of things I've noticed when writing 
ABMs using OOP patterns (though these are very subjective!):

- Python has some really great OOP features, but it sometimes feels that the
  flexibility and dynamic nature of python makes using OOP patterns a bit 
  tedious when writing an ABM; where you often want interfaces and 
  encapsulation to be followed quite strictly. This is obviously less of a 
  concern in other typed/compiled languages, but then you lose the flexibility 
  and speed of work in python.
- I also feel I end up writing a lot of boilerplate code, and ABM APIs
  tend not to offer much apart from templates classes, or the patterns they do 
  recommend tend to be quite restrictive.

So the aim was to try and create and API that did a lot of work in the
background, and offered a clean and flexible API to build a model.

> **Disclaimer** *The design I ended up with did not turn out strictly
> "functional" as it still sometimes relies on updating objects in-place for 
> reasons that will be outlined in this post. Though I feel a proper functional
> approach is possible with some tweaks.*

## Theory

Very an broadly an agent based model consists of agents that interact in some
manner (possibly inside a simulated environment). As the model progress
the state of agents are updated depending on the state of other agents
and the environment. It's usually the job of the model designer to 
dictate how the agents behave and also how agents interact within the 
model/environment. 

At the core of this is the concept of representing the time evolution of the 
model as a causal graph. 

On the graph agents are represented by nodes, and edges represent causal 
dependence between agents. Each agent owns its own state, so altogether
the agents represent the model/simulation environment, and the graph how the 
components of the environment interact over time. 

Each node (i.e. agent) then has:

- *Nodes that precede it causally*: The state of these nodes then act as 
  inputs to the update function of the node. These nodes could also 
  be thought of as being the observed state of the model when the node updates
- Downstream nodes that the node can update. This allows the node to alter
  the environment outside the state it owns. 
    
{% include image.html 
url="/assets/functional_abm/causal_graph.png" 
description="Representation of an ABM as a causal graph:<br>
a) Agents are represented as nodes on a causal graph, directed edges 
represent where an agent is dependent on the state of other agents, and the 
direction of the arrows indicate the direction of time direction. 
Unlike a DAG computational graph the graph can be recursive with agent 
updating at multiple time-steps.<br>
b) Each agent(node) has nodes in its causal past that act as 
inputs/observations, and downstream nodes that it can alter. 
" %} 

If you are familiar with DAGS (or computational graphs, neural networks etc.)  
this will likely seem familiar with the differences being that the causal 
graph can be recursive (as agents can update multiple times/repeatedly) and 
the agents are stateful.

To represent this in a functional framework we break up an agent definition
into two components

- The state of the agent
- An update function that is called when the agent is updated

When the agent is updated, its update function is called with 
the states of the preceding agents and the current state of the agent
as arguments. The update function then returns the new state of the node, 
and any updates to the state of downstream nodes.  

{% include image.html 
url="/assets/functional_abm/update_function.png" 
description="Functional implementation of agent updates. When an agent is 
updated its update function is called, with the inputs being the upstream
nodes and current state of the agent and the outputs the new state of the 
node and any updates to downstream nodes." %}

## Implementation

### `@agent` Decorator

An agents behaviour can be defined by decorating an update function with 
the `@agent` decorator. The function should have the signature

{% highlight python %}
@agent(scheduler=scheduler)
def foo(t, antecedents, state, descendants):
    # Update state and descendants
    ...
    return next_event_time
{% endhighlight %}

When the update function is called the arguments will be 

- `t`: The current model time
- `antecedents`: Data structure containing the states of preceding nodes
- `state`: The state of the node that is updating
- `descendants`: Data structure containing states of descendant nodes
  that this node can update
  
and it returns one value, the time this agent will next update. 

The decorator argument `scheduler` is a class that controls when agents are 
called and should be provided when the agent is defined (provided as part of 
the package). 

In this example the decorator would create a new type (class) 
called `foo` wrapping the update function and allowing it to be used as a 
component of the model.

----
*As noted above this is where the 'functional' approach breaks down as a bit
as the update function is expected to update the `state` and `descendants` in 
place. This should be possible by instead having the function return 
new state and descendants*
*This was done here as in some cases it's not possible to store a reference
and assign a value to it. For example for numpy arrays if you try something*
{% highlight python %}
x = np.array([1,2,3])
y = x[:1]
y = np.array([10])
{% endhighlight %}
*this will not update the slice of `x` that `y` refers to, it will just change 
what y refers to. So in practice it's easier to pass states by reference and
update them in place*

----

### Model Initialization

Initializing the model is then done by creating instances of agent types
following the same signature of the update function, for example for the 
`foo` function we decorated above we might do

{% highlight python %}
# Initial states of the nodes
agent_state_1 = [0]
agent_state_2 = [1]

# Initialize instances of agent nodes
foo(0, agent_state_2, agent_state_1, {})
foo(1, agent_state_1, agent_state_2, {})
{% endhighlight %}

Where the `t` argument is the time of the agents first event. In the background
this would initialize agent nodes with `agent_state_2` be a precedent of 
`agent_state_1` and vice versa (and both nodes have no downstream dependents).

## Example

This is probably better explained with a more in depth example. We can 
implement a function that initializes and runs 
[Conway's game of life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) 
using this API. In full this looks like:

{% highlight python%}
def gol(steps, initial_state):
    
    scheduler = StepBasedScheduler(steps)
    history = []
    
    @agent(scheduler=scheduler)
    def cell(t, antecedents, state, descendants):
        
        live_neighbours = np.sum(antecedents) - antecedents[1,1]
        
        if live_neighbours < 2:
            new_state = 0
        elif live_neighbours == 2:
            new_state = state[0,0]
        elif live_neighbours == 3:
            new_state = 1
        elif live_neighbours > 3:
            new_state = 0
        
        state[0:1,0:1] = new_state
        
        return t + 1
    
    # Initialize the nodes on a grid        
    for i in range(1, initial_state.shape[0]-1):
        for j in range(1, initial_state.shape[1]-1):
            cell(0, 
                 initial_state[i-1: i+2, j-1: j+2],
                 initial_state[i:i+1, j:j+1],
                 {})
    
    # Run the model for the requested number of steps and 
    # at each step store a copy of the array
    while not scheduler.finished:
        history.append(initial_state.copy())
        scheduler.step()
        
    return history {% endhighlight %}

This function accepts the initial state of the model (a numpy array) and the
number of steps to run the model for. Breaking it down

- This line initializes a scheduler that will run the model for a fixed number 
  of steps 
  {% highlight python%}scheduler = StepBasedScheduler(steps){% endhighlight %}
- The agent behaviour (an agent is a cell on the array in this case) is defined 
  using the decorator
  {% highlight python%}
  @agent(scheduler=scheduler)
      def cell(t, antecedents, state, descendants):
          live_neighbours = np.sum(antecedents) - antecedents[1,1]
        
        if live_neighbours < 2:
            new_state = 0
        elif live_neighbours == 2:
            new_state = state[0,0]
        elif live_neighbours == 3:
            new_state = 1
        elif live_neighbours > 3:
            new_state = 0
        
        state[0:1,0:1] = new_state
        
        return t + 1{% endhighlight %}
  as expected it looks at the previous state of the cells surrounding it 
  (its antecedents) and counts the number of live cells then updates its own
  state accordingly. It returns `t+1` which is the next of the model (all the
  cells will use this function so this will all the cells update each 
  step as desired).
- We then initialize the cells with the relevant antecedents and state. In this
  case the antecedents are slices of the numpy array representing the
  neighbourhood surrounding each cell; and the state the relevant cell. It's
  then just case of iterating over the array and assigning each cell to 
  an agent. We also set the first step of each agent to fire at 0:
  {% highlight python%}
  for i in range(1, initial_state.shape[0]-1):
        for j in range(1, initial_state.shape[1]-1):
            cell(0, 
                 initial_state[i-1: i+2, j-1: j+2],
                 initial_state[i:i+1, j:j+1],
                 {}){% endhighlight %}
- The model can then be run using the scheduler, and additionally we store a 
  copy of the state at each step ot track the history of the model
  {% highlight python%}
  while not scheduler.finished:
        history.append(initial_state.copy())
        scheduler.step(){% endhighlight %}

This example used numpy as a data-structure to store the state of the model,
but the API is very general, and as long as it can be passed by reference any
data structure can be used to track the state and pass the relevant parts of it 
to the update function.
 
The scheduling is also very general, in this case the model was designed to 
update all the agents at fixed steps, but you could just as well use
`datetime` as a variable, or have agents act at different intervals.
 
For more examples see [here](https://github.com/zombie-einstein/functional_abm/tree/master/examples) 
in the repo.

## Conclusion

For a short project I feel this turned out quite neatly. I think one of the
nice features that came out as a side effect is being able to make use of 
different backends to store the state of the model. An approach using classes
would require storing state as part of the class structure but with this API we
can make more efficient use of data structures to track state. For small models
this results in quite neat code.

This does come with some drawbacks though. 

- You need to pass everything required as part of the arguments to the update 
  function (where as part of an OOP pattern you would be able to look up 
  attributes on the class). For example, you might want to pass in a global 
  object like a random number generator (the API might benefit from having an 
  additional `context` argument to make this easier)
- Allowing nodes to alter other nodes can cause some conflicts. This seems
  like a necessary feature to have to allow for a broader range of models
  (agents should be able to effect their environments right?) but in a model
  where agents update on the same step, this can cause issues where an agents
  state is updated by itself and another node. Though this can be avoided
  with the appropriate design choices.
  
A python package to be able to use this API can be found on it's 
[github repo](https://github.com/zombie-einstein/functional_abm) along with 
usage examples. Please try it out, it'd be interesting to see how this API
holds up across more model implementations.
