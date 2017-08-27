---
title: "First"
date: 2017-08-27T11:41:29-04:00
draft: true
---

R comes with a number of built-in FP tools, one of which is the `Reduce` function which recursively calls a binary function over a vector of inputs. It's quite versatile, but most explanations of `Reduce` tend to focus on trivial use cases that are not particularly inspiring. Even Hadley's _Advanced R_, which is an otherwise fantastic reference, limits the discussion to implementations of cumulative sum---for which there is already `cumsum`---and multiple intersect. I'll use this post to demonstrate slightly more sophisticated examples of how `Reduce` can come in handy, based on an alternative interpretation of the function as a finite state machine. The examples aren't necessarily applicable to day-to-day programming tasks, but rather interesting enough that the reader will leave with a more expansive understanding of what `Reduce` can do.

I think it's appropriate to start by revisiting an example from _Advanced R_ where Hadley claims it is _not_ possible to use a functional. Of a for-loop implementation of an [exponential moving average](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average) (also _exponential smooth_), he writes, "We can't eliminate the for loop because none of the functionals we've seen allow the output at position `i` to depend on both the input and output at position `i-1`" ([Chapter 11, page 226](http://adv-r.had.co.nz/Functionals.html#functionals-not)).

This is precisely what `Reduce` does! In fact, given `x` and smoothing parameter `alpha`, we can compute the EMA without the use of an explicit for loop:


{% highlight r %}
exps <- function(x, alpha) {
  Reduce(function(s, x_) alpha*x_ + (1-alpha)*s, x, accumulate=TRUE)
}

x <- runif(30)

plot(x, type='l')
lines(exps(x, alpha=.2), col='red')
{% endhighlight %}

![plot of chunk unnamed-chunk-1](/assets/Rfig/unnamed-chunk-1-1.svg)

(As a reminder, specifying `accumulate=TRUE` causes `Reduce` to return all intermediate results, so that the output has the same length as `x`. We can also specify an initial value with the `init` argument, in which case the output will have length `length(x) + 1`.)


More generally, `Reduce` can implement a simple state-space model (a "process") where the state is updated according to its previous value and an input. In the EMA example, the state is `s` and the input is `x_`, an element of `x`. This conception of `Reduce` is related to the idea of applying a binary function recursively, but perhaps provides a more natural way of thinking about certain kinds of problems.

Suppose next that we'd like to compute a cumulative sum that "resets" after encountering the value zero. All we need to do is write a function that takes the current sum (the state) as given, and either adds to or resets that sum depending on the value of the second argument (the input):

{% highlight r %}
x <- c(1,2,4,0,-1,3,0,1,1,1)

Reduce(function(s, x_) (s + x_)*(x_ != 0), x, accumulate=TRUE)
{% endhighlight %}



{% highlight text %}
##  [1]  1  3  7  0 -1  2  0  1  2  3
{% endhighlight %}
Easy!

We can also use `Reduce` to implement a version of `zoo::na.locf` ("last observation carry forward"), which replaces `NA` with the last non-`NA` value:


{% highlight r %}
x <- rep(NA_integer_, 14)
x[sample(length(x), 5)] <- 1:5
x
{% endhighlight %}



{% highlight text %}
##  [1] NA NA NA NA  5 NA  1 NA  4 NA  3 NA  2 NA
{% endhighlight %}



{% highlight r %}
Reduce(function(s, x_) if (is.na(x_)) s else x_, x, accumulate=TRUE)
{% endhighlight %}



{% highlight text %}
##  [1] NA NA NA NA  5  5  1  1  4  4  3  3  2  2
{% endhighlight %}

Suppose we have a vector of 0, 1, and -1 which represent actions peformed on some state which can be 0 or 1. If the action "1" is performed, the state switches to (or remains) "1"; if "0" is performed, there is no effect; and if "-1" is performed, the state switches to (or remains) "0". (Think of this as flipping a light switch on or off.) We can use `Reduce` to compute the values of the state given the sequence of actions `a` and an initial state:


{% highlight r %}
(a <- sample(c(-1,0,1), size=14, replace=TRUE))
{% endhighlight %}



{% highlight text %}
##  [1]  1  1 -1  1  1  1  1 -1  0 -1  0 -1  0  1
{% endhighlight %}



{% highlight r %}
Reduce(function(s, a_) as.integer(s + a_ > 0), a, init=1L, accumulate=TRUE)
{% endhighlight %}



{% highlight text %}
##  [1] 1 1 1 0 1 1 1 1 0 0 0 0 0 0 1
{% endhighlight %}

Another example from [StackOverflow](https://stackoverflow.com/questions/41073261/). Say we have a binary vector `x` and want to replace the two elements following every "1" with "1". We can use `Reduce` to create a counter that decrements by 1 in the case of "0" and resets to 3 in the case of "1", so that any two "0" elements following a "1" correspond to positive values in the counter. Taking the `sign` then gives the desired result:

{% highlight r %}
x <- rep(0, 14)
x[sample(length(x), 5)] <- 1
x
{% endhighlight %}



{% highlight text %}
##  [1] 1 1 0 0 1 0 0 0 0 0 0 1 1 0
{% endhighlight %}



{% highlight r %}
(counter <- Reduce(function(s, x_) min(3, max(s-1, 0) + x_), 3*x, accumulate=TRUE))
{% endhighlight %}



{% highlight text %}
##  [1] 3 3 2 1 3 2 1 0 0 0 0 3 3 2
{% endhighlight %}



{% highlight r %}
sign(counter)
{% endhighlight %}



{% highlight text %}
##  [1] 1 1 1 1 1 1 1 0 0 0 0 1 1 1
{% endhighlight %}

This solution takes some thought, and perhaps an alternative method would be preferable for the sake of clarity. But the point is that `Reduce` is flexible enough to be a contender for a much wider range of problems than is usually recognized.

I hope that some of the above examples help to illuminate . Used cleverly, `Reduce` can provide a compact solution

<hr>
