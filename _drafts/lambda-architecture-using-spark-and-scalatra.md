---
layout: post
title: "Lambda Architecture Using Spark and Scalatra"
comments: true
tags: [scala, spark, big-data]
---
I have read quite a few articles that use Spark plus some external databases to do [Lambda Architecture](https://en.wikipedia.org/wiki/Lambda_architecture). I was wondering if I could achieve lambda architecture using only Spark and no external databases/datastores (except HDFS). Ofcourse doing this will also mean that the latencies will be higher when querying but I am ok with that.

Quick overview of Lambda Architecture:
Lambda architecture is a technique to provide close to realtime access to big data albeit with a slight error in accuracy. This error % in accuracy can be tuned that I will explain later.
The Lambda Architecture consits of 3 components 
1. Speed Layer
2. Batch Layer
3. Serving Layer

Speed layer is where the streaming computation happens. I will be using [Spark Streaming](https://spark.apache.org/streaming/) for this and will be storing the results of this computation as RDDs in memory.

Batch layer is where the periodic batch computations happen and I will be using Spark for this and storing the results of this computation into [Parquet](https://parquet.apache.org/) which is an excellent columnar file store.

For Serving I will be using Scalatra. I will create a Spark context in Scalatra server and use that to satisfy REST queries for data.

I created a simple script to generate fake data. The data rows are in JSON format, following is a sample record from this fake data
{%highlight javascript linenos%} 
{
  "created": 1433262971414,
  "url": "http://www.example.com/id=1",
  "country": "US",
  "browser": "FF"
}
{%endhighlight%}

Following is the program to generate the random data 

{%highlight javascript linenos%} 
import org.json4s._
import org.json4s.jackson._

import scala.util.Random

class SparkLambda {
}

/**
{
  "created": 1433262971414,
  "url": "http://www.example.com/id=1",
  "country": "US",
  "browser": "FF"
}
*/
case class LogLine(created: Long, url: String, country: String, browser: String)

object SparkLambda extends App {
  implicit val formats = DefaultFormats

  def genFakeData(numRows: Int): Unit = {
    def randomUrl: String = {
      val id = Random.nextInt(100)
      s"http://www.example.com/id=$id"
    }

    var startTime: Long = 0

    for (i <- 0 to numRows) {
      // Get random url
      // Get random country
      // Get random browser
    }
  }
}
{%endhighlight%}
