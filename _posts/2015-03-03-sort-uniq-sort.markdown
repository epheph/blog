---
layout: post
title:  "sort | uniq -c | sort -n : reducing your data to counts"
date:   2015-03-03 18:12:44
categories: cli
---
One of the more powerful command-line patterns, and one that I find myself using daily:

{% highlight bash %}
$ __SOME_COMMAND__ | sort | uniq -c | sort -n
{% endhighlight %}

This takes an input of values, and outputs a list sorted by the frequency of each item's appearance. Let's see a basic example before diving into some more advanced ones and considering performance implications:

{% highlight bash %}
$ head -n 1 haproxy.log
Mar  3 03:06:01 server-name haproxy[29132]: 10.11.22.33:37983 [03/Mar/2015:03:06:01.215] cool-api cool-api/api22 42/0/0/14/56 200 205 - - ---- 761/191/34/18/0 0/0 "POST /neato/endpoint HTTP/1.1"
Mar  3 03:06:01 server-name haproxy[29132]: 10.11.22.33:37993 [03/Mar/2015:03:06:01.222] some-other-api some-other-api/api14 25/0/0/12/37 200 205 - - ---- 761/191/34/18/0 0/0 "POST /neato/endpoint HTTP/1.1"

$ head -n 10 haproxy.log  | awk '{ print $8 }'
cool-api
some-other-api
hihi-api
some-other-api
cool-api
cool-api
some-other-api
hihi-api
some-other-api
hihi-api

$ head -n 1000 haproxy.log  | awk '{ print $8 }' | sort | uniq -c | sort -n
     13 integration-api
     14 batch-api
     26 developer-api
     53 bigdata-api
    175 hihi-api
    354 some-other-api
    365 cool-api
$
{% endhighlight %}

We have quickly determined, from a small sample of the haproxy log, the distribution of requests to different haproxy services.

You may not want the final `sort -n` if the natural order of the text is more consumable. Take, for instance, extracting minutes from a log file:

With "grep -o"
--------------------------
{% highlight bash %}
$ grep -o --extended-regexp ^.{12} haproxy.log | sort | uniq -c
  80387 Mar  4 03:24
  82871 Mar  4 03:25
  82657 Mar  4 03:26
  83519 Mar  4 03:27
  83191 Mar  4 03:28
  83815 Mar  4 03:29
  81952 Mar  4 03:30
  81850 Mar  4 03:31
  82250 Mar  4 03:32
  81781 Mar  4 03:33
  80342 Mar  4 03:34
  80818 Mar  4 03:35
  80565 Mar  4 03:36
  79868 Mar  4 03:37
  79540 Mar  4 03:38
  80111 Mar  4 03:39
  80141 Mar  4 03:40
  79224 Mar  4 03:41
  79434 Mar  4 03:42
  79633 Mar  4 03:43
  79032 Mar  4 03:44
  78831 Mar  4 03:45
  78371 Mar  4 03:46
  78939 Mar  4 03:47
...
{% endhighlight %}

 - `grep -o` returns only the text that matches instead of the whole line. The expression "^.{12}" matches the first 12 characters of any line (requires the "--extended-regexp", which here grabs the date, hour, and minute
 - Why do we throw in a `sort` even though the log file is naturally time ordered? It is possible in high traffic environments to end up with times out of order if the server is logging the request START time vs. the request END time.
 - Adding `sort -n` would be useful to find the busiest or least busy minutes.

With "tcpdump -A" / "ngrep -Wbyline"
-----------------
With a little knowledge of an unencrypted network protocol like memcache, you can gather some interesting data about access patterns. Both `tcpdump -A` and `ngrep -Wbyline` dump raw packets with a pcap filter.

ngrep is better here, given that you can more easily anchor your regex's to the beginning of the line, but the idea is extremely similar. I'll give both examples:

### tcpdump -A
{% highlight bash %}
[scottb@api1-north ~]$ sudo tcpdump -c 100000 port 11211 -A  | grep -o "get .*" | sort | uniq -c | sort -n | tail
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on bond0, link-type EN10MB (Ethernet), capture size 65535 bytes
100000 packets captured
100397 packets received by filter
355 packets dropped by kernel
    269 get mint:analyze:etl:5310
    275 get mint:analyze:detail:ade4359efb4ec412ca9fa5c3f647106ed
    290 get content:detail:23192
    354 get mint:analyze:detail:397bba35bde44c6ebdetle7299e30f9b4
    378 get mint:analyze:etl:154506
    396 get mint:analyze:detail:1756bdd3a5f24498b166f776974cade3
    407 get mint:analyze:etl:74814
    707 get mint:analyze:etl:129284
    710 get mint:analyze:content:939112
    820 get mint:analyze:etl:361970
{% endhighlight %}

### ngrep -Wbyline
{% highlight bash %}
[scottb@api1-north ~]$ sudo ngrep port 11211 -Wbyline -n 100000 |  grep -o "^get .*" | sort | uniq -c | sort -n | tail
    147 get content:detail:23192.
    151 get mint:analyze:detail:ade4359efb4ec412ca9fa5c3f647106ed.
    187 get mint:analyze:detail:397bba35bde44c6ebdetle7299e30f9b4.
    209 get mint:analyze:detail:1756bdd3a5f24498b166f776974cade3.
    227 get mint:analyze:etl:154506.
    241 get content:detail:19288.
    298 get mint:analyze:etl:74814.
    363 get mint:analyze:etl:129284.
    416 get mint:analyze:content:939112.
    473 get mint:analyze:etl:361970.
{% endhighlight %}
- With ngrep, we have added "^" to regexp, making mismatches within actual protocol text less likely

Performance Considerations
--------------------------
There are a few some considerable inefficiencies with this command-line pattern. Aside from instantiating 3 processes, each of them reading, writing, and parsing each bit of text multiple times, there's a much more important one: the initial `sort` to allow us to run `uniq -c` is a complete waste, especially when we're going to `sort -n` the results anyway. When processing larger quantities of data, you might want to consider using `awk` directly to create a dictionary containing counts. 


Running with `sort | uniq -c | sort -n` takes ~37s
{% highlight bash %}
$ echo 3 | sudo tee /proc/sys/vm/drop_caches
3
$ time head -n 5000000 haproxy.log  | awk '{ print $8 }' | sort | uniq -c | sort -n
      2 magic-api
      3 rpc-api
     16 creative-api
     30 message-api
     81 soap-api
   1305 def-api
   1575 haproxy-stats
   4516 rest-api
  59937 encode-api
 110479 job-api
 211108 file-api
 337559 abc-api
1074764 unicorn-api
3198625 foo-api

real	0m36.840s
user	0m28.077s
sys	0m5.390s
{% endhighlight %}


Running entirely in awk takes ~21s:
{% highlight bash %}
$ echo 3 | sudo tee /proc/sys/vm/drop_caches
3
$ time head -n 5000000 haproxy.log   | awk '{ a[$8]++ } END { for(i in a) print a[i], "\t", i  }' | sort -n
2 	 magic-api
3 	 rpc-api
16 	 creative-api
30 	 message-api
81 	 soap-api
1305 	 def-api
1575 	 haproxy-stats
4516 	 rest-api
59937 	 encode-api
110479 	 job-api
211108 	 file-api
337559 	 abc-api
1074764 	 unicorn-api
3198625 	 foo-api

real	0m20.912s
user	0m13.151s
sys	0m5.017s
{% endhighlight %}

You might want to add this to your .bashrc:

{% highlight bash %}
alias reduce="awk '{ a[\$1]++ } END { for(i in a) print a[i], \"\\t\", i  }'"
{% endhighlight %}
 - Note the few extra slashes in there

Conclusion
----------
`sort | uniq -c` is a powerful way reduce a line-delimitted list of items into a count of each item's appearance. Appending `sort -n` orders the counts numerically, helping to identify the highest or lowest frequencies in any list. `grep -o` can be a powerful way to extract the data you need before passing it into `sort | uniq -c`. When dealing with large quanities of items, a native `awk` solution is often faster, as the initial `sort` in this pattern is not truly necessary.
