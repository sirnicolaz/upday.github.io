---
layout: post
title: "Subjects - Neither Good nor Evil"
modified:
categories: blog
author: tomek_polanski
excerpt: Why are the Rx Subjects so often misunderstood?
tags: [RxJava]
image:
date: 2016-07-18T13:00:55-01:00
---

I’ve heard plenty of mixed feelings about RxJava ``Subjects`` - some developers I’ve spoken to say they are evil, others say they are the best thing since sliced bacon.

In my early days of using Rx, I thought they were awesome. I was able to create ``Observables`` without the fancy operators like ``fromCallable``, ``from`` or ``create``. In a <a href="http://futurice.com/blog/reactive-c-number-in-practice">blog post</a> from 2014, I even wrote

>*I would not advise using Observable.Create() as Subject is much easier to use and good enough 99% of time*.

Since then I’ve learnt that they are just simple tools to start an ``Observable`` stream and by themselves tools are neither good nor evil, it just depends how we use them.

## The ease of usage...

The best thing about ``Subjects`` is how easy they are to use. You can simply push a value in one place and use it somewhere else as a normal ``Observable``:

{% highlight java %}
PublishSubject<String> subject = PublishSubject.create();
subject.subscribe(text -> Log.d("Subjects: ", text));
subject.onNext("Easy to use");
{% endhighlight %}

This is because ``Subjects`` extend ``Observable`` and implement ``Observer``.

Thanks to being an ``Observable``, we can use all the typical operators (``map``, ``flatMap``, etc) without any extra hassle:

{% highlight java %}
subject.filter(str -> !str.isEmpty())
       .map(str -> "This is a not empty string: " + str);
{% endhighlight %}

The benefit of being an ``Observer`` is that we can directly put any value to ``onNext``, throw an exception in ``onError`` or just complete.

{% highlight java %}
subject.onNext("Some value");
subject.onComplete();
{% endhighlight %}

## ... comes with a price

Some say that ``Subjects`` are broken, but I would disagree. The creators of ``Subjects`` did a great job, they've focused on creating a user-friendly tool, despite some limitations.

With ``Subjects``, a great power comes with a great burden.

### Backpressure

Basically, backpressure exception occurs when an operator produces values faster than you can handle further downstream.

I like to compare operators that can handle backpressure to waiters. If clients of a restaurant are full, then they would tell the waiter to stop bringing food and naturally, no more food will be served. On the other hand, operators that cannot handle backpressure are like grandmas. If you tell her that you cannot eat any more, she will just ignore you and keep on feeding you more food until you explode or run away.


The issue with ``Subjects`` and backpressure comes from the fact that ``Subjects`` do not extend ``Subscriber``. As <a href="https://twitter.com/akarnokd">Dávid Karnok</a> mentioned in his blog, ``Subscriber`` is the main way to handle backpressure in RxJava. Unfortunately, Java does not have multiple inheritance. The designers have chosen to make ``Subjects`` simpler to use rather than support backpressure.

There is really simple test to check if your ``Observable`` supports backpressure. The trick is to request via ``TestSubscriber`` less elements than we put into the ``Subject``:

{% highlight java %}
@Test
public void check_backpressure_support() {
    // The Subscriber requests only 1 value:
    TestSubscriber<Boolean> ts = new TestSubscriber<>(1);
    PublishSubject<Boolean> subject = PublishSubject.create();
    subject.subscribe(ts);

    subject.onNext(true); // We put 2 values into the stream
    subject.onNext(true);

    ts.assertValueCount(1); // We should receive only 1 value
}
{% endhighlight %}

Let’s assume ``ts.assertValueCount(1)`` passes, that would mean that subject supports backpressure. But ``PublishSubject`` does not support backpressure, so in this case it fails as there will be two values emitted by the ``Subject``.

**Solution:**

Backpressure can happen to ``Subjects`` when we put items into the ``Subject`` faster than we are processing it downstream.

This could happen in two cases:

- We put items to the ``Subject`` really frequently (perhaps, hundreds items per second).
- An Operator or subscribe takes too long in processing the events:

{% highlight java %}
subject.subscribe(value -> {
    try {
        Thread.sleep(1000);
        Log.e("", value);
    } catch (InterruptedException e) {
    }
});
{% endhighlight %}

You probably won’t have backpressure problems if you only put events into the ``Subject`` infrequently. However, if you are not so fortunate you can handle it using one of the ``onBackpressure`` operators. For example:

{% highlight java %}
subject.onBackpressureDrop().map(...)
{% endhighlight %}

We could then alter the previous test to make the ``Subject`` in question handle backpressure, as follows:

{% highlight java %}
@Test
public void check_backpressure_support() {
    // The Subscriber requests only 1 value:
    TestSubscriber<Boolean> ts = new TestSubscriber<>(1);
    PublishSubject<Boolean> subject = PublishSubject.create();
    subject.onBackpressureDrop().subscribe(ts);

    subject.onNext(true); // We put 2 values into the stream
    subject.onNext(true);

    ts.assertValueCount(1); // We should receive only 1 value
}
{% endhighlight %}

Remember, ``onBackpressure`` operators fix only the symptom, not the root cause of the issue; you still have a bottleneck in your stream that you should try to improve!

### ``subscribeOn``

An additional challenge comes with ``subscribeOn``, it specifies on what thread the subscription should be done and on what thread the downstream should be executed (unless changed with ``observeOn``).
To use subscribeOn properly we have to know how the stream started - if it was created from ``Observable.create``, ``Observable.from``, ``Observable.just`` or it was a ``Subject``.
As I’ve mentioned in my <a href="https://upday.github.io/blog/subscribe_on/">previous post</a>, ``subscribeOn`` does not work with the ``Subjects``.

**Solution:**

If you want the downstream to be executed on another thread than the one where the onNext was triggered, always call ``subject.observeOn(...)`` before.

## Conclusion

In the early days, ``Subjects`` really helped me to get on the “reactive” horse.  Without them learning Rx would have been an even harsher process.
If you choose to use ``Subjects``, be aware of their strengths and weaknesses.
The alternative is to create your own Subject-like utility as <a href="https://github.com/upday/RxProxy">we did</a> in our project.


>*I really enjoyed my journey with Reactive Programming and hope it gains traction in the development world.*
<p align="right">
-  <a href="http://futurice.com/blog/reactive-c-number-in-practice">Rx in practice</a>
</p>
