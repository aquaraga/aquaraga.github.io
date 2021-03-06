---
layout: post
title:  "Reactive programming tips"
date:   2019-08-03 10:10:14 +0530
categories: reactor reactive
---

It is possible to wade through the complexities of reactive data types and APIs and embrace the full power and elegance of functional reactive programming by appreciating a few concepts and watching out for pitfalls. In this article, we'll explore a few tips to keep in mind while programming with a reactive framework. We'll choose the [Reactor][reactor] framework to illustrate the concepts (Similar ones apply to the other popular framework out there: [Rx][rx]).

Although many examples use a rather hard-coded publisher such as `Flux.just(1, 2, 3, 4)`, it is important to note that many real-world publishers represent I/O operations such as outbound HTTP calls or database calls. To depict a true I/O, some examples introduce artificial delays - as in `Mono.just(1).delayElement(Duration.ofSeconds(5))`

### Tip 1 - Make sure there are no orphaned publishers ###

Reactive streams are lazy, i.e. publishers donot evaluate unless there is atleast one subscription to them.

{% highlight java %}
Flux<Integer> theFlux = Flux.just(1, 2, 3, 4)
        .map(x -> x * 2);

System.out.println(theFlux); //prints 'FluxMapFuseable', i.e. the flux is not evaluated
System.out.println(theFlux.collectList().block()); //prints '[2, 4, 6, 8]'
{% endhighlight %}

In the above example, a `.block()` (after a `collectList()`) can be used to evaluate the transformed publisher, although this is a bad idea if this code is executed in the context of a non-blocking server such as Netty. Functions in real-world applications using reactive streams keep returning them to their caller, eventually bubbling the publishers up to the end consumer. For eg; a REST API built using Springboot 2.x most certainly return `Flux` or `Mono` reactive types from the Controller methods, which in turn get consumed by HTTP response streams.

A better way of consuming a publisher is via the `subscribe` method. However, to keep things as simple as possible (i.e. to run the examples on a main method), we'll stick to printing with blocking calls. Let us define convenience functions to consume and print a publisher:

{% highlight java %}
void print(Flux<?> flux) {
    System.out.println(flux.collectList().block());
}

void print(Mono<?> mono) {
    System.out.println(mono.block());
}

print(Flux.just(1, 2, 3, 4)); //prints '[1, 2, 3, 4]'
{% endhighlight %}

#### `map` vs `flatMap` ####

A very subtle albeit important illustration involves transformation functions that return publishers.

{% highlight java %}
print(Flux.just(1, 2, 3, 4)
          .map(x -> Mono.just(x * 2)));
              //prints '[MonoJust, MonoJust, MonoJust, MonoJust]'
{% endhighlight %}

The inner `Mono`s were silently discarded since `map` is not a good enough consumer. For a 'deep consumption', use `flatMap`:

{% highlight java %}
print(Flux.just(1, 2, 3, 4)
          .flatMap(x -> Mono.just(x * 2)));
              //prints '[2, 4, 6, 8]'
{% endhighlight %}

In chained method invocations on reactive streams, an accidental `map` instead of `flatMap` can introduce bugs that are hard to troubleshoot.

### Tip 2 - Donot over-subscribe a publisher ###

Multiple subscriptions to a publisher, especially ones representing I/O, can significantly degrade the performance of your application. In the below example, the same I/O operation gets executed as many times as there are subscriptions, which is 2 in this case.

{% highlight java %}
Flux<Integer> slowIO = Flux.just(1, 2, 3, 4).delayElements(Duration.ofSeconds(1));

Flux<?> first = slowIO.map(x -> f1(x));
//... After several lines of code ...
Flux<?> second = slowIO.map(x -> f2(x));
//...
Flux.zip(first, second)
  .subscribe(y -> subscriptionFn(y));
{% endhighlight %}

The antidote to this problem is to perform all the required transformations (`f1` and `f2`) within the same `map` block.

### Tip 3 - Parallelize generously ###

If there are two or more publishers that can execute in parallel (for eg; the publishers talk to different downstream systems), leverage Reactive APIs such as `zip`, `zipWith` and `mergeWith` so that consumption doesn't happen serially and the total time doesn't keep adding up.

{% highlight java %}
Mono<Integer> slowIO = Mono.just(1).delayElement(Duration.ofSeconds(5));
Mono<Integer> anotherSlowIO = Mono.just(-1).delayElement(Duration.ofSeconds(4));

print(slowIO.then(anotherSlowIO)); //Takes ~9 seconds
print(slowIO.zipWith(anotherSlowIO)); //Takes ~5 seconds, i.e. only as long as the slowest publisher

{% endhighlight %}

#### The cardinal sin of `zip` ####

`zip` functions are not only useful to execute operations in parallel, but they also help to assemble results of the operations into an aggregate object.

{% highlight java %}
ProductInformation product = Mono.zip(fetchProductDetails(productId),
    fetchSubstitutions(productId), fetchOffersFor(productId), fetchImagesFor(productId))
    .map(zipped -> new ProductInformation(x.getT1(), x.getT2(), x.getT3(), x.getT4()));
{% endhighlight %}

Beware of zipped streams: they wait only for the shortest length stream to finish. It is therefore typical to zip `Mono`s since their cardinality can atmost be 1. Even here there is a catch: if one of the participating streams return a `Mono.empty()`, the entire operation short-circuits - which means some of the `Mono`s may not be even evaluated.

A `flatMap` may also be used to fire off calls in parallel, but streams resulting from each operation are merged - i.e. the shortest operation stream gets emitted first to the output stream. Be prepared to re-arrange the elements in the stream if the use-case demands that ordering be preserved.

{% highlight java %}
Flux<Mono<Integer>> slowlyEmittingStreams = Flux.just(
    Mono.just(1).delayElement(Duration.ofSeconds(3)),
    Mono.just(2).delayElement(Duration.ofSeconds(2)),
    Mono.just(3).delayElement(Duration.ofSeconds(1)));
print(slowlyEmittingStreams.flatMap(x -> x)); //prints '[3, 2, 1]'
{% endhighlight %}

### Tip 4 - Beware of nested reactive data types ###

While on a chained method invocation spree, it is easy to get caught in a mire of nested reactive and/or stream data types such as `Flux<Mono>`, `Mono<Mono>` and `Flux<Mono<List>`. Such types are recipes for over-subscribed publishers, and you must quickly quell the situation by using flattening operators.

#### Case 1: Flatten a publisher of lists (i.e. `Flux<List<T>>` to `Flux<T>`) ####

Solution: Use `flatMapIterable`
{% highlight java %}
Flux<List<Integer>> flux = Flux.just(Arrays.asList(1, 2), Arrays.asList(3));
Flux<Integer> integerFlux = flux.flatMapIterable(x -> x);
print(integerFlux); //prints '[1, 2, 3]'
{% endhighlight %}

You can also apply transformations if required. In the above example, if you want to double the numbers, specify `(x -> x * 2)`

#### Case 2: Flatten a list of publishers (i.e. `List<Mono<T>>` to `Flux<T>`) ####

Solution: Use `flatMap` after a `fromIterable`

{% highlight java %}
List<Mono<Integer>> monos = Arrays.asList(Mono.just(1), Mono.just(2), Mono.just(3));
Flux<Integer> anotherIntergerFlux = Flux.fromIterable(monos).flatMap(x -> x);
print(anotherIntergerFlux); //prints '[1, 2, 3]'
{% endhighlight %}

This is also applicable for converting a `List<Flux<T>>` to `Flux<T>`. You can even apply transformations as shown in Case 1.

#### Case 3: Flatten nested publishers (for eg; `Flux<Mono<T>>` to `Flux<T>`) ####

Solution: Use `flatMap`

{% highlight java %}
Flux<Mono<Integer>> doublePublisher = Flux.just(Mono.just(1), Mono.just(2), Mono.just(3));
Flux<Integer> singlePublisher = doublePublisher.flatMap(x -> x);
print(singlePublisher); //prints '[1, 2, 3]'
{% endhighlight %}

You can flatten other publisher combinations too, for eg; `Mono<Flux<T>>`, double monos (`Mono<Mono<T>`) and double fluxes (`Flux<Flux<T>`). Here too, you can even apply transformations as shown in Case 1.

### Tip 5 - Convert blocking calls to reactive ones ###

On an enterprise-y project, you occassionally bump into that annoying database driver or that little library method which does not support asynchronous programming and streams. Fortunately, we can convert synchronous calls to reactive by wrapping them with a `Mono.fromCallable`. Make sure that you dedicate another threadpool for this executing this operation.

{% highlight java %}
List<Order> orders = repository.findByUserId(userId); //blocking

Mono<List<Order>> ordersReactive = Mono.fromCallable(() -> repository.findByUserId(userId))
    .subscribeOn(Schedulers.elastic()); //reactive
Flux<Order> orderFlux = ordersReactive.flatMapIterable(x -> x); //Using Tip 1 of Case 4
{% endhighlight %}

[reactor]: https://projectreactor.io/
[rx]: http://reactivex.io/