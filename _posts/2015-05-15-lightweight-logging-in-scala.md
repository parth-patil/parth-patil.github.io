---
layout: post
title: "Lightweight logging in Scala"
comments: true
tags: [scala, hbase, asynchbase]
---

Sometimes you want to defer the evaluation of log lines from the point where they are logged. This helps avoid paying the cost of logging in the case when logging has been turned off.
Following is a simple logger that lets you do that. This code is for demonstration purposes only and might have potential bugs/performance issues.

{%highlight scala%}
package com.parthpatil

import java.util.concurrent.ConcurrentLinkedQueue
import scala.collection.JavaConverters._

class LightweightLogger {
  type DeferedString = () => String

  private val buffer = new ConcurrentLinkedQueue[DeferedString]()

  def log(expression: => String): Unit = {
    buffer.add(() => expression)
  }

  def getLinesAndFlush(): Seq[String] = {
    def getBufferContents(): Seq[DeferedString] = buffer.synchronized {
      val output = buffer.iterator.asScala.toSeq
      buffer.clear()
      output
    }

    val contents = getBufferContents()
    contents map { function => function() }
  }
}

object LightweightLogger extends App {
  val logger = new LightweightLogger
  logger.log(s"Expensive compute -> ${100 * 200}")
  logger.log(s"Expensive compute -> ${100.0 / 200}")
  val lines = logger.getLinesAndFlush()
  println(lines mkString ",")
}
{%endhighlight%}

The `log` method has a by name parameter to capture the expression that will generate a String. This by name parameter(expression) is not evaluated but is wrapped in a function and put on the queue. Only when the `getLinesAndFlush()` is invoked is when each item on the queue is evaluated.

Though this approach has the downside that you could end up closing over big objects and hence cause them to not get garbage collected for longer than they are needed and could put pressure on your memory or worse the objects you have closed over might no longer be in a usable state.