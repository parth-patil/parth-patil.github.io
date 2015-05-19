---
layout: post
title: "Scala Future vs Observable Part 1"
comments: true
tags: [scala, async, future, observable]
---

I have been taking the enlightening coursera course `Principles Of Reactive Programming` and it has been fun so far. I have been using scala `Future` for some time and really enjoy working with it. But I hadn't used `Observable` (Rx) till now and found them intriguing. Though I was not sure when to use one API over other. I thought implementing a few uses cases in both the APIs will help me understand the differences and see where the strength of each API lies in helping write good async code that is also composable.

Following is a naive attempt to wrap the Apache Commons Tailer into the Future and Rx APIs

Following is a helper class that lets us create a tailer using a simple handler function

{%highlight scala%}
// The following class gives a way to pass a simple function 
// from String => Unit as a handler for each log line
class MyTailer(
  filePath: String,
  handler: String => Unit,
  pollInterval: Duration,
  tailFromEnd: Boolean = true) {

  val listener = new TailerListenerAdapter {
    override def handle(line: String) { handler(line) }
  }

  val tailer = Tailer.create(
    new File(filePath),
    listener,
    pollInterval.toMillis,
    tailFromEnd)

  def stop(): Unit = { tailer.stop() }
}
{%endhighlight%}


Lets first look at the implementation using Future.

{%highlight scala%}
class FutureTailer(filePath: String, pollInterval: Duration) {
  // Create a queue to collect the tailed lines
  val queue = new LinkedBlockingQueue[String]

  val tailer = new MyTailer(filePath, { queue.add(_) }, pollInterval)

  def next(): Future[String] = Future { blocking { queue.take() } }
}

// Usage
val fTailer = createFutureTailer("/path/to/file", 100 millisecond)

def onCompleteHandler(tryLine: Try[String]): Unit = {
  tryLine match {
    case Success(line) =>
      fTailer.next() onComplete onCompleteHandler
      println(s"---------- $line ------------")

    case Failure(e) =>
  }
}

fTailer.next() onComplete onCompleteHandler
{%endhighlight%}

The Future based solution is implemented in a `pull` based manner. The library provides a next() method that when called gives you back a `Future[String]` corresponding to the line from the file that will be returned when its available.

To achive this I had to introduce a queue that the handler will populate as and when a new line is available. I then block on the queue to get a line which will be used to fulfill the Future that next() had previous returned.

I am not happy with the Future based version of this library for the following reasons:

* I have to explicitly maintain a queue
* The calling code is big and does not convey the intent quickly 
* The way next() is called recursively is confusing and its easy to screw up if you are not careful and cause a lot of parallel Futures to get created, ultimately causing OOM.

Now lets look at the Observable based implementation

{%highlight scala%}
def createObservableTailer(
  filePath: String,
  pollInterval: Duration): Observable[String] = {
  Observable { subscriber =>
    def handler(line: String): Unit = {
      if (!subscriber.isUnsubscribed)
        subscriber.onNext(line)
    }
    new MyTailer(filePath, handler, pollInterval)
  }
}

// Usage
val oTailer = createObservableTailer(file, 100 millisecond)
oTailer foreach { line =>
  println(s"---------- $line ------------")
}
{%endhighlight%}

As you can see the Observable based implementation appears more compact and easy to comprehend.

In summary I think this use case is very well suited to `push` based approach implemented using Observable than the `pull` based approach using Future.

The full code can be found here [Full Code](https://gist.github.com/parth-patil/bd374efd6e8ab5b79b5b)
