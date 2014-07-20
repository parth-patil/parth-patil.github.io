---
layout: post
title: "Scala Pattern matching makes code more readable"
date: 2014-07-19 20:17
comments: true
tags: [scala, pattern-matching]
---

Recently I was working on a scala project that required parsing text to create case classes that can be treated as commands that get passed to other subsystems. The following string commands had to be mapped into cases classes.

```
SET foo bar
GET foo
BEGIN
COMMIT
ROLLBACK
END
```

Following are the case classes the commands needed to be mapped to

{% highlight scala %}
trait Command

case class SetCommand(key: String, value: String) extends Command
case class GetCommand(key: String) extends Command
case class BeginCommand() extends Command
case class CommitCommand() extends Command
case class RollbackCommand() extends Command
case class EndCommand() extends Command
{% endhighlight %}

Initially I implemented the parsing logic using `if else` construct and this is what the code looked like

{%highlight scala%}
def parse(rawCommand: String): Option[Command] = {
  val pieces: Seq[String] = rawCommand.trim.split(" ")
  if (pieces.size == 1) {
      if (pieces(0) == "BEGIN")  Some(BeginCommand())
      if (pieces(0) == "COMMIT")  Some(CommitCommand())
      if (pieces(0) == "ROLLBACK")  Some(RollbackCommand())
      if (pieces(0) == "END")  Some(EndCommand())
      else None
  } else if (pieces.size == 3 && pieces(0) == "SET") {
    Some(SetCommand(pieces(1), pieces(2)))
  } else if (pieces.size == 2 && pieces(0) == "GET") {
    Some(GetCommand(pieces(1)))
  } else {
    None
  }
}
{%endhighlight%}

After reimplementing the parsing logic using pattern matching the code is more compact and easier to understand (IMHO)

{%highlight scala%}
def parse(rawCommand: String): Option[Command] =
  rawCommand.trim.split(" ") match {
    case Seq("BEGIN")           => Some(BeginCommand())
    case Seq("COMMIT")          => Some(CommitCommand())
    case Seq("ROLLBACK")        => Some(RollbackCommand())
    case Seq("END")             => Some(EndCommand())
    case Seq("SET", key, value) => Some(SetCommand(key, value))
    case Seq("GET", key)        => Some(GetCommand(key))
    case _                      => None
  }
{%endhighlight%}

Pattern matching FTW!