---
layout: post
title: "HBase scanning in Scala using asynchbase"
date: 2014-09-16 23:23
comments: true
tags: [scala, hbase, asynchbase]
---

I was looking for a way to do non-blocking recursive scans on HBase in Scala using the excellent [asynchbase Java library](https://github.com/OpenTSDB/asynchbase). I wanted to create Scala Futures from the Deferred that asynchbase returns as Futures are much easier to work in Scala. Following is what I came up with, its not perfect and lacks a lot of error handling. It is inspired by the follwing [gist](https://gist.github.com/tsuna/5480390)

To keep the example simple I am going to assume that we are only interested in returning the keys of our hbase table.

{% highlight scala %}

import com.stumbleupon.async.Callback
import org.hbase.async._
import org.slf4j.LoggerFactory
import scala.concurrent.{Promise, Await, ExecutionContext, Future}
import java.util
import scala.collection.mutable
import scala.util.Try

class RecursiveResultHandler(
  scanner: Scanner,
  promise: Promise[Seq[String]]) extends Callback[Object, util.ArrayList[util.ArrayList[KeyValue]]] {

  val logger = LoggerFactory.getLogger(getClass)
  val startTime = System.currentTimeMillis()
  
  // Initialize the collection to hold our results
  val results = mutable.ArrayBuffer[String]()
  var numRows: Int = 0

  def call(rawRows: util.ArrayList[util.ArrayList[KeyValue]]) = {
    try {
      // Once result of nextRows is null, we have reached the end of scan
      if (rawRows == null) {
        promise.success(results.toSeq)
        val timeTaken = System.currentTimeMillis - startTime
        logger.info(s"Num Rows = $numRows, Total time = $timeTaken ms")
        Try { scanner.close() } // close scanner & ignore exceptions
      } else {
        numRows += rawRows.size
        val rowsIterator = rawRows.iterator()
        while (rowsIterator.hasNext()) {
          val row = rowsIterator.next()
          val key = new String(row.get(0).key)
          results += key
        }

        scanner.nextRows().addCallback(this)
      }
    } catch {
      case e: Throwable =>
        promise.failure(e)
        Try { scanner.close() } // close scanner & ignore exceptions
    }
  }
}
{% endhighlight %}

Following is how you would use the above class

{%highlight scala %}
val promise = Promise[Seq[String]]()
try {
  val hbaseClient = new HBaseClient("zookeeperHost", "/zookeeper/hbase/path")
  val scanner: Scanner = hbaseClient.newScanner("my_table")

  val rrh = new RecursiveResultHandler(
    scanner = scanner,
    promise = promise
  )
  scanner.nextRows().addCallback(rrh)
} catch {
  case e: Throwable =>
    promise.failure(e)
}

val fut = promise.future

// use the future and profit !
{%endhighlight%}

