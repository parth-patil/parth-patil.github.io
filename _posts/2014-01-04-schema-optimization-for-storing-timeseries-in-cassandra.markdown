---
layout: post
title: "Schema optimization for storing timeseries in Cassandra"
date: 2014-01-04 18:02
comments: true
tags: [cassandra, time-series, analytics]
---

I have been experimenting with Cassandra for storing timeseries data and I have so far found it to be an excellent fit for this use case. There are numerous articles on how to design your schema for storing timeseries data but I just wanted to focus on a schema optimization that helped me to get decent reduction in disk space used.

For the test I did 100k inserts into the table. After the insert I flushed and compacted the namespace to make sure that all data is flushed from memtables onto disk and also all the data files for the table are compacted into one file. The default compaction strategy (Size tiered compaction) will yiled a single data file after compaction. Following are the steps to flush and compact the tables in a namespace.

{%highlight bash %}
# test2 is my namespace
# flush
nodetool flush test2 

# compact
nodetool compact test2
{%endhighlight%}

Following are the steps I took to successively refine the schema to get increased space saving at each step.

Following is the v1 of the schema as defined in CQL3
{% highlight sql %}
CREATE TABLE tseries (
  name text,
  day int,
  count double,
  PRIMARY KEY (name, day));
{% endhighlight %}

I am encoding the day in format YYYYMMDD and that should fit in an integer.

To view how the data is layed out on disk I used sstable2json utility that ships with Cassandra. Following is how it looks. I am showing only the first few lines from the output.
{%highlight json %}
[
  {
    "key": "61637469766974696573",
    "columns": [
      ["17000101:", "", 1388892730835000],
      ["17000101:count", "241.0", 1388892730835000],
      ["17000102:", "", 1388892730835000],
      ["17000102:count", "934.0", 1388892730835000],
      ["17000103:", "", 1388892730835000],
      ["17000103:count", "628.0", 1388892730835000],
      ...
    ]
  }
]
{%endhighlight%} 

Following is the command to generate the json output above
{%highlight bash %}
sudo bin/sstable2json /var/lib/cassandra/data/test2/tseries/test2-tseries-jb-14-Data.db | underscore print --color
{%endhighlight%}
I used the [underscore](https://github.com/ddopson/underscore-cli) library to pretty print the JSON

The amount of disk space that the sstable data file took was about 1.5 MB (1529196 bytes)

Now if you look at the JSON output from the sstable2json tool you will notice that the column name "count" is repeated again and again for each column value. I wanted to see if there would be any space saving if I chose a smaller column name for storing the count. So I dropped the table and recreated with column name for storing count as "c" instead of "count". Note that you can't do alter table to rename the column from "count" to "c" on the existing table because the column is not part of the primary key.

I reinserted the 100k rows and did the flush and compact again using the nodetool. Now lets see how the data stored on disk looks like using the sstable2json tool.

{%highlight json %}
[
  {
    "key": "61637469766974696573",
    "columns": [
      ["17000101:", "", 1388895012852000],
      ["17000101:c", "972.0", 1388895012852000],
      ["17000102:", "", 1388895012852000],
      ["17000102:c", "720.0", 1388895012852000],
      ["17000103:", "", 1388895012852000],
      ["17000103:c", "396.0", 1388895012852000],
      ...
    ]
  }
]
{%endhighlight%}

Now as expected instead of "count" you see "c" being repeated for each column. The amount of disk space consumed with this schema is about 1.4 MB (1491105 bytes). So there is a saving of 38091 bytes, approx 37 kb. Which is saving of about 2%. Its not much but there is some saving from choosing smaller column names.

Now if you look closely at the JSON representation of the sstable on disk you will notice that there are two entries corresponding to each column eg.

{%highlight json %}
["17000101:", "", 1388895012852000],
["17000101:c", "972.0", 1388895012852000],
{%endhighlight%}

One for the day part of the column (i.e 1700/01/01) and one for the count "c". We can optimize the schema further by storing the value for the count in the column name itself. And the column value can be left as empty.

This can be done via a slight change of the table schema. We change the primary key for the table so that count is also part of the primary key. Following is the new and improved schema.

{% highlight sql %}
CREATE TABLE tseries (
  name text,
  day int,
  count double,
  PRIMARY KEY (name, day, count));
{% endhighlight %}

I reinserted the 100k rows. Did the flush and compact. Now lets see how the data stored on disk looks like.

{%highlight json %} 
[
  {
    "key": "61637469766974696573",
    "columns": [
      ["17000101:456.0:", "", 1388897328568000],
      ["17000102:885.0:", "", 1388897328568000],
      ["17000103:879.0:", "", 1388897328568000],
      ["17000104:326.0:", "", 1388897328568000],
      ["17000105:486.0:", "", 1388897328568000],
      ["17000106:140.0:", "", 1388897328568000],
      ...
    ]
  }
]
{% endhighlight %}

As you can see the value for count has been stored in the column name itself and the column value is empty. No space is wasted to encode an empty column value. The amount of disk space for the table using this schema is 883 Kb (904369 bytes). Now that is a decent amount of saving compared to the 1st and the 2nd version of our schema. Its a 40% saving in space. Also note that using a longer column name ("count" vs "c") for storing count has no effect on the disk space consumed as the column "count" is not stored in the composite column name.

Keep these tricks in mind when storing data in Cassandra via CQL3.

