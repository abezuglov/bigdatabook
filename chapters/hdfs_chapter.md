# Hadoop Distributed File System

## Introduction

Distributed file systems and computation have been known prior to October 2003, yet this date can be considered the official public start of Big Data. In that month, Sanjay Ghemawat et al, published their paper on Google File System (GFS), where the authors shed light on how Google deals with efficient and reliable storing and processing of large files. The paper received significant interest and it was cited by 7,951 authors. Later, Sanjay Ghemawat and Jeffrey Dean published a paper focusing on MapReduce, the programming model for the data. The Apache Software Foundation used these resources to develop a free open source platform named Apache Hadoop that implements these and other technologies. This chapter discusses some details of the Google File System as well as guidelines to experimenting with Apache Hadoop Distributed File System.

## The Google File System
This section is by no means the complete description of GFS. It is rather a brief summary of ideas and implementation details that GFS engineers had used. The interested reader is encouraged to study the full paper available on Google Scholar [^GFS_paper].

To start, let us focus on how can we save a huge file? Well, the first answer that comes to mind is to get a large capacity hard drive. However, what if the file size exceeds the size of the largest hard drive and the file is also growing? The solution is to use a distributed file system, where the portions of the file will get saved on multiple hard drives, i.e. the nodes. As necessary, more hard drives can be added to the system (or replaced as the currently running drives fail) to fulfill the needs. This is exactly what the engineers at Google have implemented, which is a distributed file system containing multiple commodity computers equipped with commodity hard drives. Such hard drives have the regular life expectancy of approximately three years and thus a system with a thousand nodes will experience one failure a day on average. Hence, the first assumption is that the failure of a component is the norm. To prevent the loss of data, it gets replicated across several nodes (three by default) and when a node is down, there should still be two replicas of the data somewhere. 

To facilitate the storage of large files and in attempt to equally load the drives in the system, the files are also split into chunks. In the GFS, the chunks are 64 MB each and it is the chunks that get replicated. So in the end, the chunks of one file can be placed on many hard drives in the system. Such distribution also helps with data processing, when the clients can run on the same nodes where the data resides.

GFS has a single master node and multiple chunkservers. The master keeps in memory file metadata and the mapping of files to chunks and chunks to chunkservers. The metadata and file mapping is also saved in the operation log, which is periodically copied to a persistent storage. A significant implementation detail is that the mapping of chunks to chunkservers is not stored persistently. This information is updated as necessary via heartbeat messages that the master and chunkservers exchange. When the master does not get a heartbeat from a chunkserver, it concludes that the latter is likely down, its chunks are now underreplicated, which needs to be fixed. 

Another important implementation detail is how the client reads or updates file contents in GFS. On such operations, the client first requests the master to return the list of chunkservers containing file data. Then the client works `directly` with the chunkservers without necessarily involving the master. On write, the master selects a primary chunkserver and issues a lease to it. The primary chunkserver is responsible for transferring data further to the replicas and keeping track of the acknowledgements. To optimize the process, the primary will not wait until all data arrives and starts sending it immediately. Suppose that there is no network congestion, then the elapsed time to send B bytes to R replicas is:

\begin{equation}
ElapsedTime = \frac{B}{T}+R \cdot L
\end{equation}

where T - transfer rate and L -- latency to transfer bytes between two machines. 

At this time, let us switch to Hadoop Distributed File System (HDFS) and try a few things over there.

## Hadoop Installation and running

### Docker

The official Hadoop resources [^Hadoop_links], contain comprehensive information on how to install the framework locally or on a cluster. While this may sound as an exciting endeavour, it will unnecessary delay our acquaintance with Hadoop. Rather, we can use a Docker container that already has Hadoop cluster installed. 

In case you have not worked with Docker before, it is a container engine that runs on top of Linux operating system and uses its namespaces and control groups. The result is similar to running software on a virtual machine, however, in Docker's case, the `virtual machine' is using the kernel of the host operating system, which makes the process way more efficient.

Docker operates with images and containers, where an image is a lightweight list of instructions to create a container, and the container is a runnable instance of an image. `http://hub.docker.com` has a library of preconfigured Docker images, where you can find many ready-to-use software packages. The image that we need can be found under `abezuglov/data201:007`. The command to pull the image is:

```console
# docker pull abezuglov/data201:007
```

The command may take a few minutes to download all the packages and unpack them. After it finishes, `docker image -ls` will show the newly downloaded image. The command to start the containerized Hadoop instance is:

```console
# docker run --rm -it -p 8888:8888 abezuglov/data201:007 
```

Here, apart from the self-explanatory 'run' command, parameter `-it` specifies that the instance will be interactive and it will use the current terminal, `-p` instructs to use port 8888, and `--rm` will remove the container files after it finishes. We can drop the latter option to store and reuse the container files. 

![DATA201_start](images/figures/docker_data201_start.png)

While the container is running, open your web browser and go to `http://localhost:8888`. This will open the Jupyter notebook connected to the container machine. 

![DATA201_browser](images/figures/docker_data201_browser.png)

Besides, link `http://localhost:50070` will bring up the namenode information page:

![DATA201_namenode](images/figures/docker_data201_namenode_info.png)

To detach from the terminal window, but keep everything running, use `Ctrl-p`, `Ctrl-q`, and close the window. The container can also be paused/unpaused with `docker pause <container>` and `docker unpause <container>`.

Finally, to stop and exit from the container, use `Ctrl-C`. Note that unless you remove `--rm` parameter when starting, the files and notebooks in the container will not get saved.

### Standalone Hadoop installation
Docker containers significantly reduce time to install and setup Hadoop. However, even without containers, its installation is straightforward and a step-by-step configuration details can be found online.[^Hadoop_install_Edureka] 

While Hadoop is running in standalone mode, its instances are listening on the 9xxx ports of the host computer. The exact port numbers may depend on Hadoop configuration and can be found by using `netstat -a`. If you follow the Hadoop installation through Edureka, the ports are:

* Namenode: `http://localhost:9870`
* Secondary Namenode: `http://localhost:9868`
* DataNode: `http://localhost:9866`

Note that the procedure above installs and setups Hadoop itself. However, MapReduce and yarn are not yet configured. 

## Interacting with HDFS
When everything is configured and running, HDFS can be reached through a new terminal window in Jupyter. Alternatively, Jupyter notebook cells also allow running bash commands with `%%bash <command>`. Various HDFS functions are available by running:
```console
# hdfs dfs -<command>
```
where `-<command>` can be `mkdir`, `cat`, `chown`, `chmod`, `cp`, `mkdir`, `mv`, `rm`, `rmdir`. As usual, if `-<command>` clause is missing, Hadoop will print the list of available commands. 

Let us create a new directory in HDFS and upload a file from the local (Docker) file system. The commands below will do exactly this, plus show the contents of the directory as well as the statistics of the newly uploaded file. If shakespeare.txt is missing, any text or data file will work[^shakespeare_file]:

```console
# hdfs dfs -mkdir texts
# hdfs dfs -copyFromLocal shakespeare.txt texts/
# hdfs dfs -ls texts/
# hdfs fsck texts/shakespeare.txt -files -blocks -locations
```

![DATA201_mkdir_output](/images/figures/docker_data201_mkdir_output.png)

In this example, we operate with a single node Hadoop installation, so there is only one replica of the file. In case of a real multinode installation, the number of replicas and block sizes can be customized as follows:

```console
# hdfs dfs -Ddfs.replication=5 -Ddfs.blocksize=1048576 -copyFromLocal <...>
# hdfs fsck <...> -files -blocks -locations
```

In the above example, the file(s) will get copied to HDFS and replicated 5 times with the block size of 1MB. 

## Simple Data Analysis with HDFS via Linux commands

Suppose our goal is to compute the frequency of each word in Shakespeare sonnets (`shakespeare.txt`), the file that we have previously put to HDFS. This is a slight modification of the word count problem, which is the Hadoop's version of 'Hello, world!'. The output frequencies should look similar to these:

![DATA201_word_freqs](/images/figures/data201_word_freqs.png)

A good solution to this problem would utilize the file distirbution across data nodes on the cluster. However, we have not discussed MapReduce, which is the framework for running code on multiple nodes in parallel, so let us come up with the basic solution that only uses Linux commands and the piping: 

```console
#hdfs dfs -cat texts/shakespeare.txt | tr -s '[[:punct:][:space:]]' '\n'
	| sort | uniq -c | sort --key=1nr | python count_words.py| head
```
The first command here (`-cat`) reads the file, then `tr` puts each word into a new line, `sort` and `uniq -c` sort and count words, then the file is sorted to put the most frequent words at the top and finally a Python program is called that calculates word frequencies. The Python program is listed below and in case you work in Jupyter notebook, just put `%%writefile count_words.py` at the beginning of the cell with the program code.

```python
import sys
words = []
counts = []
total_count = 0
for line in sys.stdin:
    count,word = line.strip().split(' ')
    count = int(count)
    total_count += count
    words.append(word)
    counts.append(count)
    
for p in zip(words,counts):
    print('%s\t%.5f'%(p[0],p[1]/total_count))
```

While this data analysis example might not seem like a big deal, since other methods can do the same analysis much more efficiently. However, it gives another insight on how to process data on a distributed file system. Since the file chunks are already located on several data nodes, at least the first part of the analysis code can run on those nodes simultaneously. It is the word counting and frequency calculations that we temporarily cannot do in parallel. How can it be done? This is the primary focus of MapReduce chapter. 

<!--- 

* Download java jdk /usr/java
* Download hadoop /usr/hadoop

Note that the data saved in the container are NOT persistent, meaning that every time we run `docker run -it ...` (see above), a brand new container is created. One way to save the container state is to commit it as another image. Upon exiting the container bash, record the container's ID by running:
```console
# docker ps -a | head -5
```
This command will list five most recently used containers, where you can look up the necessary ID:
![docker ps a](/images/figures/docker_ps_a.png)

To save container D: `e6d90e1fe489` as an image with name `hdfs_shakespeare` and tag 1, the command is:
```console
# docker commit e6d90e1fe489 hdfs_shakespeare:1
```

```console
# /usr/local/hadoop/bin/hdfs dfsadmin -report
```

```console
# docker run --rm -it -p 8888:8888 data201:007
```
-->

## Review questions
* How does the master determine whihc replica is on which chunkserver? Is this information persistent?
* How does GFS track the metadata changes?
* What happens when the master crashes?
* Does GFS guarantee that the data is `defined` after a sequence of mutations?
* Does GFS guarantee that each data record will append `exactly once`?
* How does GFS store per-directory data structure?
* Explain considerations for placing replicas? I.e. same rack, same node, different racks, etc.
* Explain considerations for or against creating multiple new files on a lightly loaded chunkserver?
* When does re-replication occur? What are its priority factors?
* How does GFS delete files?
* What is a stale replica? How is it detected?

## Exercises

1. Suppose a cluster needs to store 100 files of 20TB each with replication factor of 3. How much memory should a master node have? Assume 64 bytes in memory per chunk and the replication factor of 3.
2. What is the elapsed time of sending 1TB of data to 3 replicas over a 1Gbps network with the network delay of 1ms?
3. Download a csv file from Census and upload it to HDFS. Use the block size of 2MB and replication factor 3 (in case you run a single node Hadoop installation, provide the command with the output)
4. Calculate word frequencies in Shakespeare sonnets, with the most common English words removed. You can find English stop words file online and save it locally. Create another Python program to filter out the stop words. 


[^GFS_paper]: The paper is available at https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf
[^Hadoop_links]: http://hadoop.apache.org/
[^hadoop_hub_link]: You can find the page with the image description here: https://hub.docker.com/r/sequenceiq/hadoop-docker/
[^shakespeare_file]: In case it is not available, you can download it with command: `curl https://ocw.mit.edu/ans7870/6/6.006/s08/lecturenotes/files/t8.shakespeare.txt > shakespeare.txt'
[^Hadoop_install_Edureka]: https://www.edureka.co/blog/install-hadoop-single-node-hadoop-cluster

