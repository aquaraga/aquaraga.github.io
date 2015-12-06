---
layout: post
title:  "Counting with Clojure"
date:   2015-12-06 08:32:47 +0530
categories: functional-programming clojure
---

I've been reading Carin Meier's [Living Clojure][living-clojure] and cannot recommend it highly enough for functional programming enthusiasts!
The book is divided into two parts - a guided tour of the most important Clojure concepts, and a weekly training plan.
After a quick read of the guided tour, I embarked on the training plan and have completed Week 1.

Many of the initial set of exercises are put up on [4Clojure][4-clojure], a fantastic website that hosts code exercises in a _koan-style_ (i.e. 'fill in the blanks' style). One of the problems deals with writing a function to count the total number of elements in a sequence. Ofcourse, one could use `count` - but the point here is to make use of functional programming concepts to achieve what the `count` function would have accomplished.

In short, we need to fill in the blanks - and the expression should evaluate to `true`:

{% highlight clojure %}
(= (__ '(1 2 3 3 1)) 5) 
{% endhighlight %}

### Idea 0: The `count` function

Why not use the super-lame, standard `count` function as a starter?

{% highlight clojure %}
(= (count '(1 2 3 3 1)) 5) 
;; => true
{% endhighlight %}



### Idea 1: One, two, three ...

Map the input sequence into a (1 2 3 ...). The last element would give the number of elements in the original sequence.
We could use the fact that `map` can be used with multiple collections, and terminates when the shortest collection ends.
In other words, `map f c1 c2` applies function `f` on the corresponding elements of `c1` and `c2`. The function's output is accumulated into a lazy sequence. For example, `map + [1 2 3] [4 5 6]` returns `[5 7 9]`

Going to back to our problem, we first need to transform the input sequence to (1 2 3 ...):

{% highlight clojure %}
(map (fn [i _] (inc i)) (range) '(1 2 3 3 1)) 
;; => (1 2 3 4 5)
{% endhighlight %}

Now, we need to get the last element of the above sequence:

{% highlight clojure %}
(last (map (fn [i _] (inc i)) (range) '(1 2 3 3 1)))
;; => 5
;; Or even better, I can use (comp last map) instead of (last (map))
((comp last map) (fn [i _] (inc i)) (range) '(1 2 3 3 1))
;; => 5
{% endhighlight %}

Finally, it looks like:

{% highlight clojure %}
(= ((comp last map) (fn[i _] (inc i)) (range) '(1 2 3 3 1)) 5) 
;; => true
{% endhighlight %}

### Idea 2: One, one, one ...

Map the input sequence into (1 1 1 ...) and sum it up.
Transforming the sequence into (1 1 1 ...) is easy. We just need to use `repeat 1` instead of `range` as one of the input collections to our map:

{% highlight clojure %}
(map (fn [one _] one) (repeat 1) '(1 2 3 3 1)) 
;; => (1 1 1 1 1)
{% endhighlight %}

To sum up the resultant sequence, we can use the `reduce` function. The `reduce` function takes two arguments: the ongoing result (or the __accumulator__) and the element it is processing currently. For example `reduce + [1 2 3 4]` returns `10`. Putting it together, we have:

{% highlight clojure %}
(reduce + (map (fn [one _] one) (repeat 1) '(1 2 3 3 1)))
;; => 5
{% endhighlight %}

Going back to our 'fill-in-the-blank answer', we cannot just replace the blanks with `(reduce + (map (fn [one _] one) (repeat 1)` - because we want to map before we even attempt to reduce. The solution is to create an anonymous function or a lambda. Let's illustrate that with examples:

{% highlight clojure %}
;; Lambdas take the form #(..). If there is only one argument to a lambda, that can be referred as %
(#(inc %) 1)
;; => 2
;; If there are multiple arguments, they can be referred as %1, %2 and so on
(#(+ %1 %2) 25 15)
;; => 40
{% endhighlight %}

Armed with this information, we can come up with the final 'fill-in-the-blank answer':

{% highlight clojure %}
(= (#(reduce + (map (fn[one _] one) (repeat 1) %)) '(1 2 3 3 1)) 5)
;; => true
{% endhighlight %}


### Idea 3: Aesthetic improvement on Idea 2

Instead of using the `map f c1 c2` form, we can use the more fundamental `map f c` form.
The idea is to constantly return 1 for every element of the input sequence. Luckily, we have the `constantly` built-in function.

{% highlight clojure %}
(map (constantly 1) '(1 2 3 3 1)) 
;; => (1 1 1 1 1)
{% endhighlight %}

The koan answer would now look like:

{% highlight clojure %}
(= (#(reduce + (map (constantly 1) %)) '(1 2 3 3 1)) 5)
;; => true
{% endhighlight %}

### Idea 4: Just count

Why not just use `reduce`, and keep incrementing the accumulator? Arguably, this is the simplest among all the ideas presented in this article.
So simple that I'll jump into the koan answer:

{% highlight clojure %}
(= (reduce (fn[c _] (inc c)) 0 '(1 2 3 3 1)) 5)
;; => true
{% endhighlight %}

That concludes the finger-licking experience of counting with Clojure. There might be a gazillion better ways to count in Clojure, but the bottom-line is that Clojure rocks!

[paneer-butter-masala]: http://www.vegrecipesofindia.com/paneer-butter-masala/
[living-clojure]: http://www.amazon.in/Living-Clojure-Carin-Meier/dp/1491909048
[4-clojure]: https://www.4clojure.com