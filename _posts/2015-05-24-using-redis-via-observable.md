---
layout: post
title: "Using Redis via Observable"
comments: true
tags: [scala, async, future, observable]
---

In the previous [blog post]({% post_url 2015-05-18-scala-future-vs-observable-part-1 %}) we saw how observable was used to create an abstraction over Apache Tailer. In this article I am going to use Observable to abstract over some Redis operations.

I chose [lettue](https://github.com/mp911de/lettuce) for this exercise. It has good support for the scan based operations(scan, hscan etc) in Redis and I was interested in exposing these operations via Observable.

Lets first see how to expose the PubSub APIs subscription as an Observable.

{%highlight scala%}
val obs = Observable[(String, String)] { subscriber =>
  val connection: RedisPubSubConnection[String, String] = client.connectPubSub()
  connection.addListener(new RedisPubSubAdapter[String, String]() {
    override def message(chan: String, msg: String): Unit = {
      if (!subscriber.isUnsubscribed)
        subscriber.onNext((chan, msg))
    }
  })
  connection.subscribe("test-channel")
}

obs subscribe { r => println(s"chan: ${r._1}, msg: ${r._2}") }
{%endhighlight%}

Pretty straightforward. The lettuce library provides a nice listener based API to subscribe to a given channel and I just wrapped that in the Observable's factory method. I have exposed the stream from the pubsub system as a tuple stream of channel & message.

Similarly we can expose iterating over all the keys in Redis as an Observable. Note that it is using the non-blocking `scan` API instead of the blocking `keys` redis operation.

{%highlight scala%}
Observable[String] { subscriber =>
  asyncConnection.scan(new KeyStreamingChannel[String]() {
    override def onKey(key: String) {
      if (!subscriber.isUnsubscribed)
        subscriber.onNext(key)
    }
  })
} subscribe { key => println(s"key => $key") }
{%endhighlight%}

Lettuce again provides a decent streaming API (`KeyStreamingChannel`) that I have wrapped in an Observable.

The code for wrapping `HSCAN` in an Observable looks almost similar to the code for `SCAN`

{%highlight scala%}
Observable[(String, String)] { subscriber =>
  asyncConnection.hscan(new KeyValueStreamingChannel[String, String]() {
    override def onKeyValue(key: String, value: String) {
      if (!subscriber.isUnsubscribed)
        subscriber.onNext((key, value))
    }
  }, "my-key")
} subscribe { kv => println(s"key => ${kv._1}, value = ${kv._2}") }
{%endhighlight%}

The full source code for this article can be found in this [gist](https://gist.github.com/parth-patil/8d105d82b745a6f69cb1)