---
layout: post
title: "Notes on Composing Scala Futures and Handling Errors"
comments: true
tags: [scala, async, future]
---

In this post I will discuss some of the useful Futures combinators and ways to handle errors when working with Futures. Note that whenever I use the word `wait` it does not mean `block`. block means blocking a thread and wait in the following discussion refers to waiting for an async computation to finish without blocking a thread.

For this post I am going to use a small Ecommerce website example. Following is the API and the case classes for the domain objects. I am not going to discuss any concrete implementation of the following API, we only need to be concerned about using this API and play with Future transformation and composition.

{%highlight scala linenos%} 
case class Product(id: String, name: String)
case class ProductMetadata(productId: String, category: String, imageUrl: Option[String])
case class Inventory(productId: String, quantity: Long)
case class User(id: String, email: String)
case class ProductReview(
  id: String,
  productId: String,
  userId: String,
  created: Long,
  review: String,
  rating: Double)
case class ProductInfo(
  productId: String,
  meta: Option[ProductMetadata],
  inventory: Option[Inventory],
  reviews: Seq[ProductReview])

trait ProductService {
  def getProductListing(): Future[Seq[Product]]
  def getProductMetadata(productId: String): Future[ProductMetadata]
}

trait InventoryService {
  def getInventory(productId: String): Future[Inventory]
}

trait ProductReviewService {
  def getProductReviews(productId: String): Future[Seq[ProductReview]]
}
{%endhighlight%}

Now lets look at the various transformation and composition methods & techniqueues using Future combinator methods and also the [async](https://github.com/scala/async) macro

### Transforming with map
You can map a Future into another Future as follows

{%highlight scala linenos%} 
val inventoryService: InventoryService = ???
val inventory: Future[Inventory] = inventoryService.getInventory(productId = "prod-123")
val quantity: Future[Long] = inventory map { _.quantity }
{%endhighlight%}

### map equivalent using async and await
{%highlight scala linenos%} 
val inventoryService: InventoryService = ???
val fInventory: Future[Inventory] = inventoryService.getInventory(productId = "prod-123")
val quantity: Future[Long] = async { 
  val inv = await { fInventory.quantity }
  inv.quantity
}
{%endhighlight%}

### Composing Futures with flatMap
Lets see how to do serial composition of two Futures so that you get a pipeline. Lets say you want to 

{%highlight scala linenos%} 
val inventoryService: InventoryService = ???
def getTopProduct(): Future[Product] = ???

val product: Future[Product] = getTopProduct

// Lets get the inventory of the product above
val inventory: Future[Inventory] = product flatMap { p =>
  inventoryService.getInventory(p.id)
}
{%endhighlight%}

This way we have composed two Future returning functions in a series. Note that we have to use flatMap instead of map else we will end up with Future[Future[Product]]. Also if you want to compose a lot of Future returning functions then using the flatMap straight up could make the code look ugly and lead to so called `callback hell`. Using `for comprehension` helps in writing clean Future pipelines. Following example demonstrates composing multiple future returning functions using for comprehension.

{%highlight scala linenos%} 
val pAndI: Future[(Product, Inventory)] = for {
  product   <- getTopProduct
  inventory <- inventoryService.getInventory(product.id)
} yield {
  (product, inventory)
}
{%endhighlight%}

### flatMap equivalent using async & await
{%highlight scala linenos%} 
val pAndI: Future[(Product, Inventory)] = async {
  val product   = await { getTopProduct }
  val inventory = await { inventoryService.getInventory(product.id) }
  (product, inventory)
}
{%endhighlight%}
I do personally prefer the for comprehension over the async macro. Also the async macro has some [limitations](https://github.com/scala/async#limitations) that you should check out in case you run into problems. If you think the async macro makes the code easier to read, use it by all means.

### Sequential execution of Futures vs Parallel execution
The flatMap that we saw above executes the Futures in series. It waits for the getTopProduct() to complete before calling the getInventory() method. Lets now look at an example of executing Futures in parallel. Lets say we have a product and we want to get its metadata and inventory info in parallel.

{%highlight scala linenos%} 
val product: Product = ???
val futureMetadata: Future[ProductMetadata] = getProductMetadata(product.id)
val futureInventory: Future[Inventory] = getInventory(product.id)

val mAndI: Future[(ProductMetadata, Inventory)] = for {
  metadata  <- futureMetadata
  inventory <- futureInventory
} yield {
  (metadata, inventory)
}
{%endhighlight%}
The difference from before is that the calls that return the Futures are made outside the for comprehension so that they begin executing in parallel and inside the for comprehension we are waiting for the Futures to complete and then we yield the result tuple.

### Converting multiple parallel Futures into one Future
Sometimes when you spawn off multiple parallel Futures you want to convert all those Futures into a single Future on which you can put a single onComplete handler or combine it with yet other Futures. Lets say we have a list of product ids and we want to get inventory for each of the product ids in parallel and then print all the inventory information or print the exception message if the Future fails.

{%highlight scala linenos%} 
val productIds: Seq[String] = ???
val seqOfFutureInventories: Seq[Future[Inventory]] = productIds map getInventory
val futureOfInventories: Future[Seq[Inventory]] = Future.sequence(seqOfFutureInventories)
futureOfInventories onComplete {
  case Success(inventories) => inventories foreach println
  case Failure(e) => println(s"Error : ${e.getMessage}")
}
{%endhighlight%}

### Ordering completion handlers with andThen
You could provide multiple onComplete handlers on Futures but the order in which they get executed is not guaranteed. When you want to provide multiple onComplete handlers and want to enforce an ordering of the executions of those handlers you should use the `andThen` combinator on Future objects.

{%highlight scala linenos%} 
val product: Future[Product] = ???
product andThen {
  case Success(p) => // do something with the product
} andThen {
  case Success(p) => // do yet another thing with the product
  }
{%endhighlight%}

### Trans

### Error handling using recover and recoverWith

* using recover & recoverWith
* using fallbackTo
* Future.fromTry
* transform
* failed (failure projection)
