---
layout: post
title: "Periodically sending JVM stats to StatsD in Scala"
date: 2014-01-01 15:10
comments: true
tags: [scala, stats, akka]

---


I have written a small library in Scala to allow periodic sending of JVM stats to StatsD server. One can include it in server written in Scala so that its JVM stats can be monitored over time. I am using the following [Scala class](https://github.com/etsy/statsd/blob/master/examples/StatsD.scala) to send the stats to StatsD. Also the technique to extract JVM stats is borrowed from twitter's excellent [Ostrich library](https://github.com/twitter/ostrich)

Following is the class that schedules sending of the stats periodically

{% highlight scala %}
package com.parthpatil.jvmstatsd

import scala.collection.mutable
import java.util.concurrent.{TimeUnit, Executors}

class StatsSender(statsd: StatsD, numSchedulerThreads: Int = 2) {
  // Tasks that need to be scheduled to send stats to statsd
  private val tasks = mutable.ArrayBuffer[(StatsTask, Int)]()
  val fScheduler    = Executors.newScheduledThreadPool(numSchedulerThreads)

  def addTask(task: StatsTask, secs: Int) {
    task.setStatsD(statsd)
    tasks += ((task, secs))
    fScheduler.scheduleWithFixedDelay(task, 0, secs, TimeUnit.SECONDS)
  }

  def getTasks(): Seq[(StatsTask, Int)] = {
    tasks.toSeq
  }
} 
{% endhighlight %}

Following is the interface base class for a stats task that publishes stats to StatsD using the StatsD class

{% highlight scala %}
package com.parthpatil.jvmstatsd

import java.text.DecimalFormat

class StatsTask() extends Runnable {
  protected var statsd: StatsD = null

  // formatter to remove scientific notation from doubles
  val formatter = new DecimalFormat("#")

  override def run() { }
  def setStatsD(sd: StatsD) { statsd = sd }
}
{% endhighlight %}

Following is a sample implementation of the StatsTask that publishes JVM stats. You can write your won periodic stats publisher by extending StatsTask and adding it to StatsSender via addTask() method.

{% highlight scala %}
package com.parthpatil.jvmstatsd

import java.lang.management.ManagementFactory
import scala.collection.mutable
import scala.collection.JavaConverters._
import com.twitter.conversions.string._

class JvmStatsTask extends StatsTask {
  override def run() {
    getJvmGauges() foreach { case (k, v)  => statsd.gauge(k, formatter.format(v)) }
    getJvmCounters() foreach { case (k,v) => statsd.increment(k, v) }
  }

  def getJvmGauges(): Map[String, Long] = {
    val out = mutable.Map[String, Long]()

    val mem = ManagementFactory.getMemoryMXBean()

    val heap = mem.getHeapMemoryUsage()
    out += ("jvm.heap.committed" -> heap.getCommitted())
    out += ("jvm.heap.max" -> heap.getMax())
    out += ("jvm.heap.used" -> heap.getUsed())

    val nonheap = mem.getNonHeapMemoryUsage()
    out += ("jvm.nonheap.committed" -> nonheap.getCommitted())
    out += ("jvm.nonheap.max" -> nonheap.getMax())
    out += ("jvm.nonheap.used" -> nonheap.getUsed())

    val threads = ManagementFactory.getThreadMXBean()
    out += ("jvm.thread.daemon_count" -> threads.getDaemonThreadCount().toLong)
    out += ("jvm.thread.count" -> threads.getThreadCount().toLong)
    out += ("jvm.thread.peak_count" -> threads.getPeakThreadCount().toLong)

    val runtime = ManagementFactory.getRuntimeMXBean()
    out += ("jvm.start_time" -> runtime.getStartTime())
    out += ("jvm.uptime" -> runtime.getUptime())

    val os = ManagementFactory.getOperatingSystemMXBean()
    out += ("jvm.num_cpus" -> os.getAvailableProcessors().toLong)
    os match {
      case unix: com.sun.management.UnixOperatingSystemMXBean =>
        out += ("jvm.fd.count" -> unix.getOpenFileDescriptorCount)
        out += ("jvm.fd.limit" -> unix.getMaxFileDescriptorCount)
      case _ =>   // ew, Windows... or something
    }

    var postGCTotalUsage = 0L
    var currentTotalUsage = 0L
    ManagementFactory.getMemoryPoolMXBeans().asScala.foreach { pool =>
      val name = pool.getName.regexSub("""[^\w]""".r) { m => "." }
      Option(pool.getCollectionUsage).foreach { usage =>
        out += ("jvm.post_gc." + name + ".used" -> usage.getUsed)
        postGCTotalUsage += usage.getUsed
        out += ("jvm.post_gc." + name + ".max" -> usage.getMax)
      }
      Option(pool.getUsage) foreach { usage =>
        out += ("jvm.current_mem." + name + ".used" -> usage.getUsed)
        currentTotalUsage += usage.getUsed
        out += ("jvm.current_mem." + name + ".max" -> usage.getMax)
      }
    }
    out += ("jvm.post_gc.used" -> postGCTotalUsage)
    out += ("jvm.current_mem.used" -> currentTotalUsage)

    out.toMap
  }

  def getJvmCounters(): Map[String, Long] = {
    val out = mutable.Map[String, Long]()

    var totalCycles = 0L
    var totalTime = 0L

    ManagementFactory.getGarbageCollectorMXBeans().asScala.foreach { gc =>
      val name = gc.getName.regexSub("""[^\w]""".r) { m => "." }
      val collectionCount = gc.getCollectionCount
      out += ("jvm.gc." + name + ".cycles" -> collectionCount)
      val collectionTime = gc.getCollectionTime
      out += ("jvm.gc." + name + ".msec" -> collectionTime)
      // note, these could be -1 if the collector doesn't have support for it.
      if (collectionCount > 0)
        totalCycles += collectionCount
      if (collectionTime > 0)
        totalTime += gc.getCollectionTime
    }
    out += ("jvm.gc.cycles" -> totalCycles)
    out += ("jvm.gc.msec" -> totalTime)

    out.toMap
  }
}
{% endhighlight %}

Sample Usage

{% highlight scala %}
package com.parthpatil.jvmstatsd

import akka.actor.ActorSystem
import scala.concurrent.ExecutionContext.Implicits.global

class SampleUsage {
  def main(args: Array[String]) {
    val actorSystem = ActorSystem("statsd-actor-system")
    val statsD = new StatsD(
      context = actorSystem,
      host = "localhost",
      port = 8192)

    val statsSender = new StatsSender(statsD)

    // Create the JvmStatsTask that publishes stats every 10 secs
    statsSender.addTask(task = new JvmStatsTask, secs = 10)
  }
}
{% endhighlight %}

You can find the full source in this [repo](https://github.com/parth-patil/JvmStatsD)
