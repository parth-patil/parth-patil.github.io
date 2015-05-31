---
layout: post
title: "Notes on Composing Scala Futures and Handling Errors"
comments: true
tags: [scala, async, future]
---

Note that this by no means an exhaustive list of ways in which Futures can be combined and how Error handling should be done when using Scala Futures. But I will cover some of the useful ones that I have encountered and use in my code.

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

This way we have composed two Future returning functions in a series. Note that we have to use flatMap instead of map else we will end up with Future[Future[Product]]. Also if you want to compose a lot of Future returning functions then using the flatMap straight up could make the code look ugly and lead to so called `callback hell`. Following example demonstrates composing multiple future returning functions

{%highlight scala linenos%} 
val pAndI: Future[(Product, Inventory)] = for {
  product   <- getTopProduct
  inventory <- inventoryService.getInventory(product.id)
} yield {
  (product, inventory)
}
{%endhighlight%}

### 

* composing with map
* composing with flatMap
* Converting Seq[Future[T]] to Future[Seq[T]]
* using recover & recoverWith
* using fallbackTo
* Future.fromTry
* andThen (multiple partial functions are executed in order so if you want to execute callbacks in order use this) 
*  