CloCoP
======

CloCoP is a Clojure wrapper for JaCoP, which is a Java constraint programming engine. JaCoP stands for <b>JA</b>va <b>CO</b>nstraint <b>P</b>rogramming. Can you guess what CloCoP stands for?

###Usage

Add the following to your dependencies:

    [clocop "0.2.0"]

###Sample code

If you're curious, here's some sample code based on the API I have so far.

    (use 'clocop.core
         'clocop.constraints)
    
    ; Variable X is between 1 and 2. Variable Y is between 3 and 4.
    ; We know that X + Y = 6. What are X and Y?
    (with-store (store)           ; initialize the variable store
      (let [x (int-var "x" 1 2)
            y (int-var "y" 3 4)]  ; initialize the variables
        (constrain! ($= ($+ x y) 6))   ; specify x + y = 6
        (solve!)))                ; searches for a solution

    => {"x" 2, "y" 4}

###But what about core.logic?

I'm aware of core.logic, but there are a few ways in which, in my opinion, JaCoP is better than MiniKanren:

+ JaCoP is more "plug-in-able," with an extensive set of customizations to the way that the search operates. There are interfaces for different components of the search, and each have several implementations.
+ I found that with core.logic, I was somewhat limited by the set of available constraints. JaCoP has many different global and FD constraints that seem to more suit my needs for solving challenging problems.
+ [As the core.logic people say,](https://github.com/clojure/core.logic/wiki/External-solvers) JaCoP is anywhere from 10X-100X faster than core.logic at solving Finite Domain problems.

Constraint Programming
======

Here is a very brief guide to the key components in CP, as well as the implementation of each component in CloCoP.

###The Store

The store has one job and one job only: keep track of all the variables.
Constraints, searches, and variables themselves do the hard work of actually solving the problems.

One way to think about it is that it's a box for all the variables, with several constraints attached to that box.

In CloCoP, a store is created with <code>(store)</code>. To create variables and constrain constraints, you must wrap those function calls in the following macro:

    (with-store <my-store>
      (...))

so that the variables and constraints know which store you're talking about.
###Variables

Variables can only have integer values. Every variable is assigned a "domain," i.e. a finite set of integers it could possibly be.
In JaCoP, initial domains must be assigned to variables at the time of creation.

In CloCoP, a variable is created with <code>(int-var "name" min max)</code>.

###Constraints

A constraint can conceptually be as simple as "X = 3", or as complex as "X, Y, and Z are all different".

It has two jobs:
- check if it is still feasible (e.g. in the X = 3 example, it will check if 3 is in the domain of X)
- "prune" the domains of the variables in question (e.g. in the X = 3 example, it will remove any values in the domain of X that is not 3).

In CloCoP, all of the constraints are in the clocop.constraints namespace. By convention, all constraints start with a "$", i.e. <code>$=</code> for "=". This is because there would be a lot of overlap between the constraint names and the clojure.core function names.

You can find a complete guide to clocop.constraints at the [bottom](https://github.com/aengelberg/clocop#arithmetic) of the page.

If you want to apply a constraint to the store, use <code>(constrain! ...)</code>.

###The Search

The three steps to CP search:
Step 1: repeatedly ask the constraints to prune the domains until no further pruning can be done.
Step 2: pick a variable V (and a possible value X for V), and branch into two new child searches:
- one of which with X assigned to V (i.e. V has a domain with one item),
- and the other with X removed from the domain of V.
Step 3: repeat this process on the two child nodes (the assignment node first).

To solve your store, i.e. find a satisfying assignment for all of the variables, simply call <code>(solve!)</code>.

Here is a complete list of the optional keyword arguments to <code>solve!</code>:

<code>:solutions</code> will either return one (<code>:one</code>) or all (<code>:all</code>) of the solutions.

<code>:log?</code>, when set to true, will have the search print out a log to the Console about the search.

<code>:minimize</code> takes a variable that the search will attempt to minimize the value of.

<code>:pick-var</code> will pick a variable (as described in Step 2). Possible choices:
- <code>:smallest-domain</code> (default): var with smallest domain
- <code>:most-constrained-static</code>: var with most constraints assigned to it
- <code>:most-constrained-dynamic</code>: var with most pending constraints assigned to it
- <code>:smallest-min</code>: var with smallest value in its domain
- <code>:largest-domain</code>: var with largest domain size
- <code>:largest-min</code>: var with largest minimum in its domain
- <code>:smallest-max</code>: var with smallest maximum in its domain
- <code>:max-regret</code>: var with biggest difference between min and max
- <code>(list var var ...)</code>: will choose those variables in order (skipping over already-assigned vars).
Note that this final option will induce a side effect: only the given variables will appear in the solution assignment(s).

<code>:pick-val</code> will pick a value (as described in Step 2) for the chosen variable. Possible choices:
- <code>:min</code> (default): minimum value
- <code>:max</code>: maximum value
- <code>:middle</code>: selects middle value (and later chooses the left and right values).
- <code>:random</code>: random
- <code>[:random N]</code>: random with seed
- <code>:simple-random</code>: faster than <code>:random</code> but lower quality randomness 

CloCoP Constraints
======

###A note about "arithmetic piping"

Suppose you want to add a constraint "X + Y = Z." In JaCoP, you'd write this:

    Constraint c = new XplusYeqZ(X, Y, Z);
    store.impose(c);

That's fine if you're using very small-scale constraints, but it gets complicated when you want to constrain something like "A + (B * C) = D."

    IntVar BtimesC = new IntVar(...);
    Constraint c1 = new XmulYeqZ(B, C, BtimesC);
    store.impose(c1);
    Constraint c2 = new XplusYeqZ(A, BtimesC, D);
    store.impose(c2);
    
It becomes tiring to have to work from the bottom up when designing the vars and constraints. Here's how you would write the same constraint in CloCoP:

    (constrain! ($= ($+ A ($* B C)) D))
    
which basically expands to the following:

    (constrain! (XmulYeqZ. B C tempVar1))
    (constrain! (XplusYeqZ. A tempVar1 tempVar2))
     
    (constrain! (XeqY. tempVar2 D))
    
I call this concept "piping," which lets you create your constraints top-down without the need to create temporary variables.

Now, on to the actual constraints...

### Arithmetic

Note that these aren't actual "constraints" but just piping functions, which take variables as inputs and return a new variable.

- <code>($+ x y ...)</code> - sum
- <code>($- x ...)</code> - negation or subtraction
- <code>($* x y)</code> - multiplication
- <code>($pow x y)</code> - exponent
- <code>($min ...)</code> - min
- <code>($max ...)</code> - max
- <code>($abs x)</code> - absolute value
- <code>($weighted-sum [x y z ...] [a b c ...])</code> - given some vars and some constant ints, returns <code>ax + by + cz + ...</code>

### Equality

Takes variables and returns a constraint.

- <code>($= x y)</code> - equals
- <code>($< x y)</code> - less than
- <code>($<= x y)</code> - less than or equal
- <code>($> x y)</code> - greater than
- <code>($>= x y)</code> - greater than or equal
- <code>($!= x y)</code> - not equal

### Logic

Takes constraints and returns another constraint.
Note: logic constraints can only take equality constraints, or other logic constraints. i.e. NOT global constraints.

- <code>($and & clauses)</code> - "and" statement; all of the given statements are true
- <code>($or & clauses)</code> - "or" statement; at least one of the given statements are true
- <code>($not P)</code> - "not / ~P" statement; P is not true.
- <code>($if P Q R)</code> - "if/then/else" statement; if P is true, Q is true, otherwise R is true (R is optional)
- <code>($cond ...)</code> - behaves like "cond" but made out of <code>$if</code> statements; not a macro
- <code>($iff P Q)</code> - "iff/<=>" statement; P is true if and only if Q is true

### Global

Miscellaneous constraints/piping functions which are useful in certain situations.

Constraints:
- <code>($all-different & vars)</code> - "all different" statement; none of the vars are equal to any other var.

Piping functions:
- <code>($reify c)</code> - given a constraint, returns a variable that will be 1 if the constraint is true and 0 if the constraint is false. It can only be passed logic or equality constraints.
- <code>($nth L i)</code> - given a list of vars (or a list of constants) and a var i, returns another var that will equal <code>L[i]</code>.
- <code>($occurrences L i)</code> - given a list of vars, and a constant i, returns another var that will equal the amount of times i appears in L.
