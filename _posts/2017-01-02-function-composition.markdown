---
layout: post
title:  "Composing functions"
date:   2017-01-02 08:30:27 +0530
categories: functional-programming clojure javascript
math: true
---



No function is an island (okay; some might be, but still..). Functions work with one another to achieve a goal. Let's take a simple example of finding the third element of a list:

{% highlight clojure %}
(defn third [x] (first (rest (rest x))))

(third '(:orange :guava :apple :pear)) 
;;=> :apple (kinda my third favourite fruit too)
{% endhighlight %}

The deep nesting of function calls might come across as an eye-sore. Enter function composition, a teeny-tiny treasure hidden in your high-school algebra textbook:

$$ f(g(h(x))) = (f \circ g \circ h)x $$

### The `comp` function ###

Clojure standard library provides a `comp` function, that takes in a bunch of functions and returns a _composition_ of those functions. When applied to a set of arguments, the inner-most function consumes the arguments first and passes on its results to the next function in the chain: eventually the outer-most function executes, yieliding the final result.

As an example, `(comp f g h)` would return a composition function. When a set of arguments, say `x y` are applied to the composition function:
* `h` evaluates on `x y` first
* The result obtained from above is passed to `g`
* The result obtained from above is passed to `f`
* The result obtained from above becomes the output of the composition function

Rewriting our `third` function this way gives us a function that has fewer brackets, and arguably a bit more readable:

{% highlight clojure %}
(defn third [x] ((comp first rest rest) x))
{% endhighlight %}

### Eta-reduction ###

We can go one step further and use [eta-reduction][eta-reduction]. To exemplify, instead of saying $$ k(x) = (f \circ g \circ h)x $$, we could just say $$ k = f \circ g \circ h $$, effectively 'cancelling' out `x` from the left-side and right-side. This leads us to an even better way of writing `third`:

{% highlight clojure %}
(def third (comp first rest rest))
{% endhighlight %}

It may be interesting to note that we used `def` here, and not `defn`, as there are no formal arguments to declare up-front. This is equivalent of proudly proclaiming "`third` is a composition of `first`, `rest` and `rest` functions; in that order".

### Example: sigmoid function ###

The sigmoid function is widely used in Machine learning classification problems, where we want to map a real number `z` to the interval `(0, 1)`. The function definition is:

$$ g(z) = {1 \over {1 + e^{-z}}} $$

The plot looks like:

![Sigmoid screenshot]({{ site.url }}/assets/images/Sigmoid.png){: .center-image }

To break-down the problem, we could conceptually write the sigmoid function as:

$$ sigmoid(z) = reciprocal(oneplus(epow(minus(z)))) $$

Leveraging function composition and eta-reduction,

$$ sigmoid = reciprocal \circ oneplus \circ epow \circ minus $$

### So how does it look like in Clojure? ###

Two functions are out-of-the-box: `minus` is `-` and `oneplus` is `inc`.
There is no standard `reciprocal` function: however it is nothing but a [curried]({% post_url 2015-11-18-currying %}) divide function where the numerator is 1. In other words, it is `(partial / 1)`. Ditto for the epow function, which is a curried exponent function where the base is the constant `2.7182`.

Let's put all this together:

{% highlight clojure %}
(ns comp-fun.core
  (:require [clojure.math.numeric-tower :as math]))

(def reciprocal (partial / 1))

(def epow (partial math/expt 2.7182))

(def sigmoid (comp reciprocal inc epow -))

(sigmoid 100)
;;=> 1
(sigmoid 1)
;;=> 0.7310526598891937
(sigmoid 0)
;;=> 0.5
(sigmoid -10000)
;;=> 0

{% endhighlight %}

### Can I compose everything in the world? ###

Of course not. Functions need to follow a chain (where output of one feeds as input to another) so that we can compose them. Not all functions follow this pattern. Take for instance the quadratic solution function having multiple parameters (`a`, `b` and `c`), which are referenced all over the place:

$$ \begin{equation*} x = \frac{-b \pm \sqrt{b^{2} -4ac}}{2a} \end{equation*} $$

Nevertheless, one should watch out for sub-parts of functions where composition can be utilized. Take for example the Pythagorean formula to compute the length of the hypotenuse:

$$ \begin{equation*} c = \sqrt{a^2 + b^2} \end{equation*} $$

In Clojure, we could write the function as:

{% highlight clojure %}
(ns comp-fun.core
  (:require [clojure.math.numeric-tower :as math]))

(defn hypotenuse [a b] ((comp math/sqrt +)(* a a) (* b b)))

{% endhighlight %}

In the process of finding the _root of sum of squares_, we could still abstract out _root of sum_ as a composition i.e. `(comp math/sqrt +)`. 


[eta-reduction]: https://wiki.haskell.org/Eta_conversion
