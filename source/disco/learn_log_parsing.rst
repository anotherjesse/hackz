Lesson 1: Web Log Parsing
=========================

To learn how Disco and MapReduce works, I've started with trivial log parsing, counting the number of requests by an IP.

Import logs into DDFS
---------------------

Start by copying the logs to the system, then split them into 10,000 line chunks and import them into DDFS.

.. code-block:: bash

 mkdir logs
 split access.log logs/access-
 ddfs push -r data:logs logs

Running MapReduce
-----------------

.. code-block:: python

 import sys
 from disco.core import Disco, result_iterator
 
 def fun_map(line, params):
     yield line.split(' ')[0], 1
 
 def fun_reduce(iter, out, params):
     stats = {}
     for word, count in iter:
         stats[word] = stats.get(word, 0) + int(count)
     for word, total in stats.iteritems():
             out.add(word, total)
 
 results = Disco('disco://i-0cyqw0bz').new_job(
             name='access',
             input=['tag://data:logs'],
             map=fun_map, save=False,
             reduce=fun_reduce).wait()

Using Results
-------------

Since we want to see the top 20 visitors by IP, we have to do some work.

Printing all results
++++++++++++++++++++

The simplest thing to do is to iterate through all results, printing the count.

.. code-block:: python

 for ip, count in result_iterator(results):
     print ip, count

Printing top 20 IPs
+++++++++++++++++++

The previous method printed the results unordered, which means you have to do post processing.

.. code-block:: python

 class Top(object):
     def __init__(self, max_val=5):
         self.max_count = max_val
         self.len = 0
         self.db = []
 
     def add(self, k, val):
         v = int(val)
         if self.len < self.max_count or self.db[0][1] < v:
             for idx in xrange(0, self.len):
                 if v < self.db[idx][1]:
                     self.db.insert(idx, [k,v])
                     if self.len < self.max_count:
                         self.len += 1
                     else:
                         self.db.pop(0)
                     return
             self.db.append([k,v])
             if self.len < self.max_count:
                 self.len += 1
             else:
                 self.db.pop(0)
 
     def report(self):
         db = list(self.db)
         db.reverse()
         for k,v in db:
             print "%-20s %8d" % (k,v)
 
 top = Top(20)
 for ip, count in result_iterator(results):
     top.add(ip, count)
 
 top.report()

Now we have our results.

::

 67.195.112.232          97896
 60.48.137.9             52456
 118.137.142.13          43010
 94.253.154.220          37453
 60.51.112.214           35632
 66.249.71.24            34523
 218.186.10.242          33827
 79.19.86.199            33351
 95.108.244.252          31992
 67.60.39.113            31962
 125.167.207.149         27811
 38.99.96.119            24603
 60.48.81.39             22376
 142.151.156.126         20950
 96.247.51.155           20184
 119.237.154.188         19813
 140.136.150.80          19800
 218.186.13.251          19767
 72.14.199.120           19664
 58.26.217.95            19475

