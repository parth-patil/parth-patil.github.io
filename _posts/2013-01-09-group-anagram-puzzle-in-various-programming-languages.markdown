---
layout: post
title: "Group Anagram puzzle in various programming languages"
date: 2013-01-09 22:13
comments: true
tags: [programming, puzzles, comparisons]
---

Java solution

{% highlight java %}
// Minus the boilerplate code
String[] words = {"abc", "bca", "mkzp", "cba"};

HashMap<String, ArrayList<String>> groups = new HashMap<String, ArrayList<String>>();

for (String w: words) {
  char[] chars = w.toCharArray();
  Arrays.sort(chars);
  String normalized = new String(chars);
  if (!groups.containsKey(normalized))
    groups.put(normalized, new ArrayList<String>());

  groups.get(normalized).add(w);
}
{% endhighlight %}

Scala solution

{% highlight scala %}
// Minus the boilerplate code
val words = Seq("abc", "bca", "mkzp", "cba")
words groupBy { word => word.toCharArray.sortWith(_ < _).mkString("") }
{% endhighlight %}

Coffeescript solution

{% highlight coffeescript %}
  words = ["abc", "bca", "mkzp", "cba"]
  result = {}
  for word in words
    key = (word.split '').sort().join ''
    result[key] = [] if not result[key]?
    result[key].push word
{% endhighlight %}

Ruby Solution

{% highlight ruby %}
words = ["abc", "bca", "mkzp", "cba"]
words.group_by { |w| w.chars.sort.join }
{% endhighlight %}
