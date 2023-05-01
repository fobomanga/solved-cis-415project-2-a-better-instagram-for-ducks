Download Link: https://assignmentchef.com/product/solved-cis-415project-2-a-better-instagram-for-ducks
<br>
Today, social networking is where it is at. Puddles, the Oregon Duck and a social animal himself, was browsing the University of Oregon’s <strong>Around</strong><em>the</em><strong>O </strong>webpage and got a thought about how to take it to the next level. Puddles’s idea was to create a social networking platform that would appeal to the UO undergraduates that make up his fan base. After consulting his “branding” manager, Puddles decided to call his new social media service <strong>InstaQuack</strong>! The goal is to have a way for the UO student community to communicate what they are doing with realtime photos. A cross between Twitter and Instagram, InstaQuack will have proprietary server technology called <em>Quacker </em>that can connect photo bombers (like Puddles) with the photosphere (where is his Fan Club hangs out), but with the twist that new photos will always be more accessible than old photos. Now, all he needs is to recruit OS developers from Prof. Malony’s CIS 415 class to make it happen.

At the heart of many systems that share information between users is the <em>publish/subscribe </em>model. The idea is that <em>publishers </em>of information on different <em>topics </em>want to share that information (articles, pictures, …) with <em>subscribers </em>to those topics. The publish/susbscribe (<em>pub/sub</em>) model makes this possible by allowing publishers and subscribers to be created and operate in a simple manner: publishers send the pub/sub system information for certain topics and subscribers receive information from the pub/sub system for those topics that they are subscribed to. There can be multiple publishers and subscribers and they all may be interested in different topics.

In the case of InstaQuack, what is being published are only photos with a very brief caption. Puddles, being the photophile that he is, wants the topics to be associated whatever best describes the photo, like birds, mountains, parties, people, and so on. Because he has a limited attention span, Puddles also wants to see only the most recent photos.

InstaQuack must be responsive, scalable, and rival anything that can be found at other Pac-12 schools. With the extraordinary skillset that you are learning in CIS 415, you will implement the heart of InstaQuack – the Quacker pub/sub server architecture – shown in Figure 1.

There are 5 parts to the project, each building on the other. The objective is to get experience with a combination of OS techniques in your solution, mainly threading, synchronization, and file I/O.

<h1>Part 1 – Quacker Topic Store</h1>

Quacker is at the heart of InstaQuack. It is where recently published photos are stored. Each topic has a bounded queue (buffer) where publishers <em>enqueue </em>and subscribers <em>dequeue</em>. The objective of Part 1 is to build a bounded queue that can be used by multiple publisher threads and multiple subscriber threads. It this is implemented successfully, then all that is needed to create the Quacker topic store is to replicate the queues. Implement the following:

<ul>

 <li>Create a circular, FIFO topic queue capable of holding MAXENTRIES topic entries, where each topic entry consists of:</li>

</ul>

Figure 1: InstaQuack and the Quacker pub/sub server architecture.

struct topicentry {

int entrynum; struct timeval timestamp; int pubID; char photoUrl[QUACKSIZE];           /* URL to photo */ char photoCaption[CAPTIONSIZE]; /* photo caption */ }

There will be <em>MAXTOPICS </em>total topic queues.

<ul>

 <li>Each topic queue needs to have a <em>head </em>and a <em>tail </em> The head of the topic queue points to the newest (most recent) entry in the topic queue. The tail of the topic queue points to the oldest entry put in the queue.</li>

 <li>Write an enqueue() routine to enqueue a topic entry. Each topic entry enqueued will be assigned a monotonically increasing entry number starting at 1. The topic queue itself will have a entry counter. Because multiple threads are access the topic queue, enqueue() must synchronize its access to the topic queue. Once a thread has gained access, the enqueue() routine will read the counter, increment it, and save it back in the topic entry. A timestamp will be also taken using gettimeofday() and saved in the topic entry.</li>

 <li>Write an getentry() routine to get a topic entry from the topic queue. The routine will take an argument int lastentry which is the number of the last entry read by the calling thread on this topic. The routine will attempt to get the lastentry+1 entry, if it is in the topic queue. There are 4 cases to consider:

  <ul>

   <li><strong>Case 1: topic queue is empty – </strong>getentry() will return 0.</li>

   <li><strong>Case 2: </strong>lastentry+1 <strong>entry is in the queue – </strong>getentry() will scan the queue entries, starting with the oldest entry in the queue, until it finds the lastentry+1 entry, then it copies the entry into the topicentry structure pointed to by the struct topicentry *t argument and return 1.</li>

   <li><strong>Case 3: topic queue is not empty and </strong>lastentry+1 <strong>entry is not the queue – </strong>There are 2 sub-cases:</li>

  </ul></li>

</ul>

∗ a. getentry() will scan the queue entries, starting with the oldest entry in the queue. If all entries in the queue are less than lastentry+1, that entry has yet to be put into the queue and getentry() will return 0. (Note, this is like the queue is empty.)

∗ b. getentry() will scan the queue entries, starting with the oldest entry in the queue. If it encounters an entry greater than lastentry+1, copy that entry into the topicentry structure pointed to by the struct topicentry *t argument and return the entrynum of that entry. (Note, this case occurs because the lastentry+1 entry was dequeued by the <em>cleanup </em>thread (see below). The calling thread should update its lastentry to the entrynum. If you think about this case, the first entry that is greater than lastentry+1 will be the oldest entry in the queue.)

Because multiple (subscriber) threads can call getentry(), it must gain synchronized access to the topic queue.

<ul>

 <li>Topic entries can get too old to keep in the topic queues. If a topic entry ages <em>DELTA </em>beyond when it was inserted into the queue, it should be dequeued. Write a dequeue() routine that dequeues old topic entries. This routine will be executed by only 1 thread call the <em>topic cleanup </em> It should periodically call dequeue() on every topic queue. After checking each queue, it should yield<a href="#_ftn1" name="_ftnref1"><sup>[1]</sup></a>Because there are multiple threads trying to access to the topic queue, dequeue() must synchronize its access.</li>

 <li>Test the Quacker topic queue implementation to show that it is working. Start with just a single publisher thread, a single subscriber thread, and the cleanup thread. Then add more threads. When it is working, replicate the queues and do more testing.</li>

</ul>

Figure 2 give a high-level view of the topic entry queue. There is one of these queues for each topic. It has a fixed size and should be implemented as a circular ring buffer. You should be able to retrofit the bounded buffer code that you are working on in the lab, or write your own from scratch.

perform enqueue operations

Subscriber proxy threads perform getentry operations

Figure 2: Topic entry queue and operations.

Part 4 is trickier than it seems. Let’s start with the publishers. Any topic can have multiple publishers. Enqueueing must be synchronized, but it is possible for a topic queue to become full. If a publisher wants to enqueue an entry to a full topic queue, it has to wait until there is space to do so. The simplest way to do this is to just yield the CPU and test again when the thread is re-scheduled. (If you want to get fancier, you could consider using a blocking semaphore to implement this.) Eventually, a dequeue operation will come along and free up queue space.

Now consider the subscribers. All subscribers for a topic must read topic entries in order, using a monotonically increasing message number. However, once a topic entry has reached a certain age, it might be dequeued instead of read by a subscriber. Who does the dequeuing? The getentry() routine will only indicate whether the next entry is available or, if not, whether there is a newer entry available. The problem is that either there are no newer entries for this particular subscriber thread or the queue is empty. In either case, the subscriber thread should try again. The simplest way to do this is to just yield the CPU and try again when the thread is re-scheduled. (If you want to get fancier, you could consider using a blocking semaphore to implement this.) Eventually, an enqueue() operation will come along.

<h1>Part 2 – Constructing the Quacker Server</h1>

Now that you have the Quacker topic store working, Part 2 looks to build the the rest of the Quacker server. We first will make it multithread. The idea is that when an InstaQuack producer or subscriber “connects” to the Quacker server, a “proxy” thread (of the appropriate type) is assigned to do their requested actions in the server. Because publishers and subscribers may come and go, instead of creating a new proxy thread each time, the Quacker server is initialized with a pool <em>NUMPROXIES </em>producer proxy threads and a pool of <em>NUMPROXIES </em>subscriber proxy threads. A “free” proxy thread (of the appropriate type) is used by a producer or consumer and then returned to the pool when they complete. Part 3 will implement a program that creates publisher and subscriber proxy threads, connects them to the Quacker topic store, and tests their functionality as follows:

<ul>

 <li>Create a program that will be the skeleton of the Quacker server. It will create the Quacker topic store and initialize it, it will create the proxy thread pools (see below), and then it will run experiments to test that things are working.</li>

 <li>Create <em>NUMPROXIES </em>publisher proxy threads, initialize them, and put them in a table representing the publisher proxy thread pool. Each proxy thread is assigned a table entry. Each table entry has a flag to indicate whether the thread is free and the thread’s id. Each thread (<em>T<sub>i</sub><sup>p</sup>, </em>1 ≤ <em>i </em>≤ <em>NUMPROXIES </em>is initially free.</li>

 <li>Create <em>NUMPROXIES </em>subscriber proxy threads, initialize them, and put them in a table representing the subscribr proxy thread pool. Each proxy thread is assigned a table entry. Each table entry has a flag to indicate whether the thread is free and the thread’s id. Each thread is initially free.</li>

 <li>Write tests that allocate publisher proxy threads and provide the threads with topic entries (could be a list of entries) to publish. When there are no more entries for a particular publisher proxy thread, it is returned to the pool.</li>

 <li>Write tests that allocate subscriber proxy threads and provide the threads with topics to read entries from (could be a list of topics). When there are no more topics to read for a particular subscriber proxy thread, it is returned to the pool.</li>

 <li>Run the tests concurrently as best you can. You should try to mix things up to get the various proxy threads to overlap in their execution.</li>

</ul>

You should use Pthreads to implement the threading. Be careful to make the code you develop <em>thread safe</em>. Completing this part should give a functional core Quacker server.

<h1>Part 3 – Creating InstaQuack (Pseudo) Publishers/Subscribers</h1>

In Part 3 we want to make it possible for InstaQuack publishers and subscribers to connect to Quacker. To not make it overly complicated, the idea is to represent the publisher and subscriber behaviors in a set of files that the Quacker server reads after initialization. Each file is a set of commands that a publisher or subscriber wants to do. To begin, we need to have a set of commands to create the publishers and subscribers. These will come in on standard input and be interpreted by the Quacker server as follows:

<ul>

 <li>create topic &lt;topic ID&gt; “&lt;topic name&gt;” &lt;queue length&gt; Create a topic with ID (integer) and length. This allocates a topic queue.</li>

 <li>query topics</li>

</ul>

Print out all topic IDs and their lengths.

<ul>

 <li>add publisher “&lt;publisher command file&gt;”</li>

</ul>

Create a publisher. A free thread is allocated to be the “proxy” for the publisher. When the publisher is started (see below), the thread reads its commands from the file.

<ul>

 <li>query publishers</li>

</ul>

Print out current publishers and their command file names.

<ul>

 <li>add subscriber “&lt;subscriber command file&gt;”</li>

</ul>

Create a subscriber. A free thread is allocated to be the “proxy” for the subscriber. When the subscriber is started (see below), the thread reads its commands from the file.

<ul>

 <li>query subscribers</li>

</ul>

Print out subscribers and their command file names.

<ul>

 <li>delta &lt;DELTA&gt;</li>

</ul>

Set <em>DELTA </em>to the value specified.

<ul>

 <li>start</li>

</ul>

Start all of the publishers and subscribers. Just before this happens, the cleanup thread is started.

After these commands have been processed by the main program (master thread), the publisher and subscriber threads start to read from their command files.

<h1>Part 4 – Execute Publisher/Subscriber Commands</h1>

Once the publisher and subscribers are started, they read commands from the files and process them as follows:

<ul>

 <li>put &lt;topic ID&gt; “&lt;photo URL&gt;” “&lt;caption&gt;”</li>

</ul>

The publisher thread will attempt to put a topic entry with this information into the topic ID queue.

<ul>

 <li>get &lt;topic ID&gt;</li>

</ul>

The subscriber will attempt to get a topic entry from the topic ID queue.

<ul>

 <li>sleep &lt;milliseconds&gt;</li>

</ul>

The publisher or subscriber will sleep for this number of milliseconds.

<ul>

 <li>stop</li>

</ul>

The publisher or subscriber thread stops reading commands and the thread is returned to the respective pool.

Once the publishers and subscribers are up and running, things proceed until they all are stopped. Make sure to check that the topic ID are correct. When all publishers and subscribers have stopped, the cleanup thread is also stopped. Note, how to handle the subscriber is a little tricky. For instance, you need to decide what to do when a subscriber tries to get an entry for a topic and there is nothing there. You could decide to try a certain number of times before giving up. Do not keep trying indefinitely! Also, when there are multiple entries, you might decide to just get them all.

<h1>Part 5 – InstaQuack Topic Web Pages</h1>

Part 4 did not say what the subscriber does with the topic entry data it gets. One idea is for every subscriber and for all its topics to create a simple HTML file that looks something like the following:

&lt;html&gt;

&lt;head&gt;

&lt;title&gt;

&lt;/head&gt;

&lt;body&gt;

&lt;!–

Fill in &lt;topic name&gt; in the following.

–&gt;

&lt;table&gt;

&lt;tr&gt;

&lt;td align=”left”&gt;

&lt;img “SRC=puddles.gif” WIDTH=”140″ HEIGHT=”140″&gt;&amp;nbsp&amp;nbsp&amp;nbsp&amp;nbsp&amp;nbsp&lt;/a&gt;

&lt;/td&gt;

&lt;td align=”center”&gt;

&lt;h1&gt;InstaQuack&lt;/h1&gt;

&lt;h1&gt;Subscriber &lt;subscriber I&gt; : Topic &lt;topic name&gt;&lt;/h1&gt;

&lt;/td&gt;

&lt;td align=”right”&gt;

&amp;nbsp&amp;nbsp&amp;nbsp&amp;nbsp&amp;nbsp&lt;img “SRC=puddles.gif” WIDTH=”140″ HEIGHT=”140″&gt;&lt;/a&gt;

&lt;/td&gt;

&lt;/tr&gt;

&lt;/table&gt;

&lt;!–

For all topic entries seen by THIS subscriber for THIS topic, insert the following, where &lt;photo URL&gt; and &lt;photo caption&gt; come from the entry:

–&gt;

&lt;hr&gt;

&lt;img “SRC=&lt;photo URL&gt;”&gt;&lt;/a&gt;&lt;br&gt;

&lt;photo caption&gt;

&lt;hr&gt;

&lt;!–

Continue inserting until the subscriber is done.

–&gt;

&lt;/body&gt;

&lt;/html&gt;

In this way, you will be able to open each of these files in a web browser and see that each subscriber was able to get from the topics.

The main program can be responsible for setting up the files before starting the publishers and subscribers, or they can do it themselves. You will be provided a set of routines that can be used to produce the file content from topic entry information.

<a href="#_ftnref1" name="_ftn1">[1]</a> See sched yield(2).