---
layout: post
title:  "Fluent retries"
date:   2015-11-09 10:49:27 +0530
categories: fluent csharp
---

Not long ago, I was working with an unreliable external service which our application was depending upon.
By unreliable, I mean that the service wouldn't necessarily return a successful response in the first try. After discussions with the service provider, we felt that this necessitated a retry mechanism: call the service until it returns a successful response, or error out after N - say three - retries.
Also, a particular response code indicated that retries may not have any effect.

A knee-jerk reaction to write the code was to use a `while` loop:


{% highlight c# %}
var trials = 0;
var response = Responses.INVALID;
while (trials < 3) {
  response = MakeCallToService(request);
  if (RetryMayNotHaveAnyEffect(response)) {
    break;
  }
  trials++;
}
{% endhighlight %}

_What if_ you wanted to decouple the act of retrying from what gets tried?<br/>
_What if_ you wanted to test the retry logic?<br/>
_What if_ another place in the code warranted a retry mechanism?<br/>
_Doesn't_ the `while` and the counter increment hurt your eyes?<br/>
_Wouldn't_ it be dreamy if a more readable [Fluent API][fluent-api-fowler] were available - such as this one?

{% highlight c# %}
var response = new Retry<Response>()
  .StartWith(Responses.INVALID)
  .Do(() => MakeCallToService(request))
  .BailoutWhen(RetryMayNotHaveAnyEffect)
  .AtMost(thrice)
  .Execute();
{% endhighlight %}

See how well this reads. Poetic (almost)!
Fortunately, it is not too difficult to cook up an implementation for the `Retry` fluent interface. The idea is to accumulate all building blocks - such as what defines the service call, what it means to short-circuit the retry mechanism, how many times should retry be performed at max, etc. - into a __context__ object. In this case, an object of the `Retry` class serves as the context.

Method chaining can be used to enrich the context with various building blocks. Keep returning `this` from each building block, so that the next building block can be executed.

{% highlight c# %}
public class Retry<T> {
  private int maxTimes;
  private Func<T> task;
  private Predicate<T> shouldBailOut = obj => false;
  private T initialValue;

  public Retry<T> StartWith(T initial) {
    initialValue = initial;
    return this;
  }

  public Retry<T> AtMost(int times) {
    maxTimes = times;
    return this;
  }

  public Retry<T> Do(Func<T> func) {
    task = func;
    return this;
  }

  public Retry<T> BailoutWhen(Predicate<T> condition) {
    shouldBailOut = condition;
    return this;
  }

  public T Execute() {
    int trials = 1;
    T result = initialValue;
    while (trials <= maxTimes) {
      result = task();
      if (shouldBailOut(result))
        break;
      ++trials;
    }
    return result;
  }
}
{% endhighlight %}

It's a matter of personal choice to give or not to give sensible defaults to the building blocks.
One may also add validations within the `Execute` method to check if all the necessary building blocks are in place.

Let's write a quick and sweet test to assert that atmost N retries are carried out:

{% highlight c# %}
[Test]
public void ShouldRetryNotMoreThanTheSpecifiedNumberOfTimes() {
  var init = 0;
 
  int final = new Retry<int>().StartWith(init)
    .AtMost(thrice)
    .Do(() => ++init)
    .Execute();

  Assert.That(final, Is.EqualTo(3));
}
{% endhighlight %}

Ofcourse, fluent API is not a silver bullet to the world's problems. You silently inherit the problem of [temporal coupling][temporal-coupling]: instead of passing in the must-have building blocks (like task to perform, maximum number of tries) in a constructor, you introduce methods like `Do` and `Atmost` which __must be__ called before `Execute`. Fluent APIs are also not very debug-friendly.

In the end, it is about how the artist in you wants to express herself/himself.

[fluent-api-fowler]: http://martinfowler.com/bliki/FluentInterface.html
[temporal-coupling]: http://blog.ploeh.dk/2011/05/24/DesignSmellTemporalCoupling/