---
layout: post
title: How I fixed logging
---

At my current place I'm supporting a software solution that's been in constant development since 2001 and was touched by a swarm of developers who seemed to be very good at ignoring each other.
The result was that at some unknown point in the past one of the microservices stopped showing any logging messages above `ERROR`. Nobody knows when it started, or what might have caused it.
As everyone can imagine supporting a complex legacy java app is hard, supporting a complex legacy app without any logging is also hard.

So here are the three steps that were necessary to resolve the issue.

## Check the logging chains

Just google _java logging mess_, to get some background. It's not an overly complex issue to resolve, it just requires some dilligence  when adding new project dependencies to your project.
So the first thing, analyze your dependencies and see that you have a sensible logging architecture. The magic command is

    mvn dependency:tree -Dverbose -Dincludes=....

I specifically looked for everything that used artifacts from the following groups `log4j`, `org.slf4j`, `commons-logging` and `org.apache.logging.log4j` and made sure the whole logging went through `log4j`.

## Enable debug logging to see as much as possible

Still nothing? Well, at this point I was sure there was some misconfiguration somewhere, but I couldn't see where because there were no logging. So the next thing that I've done was setting the logging level to `DEBUG` programmatically.
Add the following as close to your `main` as possible

{% highlight java %}
Logger.getRootLogger().setLevel(Level.DEBUG);
{% endhighlight %}

## Watch closely

After restart I've noticed a lot of messages that looked like

    2015-11-06 22:42:41,451 | INFO | ...
     2015-11-06 22:42:41,451 | INFO | ...
     2015-11-06 22:42:41,451 | INFO | ...
     2015-11-06 22:42:41,479 | INFO | ...

Note the space at the beginning of the line. I looked for '%n ' in the project and library files and sure enough found a 'rougue' `log4j.xml` pulled into my web project from some unrelated java desktop application
and getting a priority over `log4j.properties` from the web project.
