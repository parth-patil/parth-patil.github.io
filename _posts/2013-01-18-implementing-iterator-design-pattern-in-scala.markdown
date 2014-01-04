---
layout: post
title: "Implementing Iterator Design Pattern in Scala"
date: 2013-01-18 22:32
comments: true
tags: [scala, design-patterns]
---
The other day I came across a puzzle that involved implementing a spreadsheet class. Like a
regular spreadsheet the cells were referenced as A1, A2 etc .. I wanted to iterate the cells
in the spreadsheet. My first stab at the implementation looked as follows

Spreadsheet iteration v1

{% highlight scala %}
import scala.collection.mutable

object IteratorTest {
  def main(args: Array[String]) {
    val height = 4
    val width = 4
    val ss = new SpreadsheetV1(width, height)
    for (i <- 0 until height) {
      for (j <- 0 until width) {
        val id = Spreadsheet.getIdFromRowAndCol(i, j)
        println("id = " + id)
      }
    }
  }
}

class SpreadsheetV1(width: Int, height: Int) {
  val data = Array.ofDim[String](height, width)
  def get(i:Int, j:Int) = data(i)(j)
  def set(i:Int, j:Int, value: String) { data(i)(j) = value }
}

object Spreadsheet {
  val Coordinates = """([A-Z])(\d+)""".r
  val asciiValueForA = 'A'.toInt

  /**
   * Given a cell id retuns the row and col as tuple
   * eg. A2 -> (0, 1)
   */
  def getRowAndColFromId(id: String): (Int, Int) = {
    val Coordinates(row: String, col: String) = id // apply the regex to extract row & col

    // Now convert these to actual Int indexes in our 2d array
    val realRow = row.toCharArray()(0).toInt - asciiValueForA
    val realCol = col.toInt -1
    (realRow, realCol)
  }

  /**
   * Given row and col returns id
   * eg (0,0) => A1, (0,1) => A2 etc
   */
  def getIdFromRowAndCol(row: Int, col: Int): String = {
    (asciiValueForA + row).toChar + (col + 1).toString
  }
}
{% endhighlight %}

But its possible to have a nicer interface to iterate over the cells by implementing Iterator
trait. It also results in better encapsulation of the internal structure of the Spreadsheet
class. Following is an implementation of Spreadsheet class with Iterator trait.

Spreadsheet iteration v2

{% highlight scala %}
import scala.collection.mutable

object IteratorTest {
  def main(args: Array[String]) {
    val height = 4
    val width = 4
    val ss = new SpreadsheetV2(width, height)
    for (id <- ss)
      println(id)
  }
}

class SpreadsheetV2(width: Int, height: Int) extends Iterator[String] {
  import Spreadsheet._

  val data = Array.ofDim[String](height, width)

  def get(id: String): String = {
    val (row, col) = getRowAndColFromId(id)
    data(row)(col)
  }

  def set(id: String, value: String) {
    val (row, col) = getRowAndColFromId(id)
    data(row)(col)  = value
  }

  //------------------------
  // Iterator Implementation
  //------------------------
  protected var _nextRow: Int = 0
  protected var _nextCol: Int = 0
  protected var _next: String = null

  def hasNext = {
    if (_next != null)
      true
    else
      computeNext
  }

  def next = {
    if (hasNext) {
      val result = _next
      _next = null
      result
    } else {
      throw new NoSuchElementException
    }
  }

  protected def computeNext: Boolean = {
    if (_nextRow < height && _nextCol < width) {
      _next = getIdFromRowAndCol(_nextRow, _nextCol)

      if (_nextCol == (width -1)) {
        _nextCol = 0
        _nextRow += 1
      } else {
        _nextCol += 1
      }

      true
    } else {
      false
    }
  }
}
{% endhighlight %}

The iterator code is not a core part of the Spreadsheet's logic so there is no need for the
Spreadsheet class to directly implement Iterator. The iterator implementation can be moved out
into a separate class and the Spreadsheet class's implementation can be kept simple.

Spreadsheet iteration v3

{% highlight scala %}
import scala.collection.mutable

object IteratorTest {
  def main(args: Array[String]) {
    val height = 4
    val width = 4
    val ss = new SpreadsheetV3(width, height)
    for (id <- ss.getIterator)
      println(id)
  }
}

class SpreadsheetV3(width: Int, height: Int) {
  import Spreadsheet._

  val data = Array.ofDim[String](height, width)

  def get(id: String): String = {
    val (row, col) = getRowAndColFromId(id)
    data(row)(col)
  }

  def set(id: String, value: String) {
    val (row, col) = getRowAndColFromId(id)
    data(row)(col)  = value
  }

  def getIterator(): Iterator[String] = {
    new SpreadsheetIterator(width, height)
  }
}

class SpreadsheetIterator(width: Int, height: Int) extends Iterator [String] {
  import Spreadsheet._

  //------------------------
  // Iterator Implementation
  //------------------------
  protected var _nextRow: Int = 0
  protected var _nextCol: Int = 0
  protected var _next: String = null

  def hasNext = {
    if (_next != null)
      true
    else
      computeNext
  }

  def next = {
    if (hasNext) {
      val result = _next
      _next = null
      result
    } else {
      throw new NoSuchElementException
    }
  }

  protected def computeNext: Boolean = {
    if (_nextRow < height && _nextCol < width) {
      _next = getIdFromRowAndCol(_nextRow, _nextCol)

      if (_nextCol == (width -1)) {
        _nextCol = 0
        _nextRow += 1
      } else {
        _nextCol += 1
      }

      true
    } else {
      false
    }
  }
}
{% endhighlight %}
