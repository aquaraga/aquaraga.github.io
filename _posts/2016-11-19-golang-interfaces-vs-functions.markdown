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

Note that the function returned by `GetIncrementer()` closes over the variable `i`. For more details on higher-order functions and closures, you could refer to [this previous article][my-blog-on-closures]

### A brief introduction to interfaces ###

One of the other important features in Golang is interfaces. A `struct` is said to implement an `interface` if it has implementations for all the methods declared in that `interface`. The `struct` need not explicitly declare that it implements the `interface`.

{% highlight go %}
type Vehicle interface {
  Move()
  Honk()
}

type Car struct {
}

func (car *Car) Move() {
  fmt.Println("Car is moving")
}

func (car *Car) Honk() {
  fmt.Println("Car horn goes honk honk honk")
}
{% endhighlight %}

In the above example, Car *is a* Vehicle by virtue of implementing the two methods that the Vehicle interface declares. This is akin to [duck typing][duck-typing], because the Car doesn't explicitly declare itself to be implementing Vehicle.

### So how are these related? ###

An interesting case arises when you have one-method interfaces. Many interfaces in the standard library have only a single method - and it seems that the community has a preference for fine-grained interfaces than coarse-grained, perhaps because they are more re-usable. An example is the http `Handler` interface in the `net/http` package of the standard library.

{% highlight go %}
type Handler interface {
  ServeHTTP(ResponseWriter, *Request)
}
{% endhighlight %}

Implementations of handlers are generally wired to a http route, as is illustrated by the following code.

{% highlight go %}
http.Handle("/foo", fooHandler)
{% endhighlight %}

We can see that the `Handle` method expects two arguments - a route and a handler implementation. The API looks like:

{% highlight go %}
func Handle(pattern string, handler Handler)
{% endhighlight %}

A `FooHandler` that logs stuff and interacts with the database could be written as:

{% highlight go %}
type FooHandler struct {
  db *Database
  logger *Logger
}

func(fh *FooHandler) ServeHttp(rw ResponseWriter, req *Request) {
  //Do stuff with fh.logger
  //Do stuff with fh.db
}

fooHandler := &FooHandler{db: db, logger: logger}
{% endhighlight %}

The authors of the `Handle` method could have as well designed it as taking in a handler function instead of a handler implementation (afterall, the interface just had one method). Which is exactly why there is a `HandleFunc` method for API consumers who wish to pass in a function. The API looks like::

{% highlight go %}
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
{% endhighlight %}

But how would we define the function equivalent of the FooHandler, given that the handling depends on logger and database?
Enter closures:

{% highlight go %}
func FooHandlerFunc(db *Database, logger *Logger) func(ResponseWriter, *Request) {
  return func (rw ResponseWriter, req *Request) { //Inner handler function
    //Do stuff with logger
    //Do stuff with db
  }
}

fooHandlerFunc := FooHandlerFunc(db, logger)
http.HandleFunc("/foo", fooHandlerFunc)
{% endhighlight %}

Note that the inner handler function closes over the variables declared in the outer scope, namely `db` and `logger`.

### What's that again? ###

One could argue that APIs that take in a one-method interface could be re-written as taking a function type. More formally,

{% highlight go %}
type I interface {
  M(in) out
}

func API(I)  //The interface-way

func API(func M(in) out)  //The higher-order function way
{% endhighlight %}

### So which way do I choose? ###

I'm tempted to say: go the higher-order function way, but the standard library seems to favour the interface way. The io package defines one-method interfaces such as `Reader` and `Writer` and APIs that use them, but there are no counterpart higher-order function APIs. In some cases, such as the http handling APIs illustrated in this article, the standard library offers us both ways.

Some of the benefits of using a struct as an interface implementer are:

* The logic could be decomposed into several functions for complex implementations.
* A struct can implement different related interfaces: a solution that reinforces cohesion.
* A struct could expose other functions, such as a fluent API to build the struct (aka Builder pattern)

A higher-order function could be more elegant if the implementation is straight-forward.

As an API author, one could provide both mechanisms and let the API consumer decide the style of invocation.










[duck-typing]: http://stackoverflow.com/questions/4205130/what-is-duck-typing
[my-blog-on-closures]: http://aquaraga.github.io/functional-programming/javascript/2015/11/18/currying.html
