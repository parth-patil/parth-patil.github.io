---
layout: post
title: "Custom functions in HBase shell"
date: 2013-01-10 09:04
comments: true
tags: [hbase, tips]
---
As many of you might know that the HBase shell is a jruby repl. So you can write ruby code in the
shell. You can also save your custom ruby functions in `~/.irbrc` and the next time you
start the hbase shell those functions will be available to you. At work I often need
to truncate a bunch of HBase tables before I can begin my testing. I wanted to automate
this. So I wrote the following custom function to truncate a list of tables and added
it to my `~/.irbrc`

{% highlight ruby %}
def truncate_tables()
  tables = [
    'table1',
    'table2',
    'table3'
  ]

  tables.each {|x| truncate x}
end
{% endhighlight %}

You have to restart your HBase shell for this function to be available in the shell.
Now this custom function can be invoked from the shell as follows
{% highlight bash %}
ruby-1.9.2-p136 :001 > truncate_tables
{% endhighlight %}

For more information refer to the following links:

[HBase Wiki](http://wiki.apache.org/hadoop/Hbase/Shell)

[Stack Overflow Question](http://stackoverflow.com/questions/7256100/how-to-scan-hbase-from-hbase-shell-using-filter)
