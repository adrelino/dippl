---
layout: chapter
title: Early, incremental evidence
description: Inserting and commuting factor statements to get the right incremental sequencing.
---

Many models that are important in applications have state spaces so large that, when the model is naively written, information only becomes available to guide inference at the very end of the computation. This makes it difficult for sequential exploration strategies (such as enumeration and particle filtering) to work. Two common examples are the hidden Markov model (HMM) and the probabilistic context free grammar (PCFG). We first introduce these models, then describe techniques to transform them into a form that makes sequential inference more efficient. Finally, we will consider a harder class of models with 'global' conditions.

## Unfolding data structures

### The HMM
 
Below, we assume that `transition` is a stochastic transition function from hidden to hidden states, `observeState` is an observation function from hidden to observeStated states, and `init` is an initial distribution.

~~~
var transition = function(s) {
  return s ? flip(0.7) : flip(0.3)
}

var observeState = function(s) {
  return s ? flip(0.9) : flip(0.1)
}

observeState(transition(true))
~~~

Here is a fairly standard version of the HMM:

~~~
///fold:
var transition = function(s) {
  return s ? flip(0.7) : flip(0.3)
}

var observeState = function(s) {
  return s ? flip(0.9) : flip(0.1)
}
///

var hmm = function(n) {
  var prev = (n==1) ? {states: [true], observations:[]} : hmm(n-1)
  var newState = transition(prev.states[prev.states.length-1])
  var newObs = observeState(newState)
  return {
    states: prev.states.concat([newState]),
    observations: prev.observations.concat([newObs])
  }
}

hmm(4)
~~~

We can condition on some observed states:

~~~
///fold:
var transition = function(s) {
  return s ? flip(0.7) : flip(0.3)
}

var observeState = function(s) {
  return s ? flip(0.9) : flip(0.1)
}

var hmm = function(n) {
  var prev = (n==1) ? {states: [true], observations:[]} : hmm(n-1)
  var newState = transition(prev.states[prev.states.length-1])
  var newObs = observeState(newState)
  return {
    states: prev.states.concat([newState]),
    observations: prev.observations.concat([newObs])
  }
}
///

//some true observations (the data we observe):
var trueObs = [false, false, false]

var model = function(){
  var r = hmm(3)
  factor(_.isEqual(r.observations, trueObs) ? 0 : -Infinity)
  return r.states
};

viz.table(Infer({ model }))
~~~

Notice that if we allow enumeration only a few executions, it will not find the correct state. It doesn't realize until 'the end' that the observations must match the `trueObs`. Hence, the hidden state is likely to have been `[false, false, false]`.

### The PCFG

The PCFG is very similar to the HMM, except it has an underlying tree structure instead of a linear one.

~~~
var pcfgTransition = function(symbol) {
  var rules = {'start': {rhs: [['NP', 'V', 'NP'], ['NP', 'V']], probs: [0.4, 0.6]},
               'NP': {rhs: [['A', 'NP'], ['N']], probs: [0.4, 0.6]} }
  return rules[symbol].rhs[ discrete(rules[symbol].probs) ]
}

var preTerminal = function(symbol) {
  return symbol=='N' | symbol=='V' | symbol=='A'
}

var terminal = function(symbol) {
  var rules = {'N': {words: ['John', 'soup'], probs: [0.6, 0.4]},
               'V': {words: ['loves', 'hates', 'runs'], probs: [0.3, 0.3, 0.4]},
               'A': {words: ['tall', 'salty'], probs: [0.6, 0.4]} }
  return rules[symbol].words[ discrete(rules[symbol].probs) ]
}


var pcfg = function(symbol) {
  preTerminal(symbol) ? [terminal(symbol)] : expand(pcfgTransition(symbol))
}

var expand = function(symbols) {
  if(symbols.length==0) {
    return []
  } else {
    var f = pcfg(symbols[0])
    return f.concat(expand(symbols.slice(1)))
  }
}

var model = function(){
  var y = pcfg("start")
  factor(_.isEqual(y.slice(0,2), ["tall", "John"]) ? 0 : -Infinity) // yield starts with "tall John"
  return y[2] ? y[2] : "" // distribution on next word?
}

viz.table(Infer({ model, method: 'enumerate', maxExecutions: 20}))
~~~

This program computes the probability distribution on the next word of a sentence that starts 'tall John....' It finds a few parses that start this way. However, this grammar was specially chosen to place the highest probability on such sentences. Try looking for completions of 'salty soup...' and you will be less happy.


## Decomposing and interleaving factors

To see how we can provide evidence earlier in the execution for models such as the above, first consider a simpler model:

~~~
var binomial = function(){
  var a = sample(Bernoulli({ p: 0.1 }))
  var b = sample(Bernoulli({ p: 0.9 }))
  var c = sample(Bernoulli({ p: 0.1 }))
  factor((a && b) ? 0 : -Infinity)
  return a + b + c
}

viz(Infer({ model: binomial }))
~~~

First of all, we can clearly move the factor up, to the point when it's first dependency is bound. In general, factor statements can be moved anywhere in the same control scope in which they started (i.e., they must be reached in the same program executions and not cross a marginalization boundary). In this case:

~~~
var binomial = function(){
  var a = sample(Bernoulli({ p: 0.1 }))
  var b = sample(Bernoulli({ p: 0.9 }))
  factor((a && b) ? 0 : -Infinity)  
  var c = sample(Bernoulli({ p: 0.1 }))
  return a + b + c
}

viz(Infer({ model: binomial }))
~~~

But we can do much better by noticing that this factor can be broken into an equivalent two factors, and again one can be moved up:

~~~
var binomial = function(){
  var a = sample(Bernoulli({ p: 0.1 }))
  factor(a ? 0 : -Infinity)
  var b = sample(Bernoulli({ p: 0.9 }))
  factor(b ? 0 : -Infinity)
  var c = sample(Bernoulli({ p: 0.1 }))
  return a + b + c
}

viz(Infer({ model: binomial, method: 'enumerate', maxExecutions: 4 }))
~~~

Notice that this version will find the best execution very early!


### Exposing the intermediate state for HMM and PCFG

In order to apply the above tricks, decomposing and moving up factors, to more complex models, it helps to put the model into a form that explicitly constructs the intermediate states.
This version of the HMM is equivalent to the earlier one, but recurses the other way, passing along the partial state sequences:

~~~
///fold:
var transition = function(s) {
  return s ? flip(0.7) : flip(0.3)
}

var observeState = function(s) {
  return s ? flip(0.9) : flip(0.1)
}
///

var hmmRecur = function(n, states, observations){
  var newState = transition(states[states.length-1]);
  var newObs = observeState(newState);
  var newStates = states.concat([newState]);
  var newObservations = observations.concat([newObs]);
  return (n==1) ? { states: newStates, observations: newObservations } : 
  hmmRecur(n-1, newStates, newObservations);
}

var hmm = function(n) {
  return hmmRecur(n,[true],[])
}

var trueObs = [false, false, false]

var model = function(){
  var r = hmm(3)
  factor(_.isEqual(r.observations, trueObs) ? 0 : -Infinity)
  return r.states
};

viz.table(Infer({ model }))
~~~

Similarly, the PCFG can be written as:

~~~
///fold:
var pcfgtransition = function(symbol) {
  var rules = {'start': {rhs: [['NP', 'V', 'NP'], ['NP', 'V']], probs: [0.4, 0.6]},
               'NP': {rhs: [['A', 'NP'], ['N']], probs: [0.4, 0.6]} }
  return rules[symbol].rhs[ discrete(rules[symbol].probs) ]
}

var preTerminal = function(symbol) {
  return symbol=='N' | symbol=='V' | symbol=='A'
}

var terminal = function(symbol) {
  var rules = {'N': {words: ['John', 'soup'], probs: [0.6, 0.4]},
               'V': {words: ['loves', 'hates', 'runs'], probs: [0.3, 0.3, 0.4]},
               'A': {words: ['tall', 'salty'], probs: [0.6, 0.4]} }
  return rules[symbol].words[ discrete(rules[symbol].probs) ]
}
///

var pcfg = function(symbol, yieldsofar) {
  return preTerminal(symbol) ? yieldsofar.concat([terminal(symbol)]) : expand(pcfgtransition(symbol), yieldsofar)
}

var expand = function(symbols, yieldsofar) {
  return symbols.length==0 ? yieldsofar : expand(symbols.slice(1), pcfg(symbols[0], yieldsofar))
}

var model = function(){
  var y = pcfg("start",[])
  factor(_.isEqual(y.slice(0,2), ["tall", "John"]) ?0:-Infinity) //yield starts with "tall John"
  return y[2]?y[2]:"" //distribution on next word?
}

viz.table(Infer({ model, method: 'enumerate', maxExecutions: 20}))
~~~



### Incrementalizing the HMM and PCFG

We can now decompose and move factors. In the HMM, we first observe that the factor `factor( arrayEq(r.observations, trueObs) ? 0 : -Infinity )` can be seen as `factor(r.observations[0]==trueObs[0] ? 0 : -Infinity); factor(r.observations[1]==trueObs[1] ? 0 : -Infinity); ...`. Then we observe that these factors can be moved 'up' into the recursion to give:

~~~
///fold:
var transition = function(s) {
  return s ? flip(0.7) : flip(0.3)
}

var observeState = function(s) {
  return s ? flip(0.9) : flip(0.1)
}

var trueObs = [false, false, false]
///

var hmmRecur = function(n, states, observations){
  var newState = transition(states[states.length-1])
  var newObs = observeState(newState)
  factor(newObs==trueObs[observations.length] ? 0 : -Infinity)
  var newStates = states.concat([newState])
  var newObservations = observations.concat([newObs])
  return (n==1) ? { states: newStates, observations: newObservations } : 
                  hmmRecur(n-1, newStates, newObservations)
}

var hmm = function(n) {
  return hmmRecur(n, [true], [])
}

var model = function(){
  var r = hmm(3)
  return r.states
}

viz.table(Infer({ model }))
~~~

Try varying the number of executions explored in this version and the original version. Start with 1 and increase... how do they differ?

Similarly for the PCFG:

~~~
///fold:
var pcfgTransition = function(symbol) {
  var rules = {'start': {rhs: [['NP', 'V', 'NP'], ['NP', 'V']], probs: [0.4, 0.6]},
               'NP': {rhs: [['A', 'NP'], ['N']], probs: [0.4, 0.6]} }
  return rules[symbol].rhs[ discrete(rules[symbol].probs) ]
}

var preTerminal = function(symbol) {
  return symbol=='N' | symbol=='V' | symbol=='A'
}

var terminal = function(symbol) {
  var rules = {'N': {words: ['John', 'soup'], probs: [0.6, 0.4]},
               'V': {words: ['loves', 'hates', 'runs'], probs: [0.3, 0.3, 0.4]},
               'A': {words: ['tall', 'salty'], probs: [0.6, 0.4]} }
  return rules[symbol].words[ discrete(rules[symbol].probs) ]
}
///

var pcfg = function(symbol, yieldsofar, trueyield) {
  if (preTerminal(symbol)){
    var t = terminal(symbol)
    if (yieldsofar.length < trueyield.length){
      factor(t==trueyield[yieldsofar.length] ?0:-Infinity)
    }
    return yieldsofar.concat([t])
  } else {
    return expand(pcfgTransition(symbol), yieldsofar, trueyield) }
}

var expand = function(symbols, yieldsofar, trueyield) {
  return symbols.length==0 ? yieldsofar : expand(symbols.slice(1), pcfg(symbols[0], yieldsofar, trueyield), trueyield)
}

var model = function(){
  var y = pcfg('start', [], ['tall', 'John'])
  return y[2]?y[2]:"" //distribution on next word?
}

viz.table(Infer({ model, method: 'enumerate', maxExecutions: 20}))
~~~

### sampleWithFactor

It is fairly common to end up with a factor that provides some evidence just after the sampled value it depends on. If we separate `sample` and `factor`, we will often try to explore sample paths that the factor will shortly tell us are very bad. To account for this, we introduce a compound operator `sampleWithFactor`, that takes a distribution, like `sample`, and also takes a function that is applied to the sampled value to compute a score for `factor`. By default, marginalization functions will simply treat `sampleWithFactor(dist,params,scoreFn)` as `var v = sample(dist,params); factor(scoreFn(v))`; however, some implementations will use this information more efficiently. The WebPPL `enumerate` inference method immediately adds the additional score to the score for the state as it is added to the queue -- this means that the additional score is included when prioritizing which states to explore next.

The binomial example becomes:

~~~
var binomial = function(){
  var a = sampleWithFactor(Bernoulli({ p: 0.1 }), function(v){return v?0:-Infinity})
  var b = sampleWithFactor(Bernoulli({ p: 0.9 }), function(v){return v?0:-Infinity})
  var c = sample(Bernoulli({ p: 0.1 }))
  return a + b + c
}

viz(Infer({ model: binomial, method: 'enumerate', maxExecutions: 2 }))
~~~

More usefully, for the HMM, this trick allows us to ensure that each `newObs` will be equal to the observed `trueObs`. We first marginalize out `observeState(..)` to get an immediate distribution from which to sample, and then use `sampleWithFactor(..)` to simultaneously sample and incorporate the factor:

~~~
///fold:
var transition = function(s) {
  return s ? flip(0.7) : flip(0.3)
}

var observeState = cache(function(s) {
  return Bernoulli({p: s ? .9 : .1})
})

var trueObs = [false, false, false]
///

var hmmRecur = function(n, states, observations){
  var newState = transition(states[states.length-1])
  var newObs = sampleWithFactor(
    observeState(newState),
    function(v){return v==trueObs[observations.length] ? 0 : -Infinity})
  var newStates = states.concat([newState])
  var newObservations = observations.concat([newObs])
  return ((n==1) ? 
          { states: newStates, observations: newObservations } :
          hmmRecur(n-1, newStates, newObservations));
}

var hmm = function(n) {
  return hmmRecur(n,[true],[])
}

var model = function(){
  var r = hmm(3)
  return r.states
}

viz.table(Infer({ model, method: 'enumerate', maxExecutions: 500 }))
~~~

(There is one more optimization for the HMM: We could achieve dynamic programming by inserting additional marginal operators at the boundary of `hmmRecur` and caching them.)



## Inserting canceling heuristic factors

What if we can't decompose the factor into separate pieces? For instance in:

~~~
var binomial = function(){
  var a = sample(Bernoulli({ p: 0.1 }))
  var b = sample(Bernoulli({ p: 0.9 }))
  var c = sample(Bernoulli({ p: 0.1 }))
  factor((a||b||c) ? 0 : -10)
  return a + b + c
}

viz(Infer({ model: binomial, method: 'enumerate', maxExecutions: 2}))
~~~

We can still insert 'heuristic' factors that will help the inference algorithm explore more effectively, as long as they cancel by the end. That is, `factor(s); factor(-s)` has no effect on the meaning of the model, and so is always allowed (even if the two factors are separated, as long as they aren't separated by a marginalization operator). For instance:

~~~
var binomial = function(){
  var a = sample(Bernoulli({ p: 0.1 }))
  factor(a ? 0 : -1)
  var b = sample(Bernoulli({ p: 0.9 }))
  factor(((a||b)?0:-1) - (a?0:-1))
  var c = sample(Bernoulli({ p: 0.1 }))
  factor(((a||b||c) ? 0:-10) - ((a||b)?0:-1))
  return a + b + c
}

viz(Infer({ model: binomial, method: 'enumerate', maxExecutions: 2 }))
~~~

This will work pretty much any time you have 'guesses' about what the final factor will be, while you are executing your program. Especially if these guesses improve incrementally and steadily. For examples of this technique, see the [incremental semantic parsing example](semanticparsing.html#incremental-world-building) and the [vision example](vision.html).

There is no reason not to *learn* heuristic factors that help guide search, as long as they cancel by the end they won't compromise the correctness of the computed distribution (in the limit). While it wouldn't be worth the expense to learn heuristic factors for a single marginalization, it may be very useful to do so across multiple related marginal distributions -- this is an example of *amortized* or *meta-* inference. (Note this is a topic of ongoing research by the authors....)

Next chapter: [Particle Filtering](/chapters/05-particlefilter.html)
