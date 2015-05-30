---
layout: post
title: "Redis Job Queue With Retry Ability and Observable Interface"
comments: true
tags: [scala, async, future, observable, redis]
---

In this blog post I will explain how I implemented a simple Job queue in Redis that can have multiple workers pulling work from it and processing it. The job queue also offers a way to retry failed jobs upto certain number of times.

I have used the [Redis Sorted Set](http://redis.io/commands#sorted_set) data structure to hold the Tasks to be processed. The time when the job needs to be processed is represented by the `score` component of the sorted set and the actual Task is the `member` component. So when Tasks are added to the queue for the first time the time when they should be processed is given as the current timestamp. When the job fails a `future` timestamp is calculated and the score for the failed job is updated with this future timestamp. The consequence of this is that Tasks whose processing should be attempted farther in the future will be farther away from the front of the queue. 

The Redis sorted set is always sorted by high to low but I want my Tasks to be sorted from low to high so that the oldest Task eligible for processing/reprocessing is in the front of the queue. To achieve the correct sort I don't use the epoch time as is for the score but I use `MAX_EPOCHTIME - CURRENT_TS`.

So when Task fails its the responsibility of the client to put the Task back on the queue with the right next attempt timestamp and also update the `numFailures` so that the next time the Task fails while processing the processing worker can decide to discard the Task if the Task has crossed the num attempts threshold or to requeue the job with a future timestamp of when the Task processing should be reattempted.

Again I am using the [lettuce](https://github.com/mp911de/lettuce) Java client for Redis that I had used in the [previous blog post]({% post_url 2015-05-24-using-redis-via-observable %}). This library returns [Guava Futures](https://code.google.com/p/guava-libraries/wiki/ListenableFutureExplained) for most of its operations. Though Guava Futures are much better than the [JDK Future](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html), they still are not as easy to work with as Scala Future. So I have provided the following implicit conversion from Guava Future to Scala Future.

{%highlight scala linenos%}
implicit def guavaFutureToScalaFuture[T](gFuture: ListenableFuture[T])
                                        (implicit executor: ListeningExecutorService): Future[T] = {
  val p = Promise[T]()
  Futures.addCallback[T](gFuture, new FutureCallback[T] {
    def onSuccess(s: T)         { p.success(s) }
    def onFailure(e: Throwable) { p.failure(e) }
  }, executor)
  p.future
}
{%endhighlight%}

Secondly I need a java ExecutorService to run the Guava Futures and I need a scala ExecutionContext to run the scala Futures and I wanted to share the same thread pool for both these things. So I create a Java ExecutorService instance and then create an ExecutionContext to run Scala Futures and a ListeningExecutorService to run the Guava Futures. Following is the code.

{%highlight scala linenos%}
val executorService: ExecutorService = Executors.newFixedThreadPool(4)
implicit val executionContext = ExecutionContext.fromExecutorService(executorService)
implicit val executor = MoreExecutors.listeningDecorator(executorService)
{%endhighlight%}

Now in order to know if the task in the front of queue is ready to be reprocessed we have to check what its score is which if you remember is the timestamp (MAX_EPOCHTIME - TIMESTAMP) indicating when to attempt processing of this task. One way to do this is to get the Task from the top of the queue and if its next attempt timestamp is less than current timestamp then process it and delete it from the queue. If the timestamp is in the future then put the Task back on the queue.

To make these set of operations atomic and low cost(save on network roundtrip to Redis) I decided to implement the `pop` operation of the queue via a [Lua Script](http://redis.io/commands/EVAL). The Lua script runs on the server side as a transaction which is very useful as I want to make the pop operation atomic (pop & delete from queue). Following is what the script looks like.

{%highlight lua linenos%}
local zset_key = KEYS[1]
local reverse_current_ts = ARGV[1]
local max_ts = ARGV[2]

-- Get all items older than current timestamp
local arr = redis.call('ZRANGEBYSCORE', zset_key, reverse_current_ts, max_ts)
local arr_size = table.maxn(arr)

if (arr_size > 0) then
  -- Delete these items from the zset
  redis.call('ZREMRANGEBYSCORE', zset_key, reverse_current_ts, max_ts)
  return arr
else
 return {}
end
{%endhighlight%}

To the Lua script I pass the key for the redis sorted set, the reverse current ts (max epoch time - current ts) and the max epoch timestamp. The script checks if there are any Tasks with next attempt timestamp in the past and if yes it will fetch them, delete them from the queue and return them. Make sure to use `local` to create non-global variables in Redis Lua scripts.

Now lets look at the interesting part of exposing an Observable queue over this Redis Sorted Set. My first attempt was the way I have used Observable before by explicitly using the Observable's apply method. Following is how the code looks. It uses the `getTask()` method that returns a `Future[Seq[Task]]`.

{%highlight scala linenos%}
Observable.interval(pollInterval) flatMap { i =>
  Observable[Task] { subscriber =>
    getTasks foreach { tasks =>
      tasks foreach { subscriber.onNext(_) }
    }
  }
}
{%endhighlight%}

Note that I am not doing any error handling in the above code to keep the examples small. So I have an outer Observable that emits events every `pollInterval` duration. Inside that I am wrapping a call to `getTasks` in another Observable. Note that I am using `flatMap` at the top level else I will end up with `Observable[Observable[Task]]`.

Though the above approach to construct Observable is not bad for this small example but Erik Meijer has strongly urged in the Reactive Programming course to not construct Observables directly via its apply method or via `create` but to try to use combinator methods provided by Observable to construct new ones. So here is what I came up with.


{%highlight scala linenos%}
for {
  _         <- Observable.interval(pollInterval)
  tasks     <- Observable.from(getTasks) // Get Observable from Future[Seq[Task]]
  flattened <- Observable.from(tasks) // Get Observable from Seq[Task]
} yield {
  flattened
}
{%endhighlight%}

Note that above on line 4 I do another `Observable.from()` because I don't want to end up with `Observable[Seq[Task]]` but I want to get `Observable[Task]`.

Following is a simple case class to represent the Task 
{%highlight scala linenos%}
case class Task(created: Long, numFailures: Int, payload: String) {
  def toJValue(): JValue = {
    ("created" -> created) ~
    ("numFailures" -> numFailures) ~
    ("payload" -> payload)
  }

  override def toString(): String = {
    compact(render(toJValue))
  }
}
{%endhighlight%}

Here is how the client uses the job queue

{%highlight scala linenos%}
val client = new RedisClient("127.0.0.1")
val asyncConnection = client.connectAsync()

val rcq = new RedisConditionalQueue(
  asyncConnection = asyncConnection,
  conditionCheckingLuaScript = None,
  zsetKey = "sorted1",
  executorService = executorService)

rcq.getObservableQueue(pollInterval = 1 second) subscribe { task =>
  println(s"received task -> $task ")

  // If the task fails enqueue it back with a timestamp in the future
  processTask(task) onComplete {
    case Success(_) =>
      println(s"Job Success!, task = $task")
    case Failure(e) =>
      println(s"Job Failed!, task = $task")
      val totalFailures = task.numFailures + 1
      val newTask = task.copy(numFailures = totalFailures)
      if (totalFailures < MAX_ALLOWED_FAILURES) {
        val nextAttemptTs = System.currentTimeMillis + 2 * totalFailures * 1000
        println(s"Reenqueued -> $newTask")
        rcq.enqueue(newTask, nextAttemptTs)
      } else {
        println(s"Discarding task -> $newTask, totalFailures = $totalFailures")
      }
  }
}
{%endhighlight%}

The entire code for this example is in this [gist](https://gist.github.com/parth-patil/902ad07752829d5deba6)