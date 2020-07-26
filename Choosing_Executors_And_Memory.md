How to choose number of executors and memory for a spark cluster:

Resources we have:
6 machines
16 cores  per machine
64 GB RAM/machine

Our options:
1. Smallest Size Executors?
2. Biggest Size Executors?
3. Right Approach

So per machine we have 64 GB RAM which means each core can get 64/16= 4 GB RAM
So total there are 16*6=96 cores
And each core has 4 GB RAM

1. If we take smallest size executors we can have 1 executor per core of 4GB RAM. So we will end up with 16 executors per machine.
But we are not truly leveraging the parallel threading capacity of the machines in this approach. Because we are giving only 1 core per executor.

2. If we can have 1 executor per machine so total we can have 6 executors of 64 GB RAM. Each executor is of 64 GB RAM or 16 cores per executor.
Problems:
  IO Contention(Since we are giving 16 cores per executor and all the executor cores will try to read their own partition from HDFS
    there will be lot of HDFS IO contention. Reads/writes will become slow)
  No Resources for OS (no memory left for OS tasks like memory clean up etc. the whole machine will slow down)
  No memory overhead for Yarn (off heap memory is required for spark and yarn else there will be hit on performance)

3. Correct Approach
Since we have total of 96 cores:
Set aside 1 core on each machine for OS processes(hadoop/Yarn daemon cores). 96 -6 = 90
So that leaves us with 90 cores
So 90/6 = 15 cores per machine is left for our spark jobs
Now the optimal number of executors per core recommended is 5 cores per executor to ensure there is not too much of IO contention.
So on every machine we can have 15/5 = 3 Executors
So total of 6*3 = 18 executors with 5 cores per executor is possible

Memory required for executor:
1. Per machine we have 64 GB RAM but 1 GB we need to set aside for OS processes. So we have 63 GB per machine.
2. 63 GB / 3 = 21 GB per executor
3. Yarn overhead is 2 GB for every executor( default is max of (384MB, approximately 7% of memory per executor) whichever is the biggest between the two)
This yarn overhead is for off heap memory(things like direct buffers)
4. So ideally 19 GB per executor is available

So in the end we have total of 18 executors(5 cores per executor) with each executor having 19 GB memory.

So one executor will be for spark driver.

Refer article: https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/
