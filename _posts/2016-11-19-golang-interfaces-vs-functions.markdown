---
layout: post
title:  "Higher-order functions vs interfaces in golang"
date:   2016-11-19 10:49:27 +0530
categories: functional-programming golang
---

Golang is a "minimalist" language in the sense that there are only a handful of features and programming language constructs that it offers.
It is probably not what a functional-programming enthusiast would prefer to code in; nevertheless, its elegance lies in its simplicity. The language never gets in your way, unless you fight it.

### Thank God, we have higher-order functions ###

From a functional-programming point of view, Golang supports higher-order functions. Functions are first-class citizens of the Golang world, and are treated with respect.
One could assign pass functions around and return functions from functions.

{% highlight go %}
func GetIncrementer() func() int { //A function that returns (a function that returns an integer)
  i := -1
  return func() int {
    i = i + 1
    return i
  }
}

func main() {
  increment := GetIncrementer()
  fmt.Println(increment()) //prints 0
  fmt.Println(increment()) //prints 1
}

{% endhighlight %}

Note that the function returned by `GetIncrementer()` closes over the variable `i`. For more details on higher-order functions and closures, you could refer to [this article][my-blog-on-closures]

### A brief introduction to interfaces ###

One of the other important features in Golang is interfaces. A struct is said to implement an interface if it has implementations for all the methods declared in that interface. The struct need not explicitly declare that it implements the interface.

{% highlight go %}
type Vehicle interface {
  MoveForward()
  MoveBackward()
  Honk()
}

type Car struct {
}

func (car *Car) MoveForward() {
  fmt.Println("Car is moving forward")
}

func (car *Car) MoveBackward() {
  fmt.Println("Car is moving backward")
}

func (car *Car) Honk() {
  fmt.Println("Car horn goes honk honk honk")
}
{% endhighlight %}

In the above example, Car *is a* Vehicle by virtue of implementing the three methods that the Vehicle interface declares. This is akin to [duck typing][duck-typing], because the Car doesn't explicitly declare itself to be implementing Vehicle.



[duck-typing]: http://stackoverflow.com/questions/4205130/what-is-duck-typing
[my-blog-on-closures]: http://aquaraga.github.io/functional-programming/javascript/2015/11/18/currying.html
