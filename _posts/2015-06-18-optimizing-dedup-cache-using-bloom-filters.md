---
layout: post
title: "Optimizing dedupe cache using bloom filters"
comments: true
tags: [scala, bloom-filter, data-structures, probablistic-data-structures]
---
I have been involved in an early stage product where in we recommend urls to users and we don't want to show duplicate recommendations. So we want to store a "seen" cache per user that stores the urls that the user has already been recommended.

The straightforward thing to do is to store the seen cache on the backend in the form of a set that is keyed off of the userId. This cache could live in redis or memcache. But we are doing a proof of concept for our idea and didn't want to put in too much time in backend work upfront. So we decided to store the seen cache on the browser side. 

First option we thought of was to store the seen cache in the `Local Storage` of the browser and that would have allowed us to store a lot of data on the client side. But this would mean that the browser would have to send the seen cache with every request for recommendations. We wanted to make the seen cache transparent to the client and in the future when we would build the distributed server side seen cache the client would not have to change. To that end we decided to store the seen cache in the browser cookie. The cookie travels back and forth between the server and client without the client having to do anything special.

But as you might have guessed when using browser cookie you get a very limited amount of storage per domain. As per the speck its 4k bytes per domain but can vary depending on the browser. Some browsers will allow bigger cookies. But at any rate its a very limited amount of storage given that you have to share this storage with other cookies in our domain.

Our urlIds are MD5 of the full canonical url. In string format it ends up being 32 character string (i.e 32 bytes). 

My first attempt was to store the seen cache as concatenated md5 hashes of the urls. This approach though simple was not very efficient. We wanted to store atleast 100 urls in the seen cache. To store 100 urls we would need 3200 bytes. That would leave only ~800 bytes for all other cookies on our domain and that was not acceptable.

So after some thinking I realized that I needed a probablistic datastructure for checking set membership and that would allow me to configure the space vs accuracy tradeoff. For the seen cache we could tolerate false positives (i.e seen cache would tell that a url has been seen before when in fact it was not) but not false negatives. So for this use case the `Bloom Filter` fit the bill pretty well.

So in the backend I decided to use the Bloom Filter implemented in Google Guava. The high level idea is to set the urlIds in the Bloom Filter and then get the underlying bit vector of the bloom filter, base64 encode it and then set it in the cookie. When the client sends back the cookie in the request we can follow a reverse process to reconstruct the Bloom Filter.

Following is what the code looks like

{%highlight scala linenos%}
import org.apache.commons.codec.binary.Base64
import java.security.MessageDigest
import com.google.common.hash.{Funnels, BloomFilter}
import java.io._

class MyBloomFilter(bf: BloomFilter[Array[Byte]]) {
  def this(numItems: Int, errorPercent: Double) = {
    this(BloomFilter.create(Funnels.byteArrayFunnel, numItems, errorPercent))
  }

  private var cardinality = 0
  val md5 = MessageDigest.getInstance("MD5")

  def genHash(str: String): Array[Byte] = md5.digest(str.getBytes)

  def size = cardinality

  def put(item: String): Unit = {
    bf.put(genHash(item))
    cardinality += 1
  }

  def mightContain(item: String): Boolean = bf.mightContain(genHash(item))

  def serializeToBase64(): String = {
    val bs = new ByteArrayOutputStream
    bf.writeTo(bs)
    val bfByteArray = bs.toByteArray
    bs.close()
    Base64.encodeBase64String(bfByteArray)
  }
}

object MyBloomFilter {
  def fromBase64(serialized: String): MyBloomFilter = {
    val bfByteArray = Base64.decodeBase64(serialized)
    val inputStream = new ByteArrayInputStream(bfByteArray)
    val bf = BloomFilter.readFrom(inputStream, Funnels.byteArrayFunnel)
    new MyBloomFilter(bf)
  }
}
{%endhighlight%}

Following is example of how you would use it

{%highlight scala linenos%}
object BloomFilterTest extends App {
  val myBloom = new MyBloomFilter(10, 0.01)
  myBloom.put("http://www.google.com")
  myBloom.put("http://www.yahoo.com")

  val encodedStr = myBloom.serializeToBase64()
  
  // Recreate the Bloom Filter from the Base64 encoded string
  val myBloom2 = MyBloomFilter.fromBase64(encodedStr)

  println(s"google exits -> " + myBloom2.mightContain("http://www.google.com"))
  println(s"yahoo exits -> " + myBloom2.mightContain("http://www.yahoo.com"))
  println(s"microsoft exits -> " + myBloom2.mightContain("http://www.microsoft.com"))
}
{%endhighlight%}

Following is what I get when I run it

{%highlight bash linenos%}
google exits -> true
yahoo exits -> true
microsoft exits -> false
{%endhighlight%}

This technique resulted in a huge saving in space. The original naive technique used would have required 32 * 100 = 3200 bytes. This new technique yields a base64 encoded string of just 168 bytes. Thats ~95% reduction in space!
