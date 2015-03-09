---
layout: post
title:  "Leveraging MapReduce for log files part 2: Hadoop Streaming"
date:   2015-03-07 18:12:44
categories: cli
---
See [part 1]({% post_url 2015-03-03-map-reduce-1-sort-uniq-sort %}) for a simple non-Hadoop MapReduce tutorial using `sort | uniq -c | sort -n` and `awk`.

Now that we have explored various methods of extracting text from log files and reducing it down to simple counts of each unique item, let's explore another method: splitting the extraction of text and counting into two separate scripts. That might seem less efficient at first glance, but it breaks the job up in what Hadoop needs. We can accomplish this using any program that accepts STDIN and prints to STDOUT. Let's write these in both Python and bash.

We want to write a mapper and reducer that works when called as:
{% highlight bash %}
cat haproxy.log | ./mapper.py | sort | ./reducer.py
{% endhighlight %}

Mapper
------

The mapper should print tab-separated key-value pairs[1]. So let's implement, extracting the 8th column from the haproxy.log (the haproxy service name)

### mapper.sh

{% highlight bash %}
#!/bin/bash
awk '{ print $8 "\t1" }' 
{% endhighlight %}

### mapper.py

{% highlight python %}
#!/usr/bin/env python
import sys
for line in sys.stdin:
  print( "%s\t1" % line.split()[7])
{% endhighlight %}

Take a moment to run the a bit of log into the mapper. Here's a 3-line snippet below:

{% highlight bash %}
$ head -n 3 haproxy.log
Mar  3 03:06:01 server-name haproxy[29132]: 10.11.22.33:37983 [03/Mar/2015:03:06:01.215] cool-api cool-api/api22 42/0/0/14/56 200 205 - - ---- 761/191/34/18/0 0/0 "POST /neato/endpoint HTTP/1.1"
Mar  3 03:06:01 server-name haproxy[29132]: 10.11.22.33:37993 [03/Mar/2015:03:06:01.222] some-other-api some-other-api/api14 25/0/0/12/37 200 205 - - ---- 761/191/34/18/0 0/0 "POST /neato/endpoint HTTP/1.1"
Mar  3 03:06:01 server-name haproxy[29132]: 10.11.22.33:37983 [03/Mar/2015:03:06:01.226] cool-api cool-api/api03 31/0/0/14/56 200 205 - - ---- 761/191/34/18/0 0/0 "POST /neato/endpoint HTTP/1.1"
{% endhighlight %}

Should render this:
{% highlight bash %}
$ head -n 3 haproxy.log | ./mapper.py
cool-api	1
some-other-api	1
cool-api	1
{% endhighlight %}

Reducer
-------
The example above, as with a Hadoop MapReduce job, the sorting/combining of keys is part of the work that the is performed automatically (by `sort` in the above example, or by the JobTracker when run on a Hadoop cluster). Now we are left with the task of taking sorted output of key-value pairs and emitting what we are interested in.

Note that while the number emitted in the above mapper is always "1" and it would seem we could do away with actually extracting that number, by implementing the process to emit and parse the number sets us up for a massive performance gain later, when we get this working in Hadoop. We will be able to use the reducer as a "combiner", using the exact same code.

### reducer.sh
{% highlight bash %}
#!/bin/bash -e

CURRENT_KEY=""
CURRENT_COUNT=0

while read line; do
  IFS=$'\t' read KEY COUNT <<< "$line"
  if [ "$KEY" = "$CURRENT_KEY" ]; then
    CURRENT_COUNT=$(( $CURRENT_COUNT + $COUNT ))
  else
    [ -n "$CURRENT_KEY" ] && printf "%s\t%s\n" $CURRENT_KEY $CURRENT_COUNT
    CURRENT_KEY=$KEY
    CURRENT_COUNT=$COUNT
  fi
done < "${1:-/dev/stdin}"

printf "%s\t%s\n" $CURRENT_KEY $CURRENT_COUNT
{% endhighlight %}


### reducer.py
{% highlight python %}
#!/usr/bin/env python
import sys

# Define here to be able to use outside loop
current_key = None
current_count = 0

for line in sys.stdin:
  key,val = line.strip().split('\t', 1)

  # TODO: You may want to add try/catch here, although failing on parse error is often desireable
  count = int(val)

  if current_key == key:
    current_count += count
  else:
    # First iteration will be None
    if current_key:
      print( "%s\t%s" % (current_key, current_count ) )
    current_key = key
    current_count = count

# Last iteration
print( "%s\t%s" % (current_key, current_count ) )
{% endhighlight %}

Performance
-----------
Alright, let's try the many versions now!

Conclusion
----------
We've split up the task of extracting the data we want from the task of counting up the unique occurences into separate script. We can take these two scripts and "fake" a MapReduce run by invoking them as:

`cat __INTPUT_FILE__ | mapper.py | sort | reducer.py`

In our next entry, we'll take what we've written here and leverage Hadoop HDFS and MapReduce to process a much larger log file in a fraction of the time.

References
----------
 - (Apache Hadoop Streaming Docs)[http://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/HadoopStreaming.html]
