---
layout: post
title:  "List comprehensions"
date:   2018-03-16 14:10:12 +0530
categories: functional haskell
math: true
---

### A throwback into middle school ###

Here's a programming pearl based straight out of your grade 6 mathematics text book. Getting any déjà vu from the notation given below?

$$ \{ 2x | x\colon x \in Z, 0 \le x < 10 \} $$

This is nothing but the [set-builder notation][set-builder]: this particular example represents the set of first 10 even numbers (considering 0 to be an even number). $$ Z $$ represents the set of integers.

### Set builders in code ###

List comprehensions are rip-offs of the set-builder notation. In Clojure, the `for` macro is used to create a list comprehension. The first 10 even numbers can be generated using:

{% highlight clojure %}
(for [x (range 10)] (* x 2))
;; -> (0 2 4 6 8 10 12 14 16 18)
{% endhighlight %}

Without list comprehensions, the same can be written using a `map`:

{% highlight clojure %}
(map #(* % 2) (range 10))
;; -> (0 2 4 6 8 10 12 14 16 18)
{% endhighlight %}

Although it might seem that the `for` style is more declarative than the code using `map`, it is a matter of taste as to which style is preferable. In some cases though, one style will result in more readable code:

{% highlight clojure %}
(mapcat (fn [x] (map #(* % x) [4 5 6])) [1 2 3])
;; -> (4 5 6 8 10 12 12 15 18)
(for [x [1 2 3] y [4 5 6]] (* x y))
;; -> (4 5 6 8 10 12 12 15 18)
{% endhighlight %}

Clearly, the list comprehension wins here.

### Yet another shining use of Haskell ###

However, no programming language depicts list comprehensions as beautifully as Haskell does. 
{% highlight haskell %}
[x*y | x <- [1,2,3], y <- [4,5,6]
-- [4,5,6,8,10,12,12,15,18]
{% endhighlight %}

The beauty comes across since Haskell stays true to the set-builder notation as close as possible, and has no surrounding fluff like keywords to express constraints. Both Python and Clojure use `for` to represent list comprehensions, and they use `if` and `when` respectively to express constraints. Haskell does away with all ceremony.

Let's illustrate the same with the code to generate [Pythagorean triplets][pyth-triple] $$\le$$ 10:

{% highlight clojure %}
(defn square [x] (* x x))
(for [x (range 11) y (range x) z (range y)  
  :when (= (square x) (+ (square y) (square z)))] [x y z])
;; -> ([5 4 3] [10 8 6])
{% endhighlight %}

Okay, Clojure fans might cry foul because there isn't an out-of-the-box power function - but did anyone notice the `for` and the `when`? In Haskell, the code would look like:

{% highlight haskell %}
[(x, y, z) | x <- [1..10], y <- [1..x], z<- [1..y], x^2 == y^2 + z^2]
-- [(5,4,3),(10,8,6)]
{% endhighlight %}

### Epilogue: A Project Euler problem ###

Finally, let's whip up some code to solve the [problem on Pythagorean triplets][project-euler] on Project Euler, using list comprehensions:

{% highlight haskell %}
[x*y*z | x <- [1..1000], y <- [1..x], 
  let z = 1000 - x - y, y > z, z> 0, x^2 == y^2 + z^2]
-- [31875000]
{% endhighlight %}


[set-builder]: https://en.wikipedia.org/wiki/Set-builder_notation
[pyth-triple]: https://en.wikipedia.org/wiki/Pythagorean_triple
[project-euler]: https://projecteuler.net/problem=9