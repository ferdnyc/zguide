---
weight: 5
title: '5. Advanced Pub-Sub Patterns'
---

# Chapter 5 - Advanced Pub-Sub Patterns {#advanced-pub-sub}

In [Chapter 3 - Advanced Request-Reply Patterns](chapter3#advanced-request-reply) and [Chapter 4 - Reliable Request-Reply Patterns](chapter4#reliable-request-reply) we looked at advanced use of ZeroMQ's request-reply pattern. If you managed to digest all that, congratulations. In this chapter we'll focus on publish-subscribe and extend ZeroMQ's core pub-sub pattern with higher-level patterns for performance, reliability, state distribution, and monitoring.

We'll cover:

* When to use publish-subscribe
* How to handle too-slow subscribers (the *Suicidal Snail* pattern)
* How to design high-speed subscribers (the *Black Box* pattern)
* How to monitor a pub-sub network (the *Espresso* pattern)
* How to build a shared key-value store (the *Clone* pattern)
* How to use reactors to simplify complex servers
* How to use the Binary Star pattern to add failover to a server

## Pros and Cons of Pub-Sub {#Pros-and-Cons-of-Pub-Sub}

ZeroMQ's low-level patterns have their different characters. Pub-sub addresses an old messaging problem, which is *multicast* or *group messaging*. It has that unique mix of meticulous simplicity and brutal indifference that characterizes ZeroMQ. It's worth understanding the trade-offs that pub-sub makes, how these benefit us, and how we can work around them if needed.

First, PUB sends each message to "all of many", whereas PUSH and DEALER rotate messages to "one of many". You cannot simply replace PUSH with PUB or vice versa and hope that things will work. This bears repeating because people seem to quite often suggest doing this.

More profoundly, pub-sub is aimed at scalability. This means large volumes of data, sent rapidly to many recipients. If you need millions of messages per second sent to thousands of points, you'll appreciate pub-sub a lot more than if you need a few messages a second sent to a handful of recipients.

To get scalability, pub-sub uses the same trick as push-pull, which is to get rid of back-chatter. This means that recipients don't talk back to senders. There are some exceptions, e.g., SUB sockets will send subscriptions to PUB sockets, but it's anonymous and infrequent.

Killing back-chatter is essential to real scalability. With pub-sub, it's how the pattern can map cleanly to the PGM multicast protocol, which is handled by the network switch. In other words, subscribers don't connect to the publisher at all, they connect to a multicast *group* on the switch, to which the publisher sends its messages.

When we remove back-chatter, our overall message flow becomes *much* simpler, which lets us make simpler APIs, simpler protocols, and in general reach many more people. But we also remove any possibility to coordinate senders and receivers. What this means is:

* Publishers can't tell when subscribers are successfully connected, both on initial connections, and on reconnections after network failures.

* Subscribers can't tell publishers anything that would allow publishers to control the rate of messages they send. Publishers only have one setting, which is *full-speed*, and subscribers must either keep up or lose messages.

* Publishers can't tell when subscribers have disappeared due to processes crashing, networks breaking, and so on.

The downside is that we actually need all of these if we want to do reliable multicast. The ZeroMQ pub-sub pattern will lose messages arbitrarily when a subscriber is connecting, when a network failure occurs, or just if the subscriber or network can't keep up with the publisher.

The upside is that there are many use cases where *almost* reliable multicast is just fine. When we need this back-chatter, we can either switch to using ROUTER-DEALER (which I tend to do for most normal volume cases), or we can add a separate channel for synchronization (we'll see an example of this later in this chapter).

Pub-sub is like a radio broadcast; you miss everything before you join, and then how much information you get depends on the quality of your reception. Surprisingly, this model is useful and widespread because it maps perfectly to real world distribution of information. Think of Facebook and Twitter, the BBC World Service, and the sports results.

As we did for request-reply, let's define *reliability* in terms of what can go wrong. Here are the classic failure cases for pub-sub:

* Subscribers join late, so they miss messages the server already sent.
* Subscribers can fetch messages too slowly, so queues build up and then overflow.
* Subscribers can drop off and lose messages while they are away.
* Subscribers can crash and restart, and lose whatever data they already received.
* Networks can become overloaded and drop data (specifically, for PGM).
* Networks can become too slow, so publisher-side queues overflow and publishers crash.

A lot more can go wrong but these are the typical failures we see in a realistic system. Since v3.x, ZeroMQ forces default limits on its internal buffers (the so-called high-water mark or HWM), so publisher crashes are rarer unless you deliberately set the HWM to infinite.

All of these failure cases have answers, though not always simple ones. Reliability requires complexity that most of us don't need, most of the time, which is why ZeroMQ doesn't attempt to provide it out of the box (even if there was one global design for reliability, which there isn't).

## Pub-Sub Tracing (Espresso Pattern) {#Pub-Sub-Tracing-Espresso-Pattern}

Let's start this chapter by looking at a way to trace pub-sub networks. In [Chapter 2 - Sockets and Patterns](chapter2#sockets-and-patterns) we saw a simple proxy that used these to do transport bridging. The <tt>[zmq_proxy()](http://api.zeromq.org/3-2:zmq_proxy)</tt> method has three arguments: a *frontend* and *backend* socket that it bridges together, and a *capture* socket to which it will send all messages.

The code is deceptively simple:

{{< examples name="espresso" title="Espresso Pattern" >}}

Espresso works by creating a listener thread that reads a PAIR socket and prints anything it gets. That PAIR socket is one end of a pipe; the other end (another PAIR) is the socket we pass to <tt>[zmq_proxy()](http://api.zeromq.org/3-2:zmq_proxy)</tt>. In practice, you'd filter interesting messages to get the essence of what you want to track (hence the name of the pattern).

The subscriber thread subscribes to "A" and "B", receives five messages, and then destroys its socket. When you run the example, the listener prints two subscription messages, five data messages, two unsubscribe messages, and then silence:

```
[002] 0141
[002] 0142
[007] B-91164
[007] B-12979
[007] A-52599
[007] A-06417
[007] A-45770
[002] 0041
[002] 0042
```

This shows neatly how the publisher socket stops sending data when there are no subscribers for it. The publisher thread is still sending messages. The socket just drops them silently.

## Last Value Caching {#Last-Value-Caching}

If you've used commercial pub-sub systems, you may be used to some features that are missing in the fast and cheerful ZeroMQ pub-sub model. One of these is *last value caching* (LVC). This solves the problem of how a new subscriber catches up when it joins the network. The theory is that publishers get notified when a new subscriber joins and subscribes to some specific topics. The publisher can then rebroadcast the last message for those topics.

I've already explained why publishers don't get notified when there are new subscribers, because in large pub-sub systems, the volumes of data make it pretty much impossible. To make really large-scale pub-sub networks, you need a protocol like PGM that exploits an upscale Ethernet switch's ability to multicast data to thousands of subscribers. Trying to do a TCP unicast from the publisher to each of thousands of subscribers just doesn't scale. You get weird spikes, unfair distribution (some subscribers getting the message before others), network congestion, and general unhappiness.

PGM is a one-way protocol: the publisher sends a message to a multicast address at the switch, which then rebroadcasts that to all interested subscribers. The publisher never sees when subscribers join or leave: this all happens in the switch, which we don't really want to start reprogramming.

However, in a lower-volume network with a few dozen subscribers and a limited number of topics, we can use TCP and then the XSUB and XPUB sockets *do* talk to each other as we just saw in the Espresso pattern.

Can we make an LVC using ZeroMQ? The answer is yes, if we make a proxy that sits between the publisher and subscribers; an analog for the PGM switch, but one we can program ourselves.

I'll start by making a publisher and subscriber that highlight the worst case scenario. This publisher is pathological. It starts by immediately sending messages to each of a thousand topics, and then it sends one update a second to a random topic. A subscriber connects, and subscribes to a topic. Without LVC, a subscriber would have to wait an average of 500 seconds to get any data. To add some drama, let's pretend there's an escaped convict called Gregor threatening to rip the head off Roger the toy bunny if we can't fix that 8.3 minutes' delay.

Here's the publisher code. Note that it has the command line option to connect to some address, but otherwise binds to an endpoint. We'll use this later to connect to our last value cache:

{{< examples name="pathopub" title="Pathologic Publisher" >}}

And here's the subscriber:

{{< examples name="pathosub" title="Pathologic Subscriber" >}}

Try building and running these: first the subscriber, then the publisher. You'll see the subscriber reports getting "Save Roger" as you'd expect:

```
./pathosub &
./pathopub
```

It's when you run a second subscriber that you understand Roger's predicament. You have to leave it an awful long time before it reports getting any data. So, here's our last value cache. As I promised, it's a proxy that binds to two sockets and then handles messages on both:

{{< examples name="lvcache" title="Last Value Caching Proxy" >}}

Now, run the proxy, and then the publisher:

```
./lvcache &
./pathopub tcp://localhost:5557
```

And now run as many instances of the subscriber as you want to try, each time connecting to the proxy on port 5558:

```
./pathosub tcp://localhost:5558
```

Each subscriber happily reports "Save Roger", and Gregor the Escaped Convict slinks back to his seat for dinner and a nice cup of hot milk, which is all he really wanted in the first place.

One note: by default, the XPUB socket does not report duplicate subscriptions, which is what you want when you're naively connecting an XPUB to an XSUB. Our example sneakily gets around this by using random topics so the chance of it not working is one in a million. In a real LVC proxy, you'll want to use the <tt>ZMQ_XPUB_VERBOSE</tt> option that we implement in [Chapter 6 - The ZeroMQ Community](chapter6#the-community) as an exercise.

## Slow Subscriber Detection (Suicidal Snail Pattern) {#Slow-Subscriber-Detection-Suicidal-Snail-Pattern}

A common problem you will hit when using the pub-sub pattern in real life is the slow subscriber. In an ideal world, we stream data at full speed from publishers to subscribers. In reality, subscriber applications are often written in interpreted languages, or just do a lot of work, or are just badly written, to the extent that they can't keep up with publishers.

How do we handle a slow subscriber? The ideal fix is to make the subscriber faster, but that might take work and time. Some of the classic strategies for handling a slow subscriber are:

* **Queue messages on the publisher**. This is what Gmail does when I don't read my email for a couple of hours. But in high-volume messaging, pushing queues upstream has the thrilling but unprofitable result of making publishers run out of memory and crash--especially if there are lots of subscribers and it's not possible to flush to disk for performance reasons.

* **Queue messages on the subscriber**. This is much better, and it's what ZeroMQ does by default if the network can keep up with things. If anyone's going to run out of memory and crash, it'll be the subscriber rather than the publisher, which is fair. This is perfect for "peaky" streams where a subscriber can't keep up for a while, but can catch up when the stream slows down. However, it's no answer to a subscriber that's simply too slow in general.

* **Stop queuing new messages after a while**. This is what Gmail does when my mailbox overflows its precious gigabytes of space. New messages just get rejected or dropped. This is a great strategy from the perspective of the publisher, and it's what ZeroMQ does when the publisher sets a HWM. However, it still doesn't help us fix the slow subscriber. Now we just get gaps in our message stream.

* **Punish slow subscribers with disconnect**. This is what Hotmail (remember that?) did when I didn't log in for two weeks, which is why I was on my fifteenth Hotmail account when it hit me that there was perhaps a better way. It's a nice brutal strategy that forces subscribers to sit up and pay attention and would be ideal, but ZeroMQ doesn't do this, and there's no way to layer it on top because subscribers are invisible to publisher applications.

None of these classic strategies fit, so we need to get creative. Rather than disconnect the publisher, let's convince the subscriber to kill itself. This is the Suicidal Snail pattern. When a subscriber detects that it's running too slowly (where "too slowly" is presumably a configured option that really means "so slowly that if you ever get here, shout really loudly because I need to know, so I can fix this!"), it croaks and dies.

How can a subscriber detect this? One way would be to sequence messages (number them in order) and use a HWM at the publisher. Now, if the subscriber detects a gap (i.e., the numbering isn't consecutive), it knows something is wrong. We then tune the HWM to the "croak and die if you hit this" level.

There are two problems with this solution. One, if we have many publishers, how do we sequence messages? The solution is to give each publisher a unique ID and add that to the sequencing. Second, if subscribers use <tt>ZMQ_SUBSCRIBE</tt> filters, they will get gaps by definition. Our precious sequencing will be for nothing.

Some use cases won't use filters, and sequencing will work for them. But a more general solution is that the publisher timestamps each message. When a subscriber gets a message, it checks the time, and if the difference is more than, say, one second, it does the "croak and die" thing, possibly firing off a squawk to some operator console first.

The Suicide Snail pattern works especially when subscribers have their own clients and service-level agreements and need to guarantee certain maximum latencies. Aborting a subscriber may not seem like a constructive way to guarantee a maximum latency, but it's the assertion model. Abort today, and the problem will be fixed. Allow late data to flow downstream, and the problem may cause wider damage and take longer to appear on the radar.

Here is a minimal example of a Suicidal Snail:

{{< examples name="suisnail" title="Suicidal Snail" >}}

Here are some things to note about the Suicidal Snail example:

* The message here consists simply of the current system clock as a number of milliseconds. In a realistic application, you'd have at least a message header with the timestamp and a message body with data.

* The example has subscriber and publisher in a single process as two threads. In reality, they would be separate processes. Using threads is just convenient for the demonstration.

## High-Speed Subscribers (Black Box Pattern) {#High-Speed-Subscribers-Black-Box-Pattern}

Now lets look at one way to make our subscribers faster. A common use case for pub-sub is distributing large data streams like market data coming from stock exchanges. A typical setup would have a publisher connected to a stock exchange, taking price quotes, and sending them out to a number of subscribers. If there are a handful of subscribers, we could use TCP. If we have a larger number of subscribers, we'd probably use reliable multicast, i.e., PGM.

{{< textdiagram name="fig56.png" figno="56" title="The Simple Black Box Pattern" >}}
               #-----------#
               | Publisher |
               +-----------+
               |    PUB    |
               '-----+-----'
                     |
.--------------------|-------------------.
:                    |          Fast box :
:                    v                   :
:              .-----------.             :
:              |    SUB    |             :
:              +-----------+             :
:              | Subscriber|             :
:              +-----------+             :
:              |   PUSH    |             :
:              '-----+-----'             :
:                    |                   :
:       .------------+------------.      :
:       |            |            |      :
:       v            v            v      :
:  .--------.   .--------.   .--------.  :
:  |  PULL  |   |  PULL  |   |  PULL  |  :
:  +--------+   +--------+   +--------+  :
:  | Worker |   | Worker |   | Worker |  :
:  #--------#   #--------#   #--------#  :
'----------------------------------------'
{{< /textdiagram >}}

Let's imagine our feed has an average of 100,000 100-byte messages a second. That's a typical rate, after filtering market data we don't need to send on to subscribers. Now we decide to record a day's data (maybe 250 GB in 8 hours), and then replay it to a simulation network, i.e., a small group of subscribers. While 100K messages a second is easy for a ZeroMQ application, we want to replay it *much faster*.

So we set up our architecture with a bunch of boxes--one for the publisher and one for each subscriber. These are well-specified boxes--eight cores, twelve for the publisher.

And as we pump data into our subscribers, we notice two things:

1. When we do even the slightest amount of work with a message, it slows down our subscriber to the point where it can't catch up with the publisher again.

1. We're hitting a ceiling, at both publisher and subscriber, to around 6M messages a second, even after careful optimization and TCP tuning.

The first thing we have to do is break our subscriber into a multithreaded design so that we can do work with messages in one set of threads, while reading messages in another. Typically, we don't want to process every message the same way. Rather, the subscriber will filter some messages, perhaps by prefix key. When a message matches some criteria, the subscriber will call a worker to deal with it. In ZeroMQ terms, this means sending the message to a worker thread.

So the subscriber looks something like a queue device. We could use various sockets to connect the subscriber and workers. If we assume one-way traffic and workers that are all identical, we can use PUSH and PULL and delegate all the routing work to ZeroMQ. This is the simplest and fastest approach.

The subscriber talks to the publisher over TCP or PGM. The subscriber talks to its workers, which are all in the same process, over <tt>inproc:@<//>@</tt>.

{{< textdiagram name="fig57.png" figno="57" title="Mad Black Box Pattern" >}}
               #-----------#
               | Publisher |
               +-----+-----+
               | PUB | PUB |
               '--+--+--+--'
                  |     |
.------------=--- | -=- | ---------------.
:                 |     |      Fast box  :
:                 v     v                :
:              .-----+-----.             :
:              | SUB | SUB |             :
:              +-----+-----+             :
:              | Subscriber|             :
:              +-----+-----+             :
:              |PUSH | PUSH|             :
:              '--+--+--+--'             :
:                 |     |                :
:      .----------+-.   '--------.       :
:      |            |            |       :
:      v            v            v       :
:  .--------.   .--------.   .--------.  :
:  |  PULL  |   |  PULL  |   |  PULL  |  :
:  +--------+   +--------+   +--------+  :
:  | Worker |   | Worker |   | Worker |  :
:  #--------#   #--------#   #--------#  :
'----------------------------------------'
{{< /textdiagram >}}

Now to break that ceiling. The subscriber thread hits 100% of CPU and because it is one thread, it cannot use more than one core. A single thread will always hit a ceiling, be it at 2M, 6M, or more messages per second. We want to split the work across multiple threads that can run in parallel.

The approach used by many high-performance products, which works here, is *sharding*. Using sharding, we split the work into parallel and independent streams, such as half of the topic keys in one stream, and half in another. We could use many streams, but performance won't scale unless we have free cores. So let's see how to shard into two streams.

With two streams, working at full speed, we would configure ZeroMQ as follows:

* Two I/O threads, rather than one.
* Two network interfaces (NIC), one per subscriber.
* Each I/O thread bound to a specific NIC.
* Two subscriber threads, bound to specific cores.
* Two SUB sockets, one per subscriber thread.
* The remaining cores assigned to worker threads.
* Worker threads connected to both subscriber PUSH sockets.

Ideally, we want to match the number of fully-loaded threads in our architecture with the number of cores. When threads start to fight for cores and CPU cycles, the cost of adding more threads outweighs the benefits. There would be no benefit, for example, in creating more I/O threads.

## Reliable Pub-Sub (Clone Pattern) {#Reliable-Pub-Sub-Clone-Pattern}

As a larger worked example, we'll take the problem of making a reliable pub-sub architecture. We'll develop this in stages. The goal is to allow a set of applications to share some common state. Here are our technical challenges:

* We have a large set of client applications, say thousands or tens of thousands.
* They will join and leave the network arbitrarily.
* These applications must share a single eventually-consistent *state*.
* Any application can update the state at any point in time.

Let's say that updates are reasonably low-volume. We don't have real time goals. The whole state can fit into memory. Some plausible use cases are:

* A configuration that is shared by a group of cloud servers.
* Some game state shared by a group of players.
* Exchange rate data that is updated in real time and available to applications.

### Centralized Versus Decentralized {#Centralized-Versus-Decentralized}

A first decision we have to make is whether we work with a central server or not. It makes a big difference in the resulting design. The trade-offs are these:

* Conceptually, a central server is simpler to understand because networks are not naturally symmetrical. With a central server, we avoid all questions of discovery, bind versus connect, and so on.

* Generally, a fully-distributed architecture is technically more challenging but ends up with simpler protocols. That is, each node must act as server and client in the right way, which is delicate. When done right, the results are simpler than using a central server. We saw this in the Freelance pattern in [Chapter 4 - Reliable Request-Reply Patterns](chapter4#reliable-request-reply).

* A central server will become a bottleneck in high-volume use cases. If handling scale in the order of millions of messages a second is required, we should aim for decentralization right away.

* Ironically, a centralized architecture will scale to more nodes more easily than a decentralized one. That is, it's easier to connect 10,000 nodes to one server than to each other.

So, for the Clone pattern we'll work with a *server* that publishes state updates and a set of *clients* that represent applications.

### Representing State as Key-Value Pairs {#Representing-State-as-Key-Value-Pairs}

We'll develop Clone in stages, solving one problem at a time. First, let's look at how to update a shared state across a set of clients. We need to decide how to represent our state, as well as the updates. The simplest plausible format is a key-value store, where one key-value pair represents an atomic unit of change in the shared state.

We have a simple pub-sub example in [Chapter 1 - Basics](chapter1#basics), the weather server and client. Let's change the server to send key-value pairs, and the client to store these in a hash table. This lets us send updates from one server to a set of clients using the classic pub-sub model.

An update is either a new key-value pair, a modified value for an existing key, or a deleted key. We can assume for now that the whole store fits in memory and that applications access it by key, such as by using a hash table or dictionary. For larger stores and some kind of persistence we'd probably store the state in a database, but that's not relevant here.

This is the server:

{{< examples name="clonesrv1" title="Clone server, Model One" >}}

And here is the client:

{{< examples name="clonecli1" title="Clone client, Model One" >}}

{{< textdiagram name="fig58.png" figno="58" title="Publishing State Updates" >}}
          #-----------#
          |  Server   |
          +-----------+
          |    PUB    |
          '-----+-----'
                |
                |
                |updates
    .-----------+-----------.
    |           |           |
    v           v           v
.--------.  .--------.  .--------.
|  SUB   |  |  SUB   |  |  SUB   |
+--------+  +--------+  +--------+
| Client |  | Client |  | Client |
#--------#  #--------#  #--------#
{{< /textdiagram >}}

Here are some things to note about this first model:

* All the hard work is done in a <tt>kvmsg</tt> class. This class works with key-value message objects, which are multipart ZeroMQ messages structured as three frames: a key (a ZeroMQ string), a sequence number (64-bit value, in network byte order), and a binary body (holds everything else).

* The server generates messages with a randomized 4-digit key, which lets us simulate a large but not enormous hash table (10K entries).

* We don't implement deletions in this version: all messages are inserts or updates.

* The server does a 200 millisecond pause after binding its socket. This is to prevent *slow joiner syndrome*, where the subscriber loses messages as it connects to the server's socket. We'll remove that in later versions of the Clone code.

* We'll use the terms *publisher* and *subscriber* in the code to refer to sockets. This will help later when we have multiple sockets doing different things.

Here is the <tt>kvmsg</tt> class, in the simplest form that works for now:

{{< examples name="kvsimple" title="Key-value message class" >}}

Later, we'll make a more sophisticated <tt>kvmsg</tt> class that will work in real applications.

Both the server and client maintain hash tables, but this first model only works properly if we start all clients before the server and the clients never crash. That's very artificial.

### Getting an Out-of-Band Snapshot {#Getting-an-Out-of-Band-Snapshot}

So now we have our second problem: how to deal with late-joining clients or clients that crash and then restart.

In order to allow a late (or recovering) client to catch up with a server, it has to get a snapshot of the server's state. Just as we've reduced "message" to mean "a sequenced key-value pair", we can reduce "state" to mean "a hash table". To get the server state, a client opens a DEALER socket and asks for it explicitly.

To make this work, we have to solve a problem of timing. Getting a state snapshot will take a certain time, possibly fairly long if the snapshot is large. We need to correctly apply updates to the snapshot. But the server won't know when to start sending us updates. One way would be to start subscribing, get a first update, and then ask for "state for update N". This would require the server storing one snapshot for each update, which isn't practical.

{{< textdiagram name="fig59.png" figno="59" title="State Replication" >}}
                   #--------------#
                   |   Server     |
                   +-----+--------+
                   | PUB | ROUTER |
                   '--+--+--------'
                      |      ^
                      |      | state request
                      |      '------------------.
                      | updates                 |
   .------------------+-------------------.     |
   |                  |                   |     |
   |                  |                   |     |
   v                  v                   v     |
.-----+--------.   .-----+--------.   .-----+---+----.
| SUB | DEALER |   | SUB | DEALER |   | SUB | DEALER |
+-----+--------+   +-----+--------+   +-----+--------+
|   Client     |   |   Client     |   |   Client     |
#--------------#   #--------------#   #--------------#
{{< /textdiagram >}}

So we will do the synchronization in the client, as follows:

* The client first subscribes to updates and then makes a state request. This guarantees that the state is going to be newer than the oldest update it has.

* The client waits for the server to reply with state, and meanwhile queues all updates. It does this simply by not reading them: ZeroMQ keeps them queued on the socket queue.

* When the client receives its state update, it begins once again to read updates. However, it discards any updates that are older than the state update. So if the state update includes updates up to 200, the client will discard updates up to 201.

* The client then applies updates to its own state snapshot.

It's a simple model that exploits ZeroMQ's own internal queues. Here's the server:

{{< examples name="clonesrv2" title="Clone server, Model Two" >}}

And here is the client:

{{< examples name="clonecli2" title="Clone client, Model Two" >}}

Here are some things to note about these two programs:

* The server uses two tasks. One thread produces the updates (randomly) and sends these to the main PUB socket, while the other thread handles state requests on the ROUTER socket. The two communicate across PAIR sockets over an <tt>inproc:@<//>@</tt> connection.

* The client is really simple. In C, it consists of about fifty lines of code. A lot of the heavy lifting is done in the <tt>kvmsg</tt> class. Even so, the basic Clone pattern is easier to implement than it seemed at first.

* We don't use anything fancy for serializing the state. The hash table holds a set of <tt>kvmsg</tt> objects, and the server sends these, as a batch of messages, to the client requesting state. If multiple clients request state at once, each will get a different snapshot.

* We assume that the client has exactly one server to talk to. The server must be running; we do not try to solve the question of what happens if the server crashes.

Right now, these two programs don't do anything real, but they correctly synchronize state. It's a neat example of how to mix different patterns: PAIR-PAIR, PUB-SUB, and ROUTER-DEALER.

### Republishing Updates from Clients {#Republishing-Updates-from-Clients}

In our second model, changes to the key-value store came from the server itself. This is a centralized model that is useful, for example if we have a central configuration file we want to distribute, with local caching on each node. A more interesting model takes updates from clients, not the server. The server thus becomes a stateless broker. This gives us some benefits:

* We're less worried about the reliability of the server. If it crashes, we can start a new instance and feed it new values.

* We can use the key-value store to share knowledge between active peers.

To send updates from clients back to the server, we could use a variety of socket patterns. The simplest plausible solution is a PUSH-PULL combination.

Why don't we allow clients to publish updates directly to each other? While this would reduce latency, it would remove the guarantee of consistency. You can't get consistent shared state if you allow the order of updates to change depending on who receives them. Say we have two clients, changing different keys. This will work fine. But if the two clients try to change the same key at roughly the same time, they'll end up with different notions of its value.

There are a few strategies for obtaining consistency when changes happen in multiple places at once. We'll use the approach of centralizing all change. No matter the precise timing of the changes that clients make, they are all pushed through the server, which enforces a single sequence according to the order in which it gets updates.

{{< textdiagram name="fig60.png" figno="60" title="Republishing Updates" >}}
               #----------------------#
               |        Server        |
               +------+--------+------+
               | PUB  | ROUTER | PULL |
               '--+---+--------+------'
                  |       ^        ^
                  |       |        | state update
                  |       |        '----------.
                  |       | state request     |
                  |       '-----------.       |
                  | updates           |       |
   .--------------+------------.      |       |
   |                           |      |       |
   |      ^       ^            |      |       |
   v      |       |            v      |       |
.-----+---+----+--+---.     .-----+---+----+--+---.
| SUB | DEALER | PUSH |     | SUB | DEALER | PUSH |
+-----+--------+------+     +-----+--------+------+
|       Client        |     |       Client        |
#---------------------#     #---------------------#
{{< /textdiagram >}}

By mediating all changes, the server can also add a unique sequence number to all updates. With unique sequencing, clients can detect the nastier failures, including network congestion and queue overflow. If a client discovers that its incoming message stream has a hole, it can take action. It seems sensible that the client contact the server and ask for the missing messages, but in practice that isn't useful. If there are holes, they're caused by network stress, and adding more stress to the network will make things worse. All the client can do is warn its users that it is "unable to continue", stop, and not restart until someone has manually checked the cause of the problem.

We'll now generate state updates in the client. Here's the server:

{{< examples name="clonesrv3" title="Clone server, Model Three" >}}

And here is the client:

{{< examples name="clonecli3" title="Clone client, Model Three" >}}

Here are some things to note about this third design:

* The server has collapsed to a single task. It manages a PULL socket for incoming updates, a ROUTER socket for state requests, and a PUB socket for outgoing updates.

* The client uses a simple tickless timer to send a random update to the server once a second. In a real implementation, we would drive updates from application code.

### Working with Subtrees {#Working-with-Subtrees}

As we grow the number of clients, the size of our shared store will also grow. It stops being reasonable to send everything to every client. This is the classic story with pub-sub: when you have a very small number of clients, you can send every message to all clients. As you grow the architecture, this becomes inefficient. Clients specialize in different areas.

So even when working with a shared store, some clients will want to work only with a part of that store, which we call a *subtree*. The client has to request the subtree when it makes a state request, and it must specify the same subtree when it subscribes to updates.

There are a couple of common syntaxes for trees. One is the *path hierarchy*, and another is the *topic tree*. These look like this:

* Path hierarchy: <tt>/some/list/of/paths</tt>
* Topic tree: <tt>some.list.of.topics</tt>

We'll use the path hierarchy, and extend our client and server so that a client can work with a single subtree. Once you see how to work with a single subtree you'll be able to extend this yourself to handle multiple subtrees, if your use case demands it.

Here's the server implementing subtrees, a small variation on Model Three:

{{< examples name="clonesrv4" title="Clone server, Model Four" >}}

And here is the corresponding client:

{{< examples name="clonecli4" title="Clone client, Model Four" >}}

### Ephemeral Values {#Ephemeral-Values}

An ephemeral value is one that expires automatically unless regularly refreshed. If you think of Clone being used for a registration service, then ephemeral values would let you do dynamic values. A node joins the network, publishes its address, and refreshes this regularly. If the node dies, its address eventually gets removed.

The usual abstraction for ephemeral values is to attach them to a *session*, and delete them when the session ends. In Clone, sessions would be defined by clients, and would end if the client died. A simpler alternative is to attach a *time to live* (TTL) to ephemeral values, which the server uses to expire values that haven't been refreshed in time.

A good design principle that I use whenever possible is to *not invent concepts that are not absolutely essential*. If we have very large numbers of ephemeral values, sessions will offer better performance. If we use a handful of ephemeral values, it's fine to set a TTL on each one. If we use masses of ephemeral values, it's more efficient to attach them to sessions and expire them in bulk. This isn't a problem we face at this stage, and may never face, so sessions go out the window.

Now we will implement ephemeral values. First, we need a way to encode the TTL in the key-value message. We could add a frame. The problem with using ZeroMQ frames for properties is that each time we want to add a new property, we have to change the message structure. It breaks compatibility. So let's add a properties frame to the message, and write the code to let us get and put property values.

Next, we need a way to say, "delete this value". Up until now, servers and clients have always blindly inserted or updated new values into their hash table. We'll say that if the value is empty, that means "delete this key".

Here's a more complete version of the <tt>kvmsg</tt> class, which implements the properties frame (and adds a UUID frame, which we'll need later on). It also handles empty values by deleting the key from the hash, if necessary:

{{< examples name="kvmsg" title="Key-value message class: full" >}}

The Model Five client is almost identical to Model Four. It uses the full <tt>kvmsg</tt> class now, and sets a randomized <tt>ttl</tt> property (measured in seconds) on each message:

{{< fragment name="kvsetttl" >}}
kvmsg_set_prop (kvmsg, "ttl", "%d", randof (30));
{{< /fragment >}}

### Using a Reactor {#Using-a-Reactor}

Until now, we have used a poll loop in the server. In this next model of the server, we switch to using a reactor. In C, we use CZMQ's <tt>zloop</tt> class. Using a reactor makes the code more verbose, but easier to understand and build out because each piece of the server is handled by a separate reactor handler.

We use a single thread and pass a server object around to the reactor handlers. We could have organized the server as multiple threads, each handling one socket or timer, but that works better when threads don't have to share data. In this case all work is centered around the server's hashmap, so one thread is simpler.

There are three reactor handlers:

* One to handle snapshot requests coming on the ROUTER socket;
* One to handle incoming updates from clients, coming on the PULL socket;
* One to expire ephemeral values that have passed their TTL.

{{< examples name="clonesrv5" title="Clone server, Model Five" >}}

### Adding the Binary Star Pattern for Reliability {#Adding-the-Binary-Star-Pattern-for-Reliability}

The Clone models we've explored up to now have been relatively simple. Now we're going to get into unpleasantly complex territory, which has me getting up for another espresso. You should appreciate that making "reliable" messaging is complex enough that you always need to ask, "Do we actually need this?" before jumping into it. If you can get away with unreliable or with "good enough" reliability, you can make a huge win in terms of cost and complexity. Sure, you may lose some data now and then. It is often a good trade-off. Having said, that, and... sips... because the espresso is really good, let's jump in.

As you play with the last model, you'll stop and restart the server. It might look like it recovers, but of course it's applying updates to an empty state instead of the proper current state. Any new client joining the network will only get the latest updates instead of the full historical record.

What we want is a way for the server to recover from being killed, or crashing. We also need to provide backup in case the server is out of commission for any length of time. When someone asks for "reliability", ask them to list the failures they want to handle. In our case, these are:

* The server process crashes and is automatically or manually restarted. The process loses its state and has to get it back from somewhere.

* The server machine dies and is offline for a significant time. Clients have to switch to an alternate server somewhere.

* The server process or machine gets disconnected from the network, e.g., a switch dies or a datacenter gets knocked out. It may come back at some point, but in the meantime clients need an alternate server.

Our first step is to add a second server. We can use the Binary Star pattern from [Chapter 4 - Reliable Request-Reply Patterns](chapter4#reliable-request-reply) to organize these into primary and backup. Binary Star is a reactor, so it's useful that we already refactored the last server model into a reactor style.

We need to ensure that updates are not lost if the primary server crashes. The simplest technique is to send them to both servers. The backup server can then act as a client, and keep its state synchronized by receiving updates as all clients do. It'll also get new updates from clients. It can't yet store these in its hash table, but it can hold onto them for a while.

So, Model Six introduces the following changes over Model Five:

* We use a pub-sub flow instead of a push-pull flow for client updates sent to the servers. This takes care of fanning out the updates to both servers. Otherwise we'd have to use two DEALER sockets.

* We add heartbeats to server updates (to clients), so that a client can detect when the primary server has died. It can then switch over to the backup server.

* We connect the two servers using the Binary Star <tt>bstar</tt> reactor class. Binary Star relies on the clients to vote by making an explicit request to the server they consider active. We'll use snapshot requests as the voting mechanism.

* We make all update messages uniquely identifiable by adding a UUID field. The client generates this, and the server propagates it back on republished updates.

* The passive server keeps a "pending list" of updates that it has received from clients, but not yet from the active server; or updates it's received from the active server, but not yet from the clients. The list is ordered from oldest to newest, so that it is easy to remove updates off the head.

{{< textdiagram name="fig61.png" figno="61" title="Clone Client Finite State Machine" >}}
 #-----------#
 |           |<----------------------.
 |  Initial  |<-------------------.  |
 |           |                    |  |
 #-----+-----#                    |  |
Request| snapshot                 |  |
       |   .-------------------.  |  |
       |   |                   |  |  |
       v   v                   |  |  |
 #-----------#                 |  |  |
 |           +- INPUT----------'  |  |
 |  Syncing  |  Store snapshot    |  |
 |           +- SILENCE-----------'  |
 #-----+-----#  Failover to next     |
     KTHXBAI                         |
       |   .-------------------.     |
       |   |                   |     |
       v   v                   |     |
 #-----------#                 |     |
 |           +- INPUT----------'     |
 |  Active   |  Store update         |
 |           +- SILENCE--------------'
 #-----------#  Failover to next
{{< /textdiagram >}}

It's useful to design the client logic as a finite state machine. The client cycles through three states:

* The client opens and connects its sockets, and then requests a snapshot from the first server. To avoid request storms, it will ask any given server only twice. One request might get lost, which would be bad luck. Two getting lost would be carelessness.

* The client waits for a reply (snapshot data) from the current server, and if it gets it, it stores it. If there is no reply within some timeout, it fails over to the next server.

* When the client has gotten its snapshot, it waits for and processes updates. Again, if it doesn't hear anything from the server within some timeout, it fails over to the next server.

The client loops forever. It's quite likely during startup or failover that some clients may be trying to talk to the primary server while others are trying to talk to the backup server. The Binary Star state machine handles this, hopefully accurately. It's hard to prove software correct; instead we hammer it until we can't prove it wrong.

Failover happens as follows:

* The client detects that primary server is no longer sending heartbeats, and concludes that it has died. The client connects to the backup server and requests a new state snapshot.

* The backup server starts to receive snapshot requests from clients, and detects that primary server has gone, so it takes over as primary.

* The backup server applies its pending list to its own hash table, and then starts to process state snapshot requests.

When the primary server comes back online, it will:

* Start up as passive server, and connect to the backup server as a Clone client.

* Start to receive updates from clients, via its SUB socket.

We make a few assumptions:

* At least one server will keep running. If both servers crash, we lose all server state and there's no way to recover it.

* Multiple clients do not update the same hash keys at the same time. Client updates will arrive at the two servers in a different order. Therefore, the backup server may apply updates from its pending list in a different order than the primary server would or did. Updates from one client will always arrive in the same order on both servers, so that is safe.

Thus the architecture for our high-availability server pair using the Binary Star pattern has two servers and a set of clients that talk to both servers.

{{< textdiagram name="fig62.png" figno="62" title="High-availability Clone Server Pair" >}}
#--------------------#                 #--------------------#
|                    |      Binary     |                    |
|       Primary      |<--------------->|       Backup       |
|                    |       Star      |                    |
+-----+--------+-----+                 +-----+--------+-----+
| PUB | ROUTER | SUB |                 | PUB | ROUTER | SUB |
'--+--+--------+-----'                 '-----+--------+-----'
   |       ^      ^                                      ^
   |       |      |                                      |
   |       |      |                                      |
   |       |      +--------------------------------------'
   |       |      |
   v       |      |
.-----+----+---+--+--.
| SUB | DEALER | PUB |
+-----+--------+-----+
|       Client       |
#--------------------#
{{< /textdiagram >}}

Here is the sixth and last model of the Clone server:

{{< examples name="clonesrv6" title="Clone server, Model Six" >}}

This model is only a few hundred lines of code, but it took quite a while to get working. To be accurate, building Model Six took about a full week of "Sweet god, this is just too complex for an example" hacking. We've assembled pretty much everything and the kitchen sink into this small application. We have failover, ephemeral values, subtrees, and so on. What surprised me was that the up-front design was pretty accurate. Still the details of writing and debugging so many socket flows is quite challenging.

The reactor-based design removes a lot of the grunt work from the code, and what remains is simpler and easier to understand. We reuse the bstar reactor from [Chapter 4 - Reliable Request-Reply Patterns](chapter4#reliable-request-reply). The whole server runs as one thread, so there's no inter-thread weirdness going on--just a structure pointer (<tt>self</tt>) passed around to all handlers, which can do their thing happily. One nice side effect of using reactors is that the code, being less tightly integrated into a poll loop, is much easier to reuse. Large chunks of Model Six are taken from Model Five.

I built it piece by piece, and got each piece working *properly* before going onto the next one. Because there are four or five main socket flows, that meant quite a lot of debugging and testing. I debugged just by dumping messages to the console. Don't use classic debuggers to step through ZeroMQ applications; you need to see the message flows to make any sense of what is going on.

For testing, I always try to use Valgrind, which catches memory leaks and invalid memory accesses. In C, this is a major concern, as you can't delegate to a garbage collector. Using proper and consistent abstractions like kvmsg and CZMQ helps enormously.

### The Clustered Hashmap Protocol {#The-Clustered-Hashmap-Protocol}

While the server is pretty much a mashup of the previous model plus the Binary Star pattern, the client is quite a lot more complex. But before we get to that, let's look at the final protocol. I've written this up as a specification on the ZeroMQ RFC website as the [Clustered Hashmap Protocol](http://rfc.zeromq.org/spec:12).

Roughly, there are two ways to design a complex protocol such as this one. One way is to separate each flow into its own set of sockets. This is the approach we used here. The advantage is that each flow is simple and clean. The disadvantage is that managing multiple socket flows at once can be quite complex. Using a reactor makes it simpler, but still, it makes a lot of moving pieces that have to fit together correctly.

The second way to make such a protocol is to use a single socket pair for everything. In this case, I'd have used ROUTER for the server and DEALER for the clients, and then done everything over that connection. It makes for a more complex protocol but at least the complexity is all in one place. In [Chapter 7 - Advanced Architecture using ZeroMQ](chapter7#advanced-architecture) we'll look at an example of a protocol done over a ROUTER-DEALER combination.

Let's take a look at the CHP specification. Note that "SHOULD", "MUST" and "MAY" are key words we use in protocol specifications to indicate requirement levels.

**Goals**

CHP is meant to provide a basis for reliable pub-sub across a cluster of clients connected over a ZeroMQ network. It defines a "hashmap" abstraction consisting of key-value pairs. Any client can modify any key-value pair at any time, and changes are propagated to all clients. A client can join the network at any time.

**Architecture**

CHP connects a set of client applications and a set of servers. Clients connect to the server. Clients do not see each other. Clients can come and go arbitrarily.

**Ports and Connections**

The server MUST open three ports as follows:

* A SNAPSHOT port (ZeroMQ ROUTER socket) at port number P.
* A PUBLISHER port (ZeroMQ PUB socket) at port number P + 1.
* A COLLECTOR port (ZeroMQ SUB socket) at port number P + 2.

The client SHOULD open at least two connections:

* A SNAPSHOT connection (ZeroMQ DEALER socket) to port number P.
* A SUBSCRIBER connection (ZeroMQ SUB socket) to port number P + 1.

The client MAY open a third connection, if it wants to update the hashmap:

* A PUBLISHER connection (ZeroMQ PUB socket) to port number P + 2.

This extra frame is not shown in the commands explained below.

**State Synchronization**

The client MUST start by sending a ICANHAZ command to its snapshot connection. This command consists of two frames as follows:

```
ICANHAZ command
-----------------------------------
Frame 0: "ICANHAZ?"
Frame 1: subtree specification
```

Both frames are ZeroMQ strings. The subtree specification MAY be empty. If not empty, it consists of a slash followed by one or more path segments, ending in a slash.

The server MUST respond to a ICANHAZ command by sending zero or more KVSYNC commands to its snapshot port, followed with a KTHXBAI command. The server MUST prefix each command with the identity of the client, as provided by ZeroMQ with the ICANHAZ command. The KVSYNC command specifies a single key-value pair as follows:

```
KVSYNC command
-----------------------------------
Frame 0: key, as ZeroMQ string
Frame 1: sequence number, 8 bytes in network order
Frame 2: <empty>
Frame 3: <empty>
Frame 4: value, as blob
```

The sequence number has no significance and may be zero.

The KTHXBAI command takes this form:

```
KTHXBAI command
-----------------------------------
Frame 0: "KTHXBAI"
Frame 1: sequence number, 8 bytes in network order
Frame 2: <empty>
Frame 3: <empty>
Frame 4: subtree specification
```

The sequence number MUST be the highest sequence number of the KVSYNC commands previously sent.

When the client has received a KTHXBAI command, it SHOULD start to receive messages from its subscriber connection and apply them.

**Server-to-Client Updates**

When the server has an update for its hashmap it MUST broadcast this on its publisher socket as a KVPUB command. The KVPUB command has this form:

```
KVPUB command
-----------------------------------
Frame 0: key, as ZeroMQ string
Frame 1: sequence number, 8 bytes in network order
Frame 2: UUID, 16 bytes
Frame 3: properties, as ZeroMQ string
Frame 4: value, as blob
```

The sequence number MUST be strictly incremental. The client MUST discard any KVPUB commands whose sequence numbers are not strictly greater than the last KTHXBAI or KVPUB command received.

The UUID is optional and frame 2 MAY be empty (size zero). The properties field is formatted as zero or more instances of "name=value" followed by a newline character. If the key-value pair has no properties, the properties field is empty.

If the value is empty, the client SHOULD delete its key-value entry with the specified key.

In the absence of other updates the server SHOULD send a HUGZ command at regular intervals, e.g., once per second. The HUGZ command has this format:

```
HUGZ command
-----------------------------------
Frame 0: "HUGZ"
Frame 1: 00000000
Frame 2: <empty>
Frame 3: <empty>
Frame 4: <empty>
```

The client MAY treat the absence of HUGZ as an indicator that the server has crashed (see Reliability below).

**Client-to-Server Updates**

When the client has an update for its hashmap, it MAY send this to the server via its publisher connection as a KVSET command. The KVSET command has this form:

```
KVSET command
-----------------------------------
Frame 0: key, as ZeroMQ string
Frame 1: sequence number, 8 bytes in network order
Frame 2: UUID, 16 bytes
Frame 3: properties, as ZeroMQ string
Frame 4: value, as blob
```

The sequence number has no significance and may be zero. The UUID SHOULD be a universally unique identifier, if a reliable server architecture is used.

If the value is empty, the server MUST delete its key-value entry with the specified key.

The server SHOULD accept the following properties:

* <tt>ttl</tt>: specifies a time-to-live in seconds. If the KVSET command has a <tt>ttl</tt> property, the server SHOULD delete the key-value pair and broadcast a KVPUB with an empty value in order to delete this from all clients when the TTL has expired.

**Reliability**

CHP may be used in a dual-server configuration where a backup server takes over if the primary server fails. CHP does not specify the mechanisms used for this failover but the Binary Star pattern may be helpful.

To assist server reliability, the client MAY:

* Set a UUID in every KVSET command.
* Detect the lack of HUGZ over a time period and use this as an indicator that the current server has failed.
* Connect to a backup server and re-request a state synchronization.

**Scalability and Performance**

CHP is designed to be scalable to large numbers (thousands) of clients, limited only by system resources on the broker. Because all updates pass through a single server, the overall throughput will be limited to some millions of updates per second at peak, and probably less.

**Security**

CHP does not implement any authentication, access control, or encryption mechanisms and should not be used in any deployment where these are required.

### Building a Multithreaded Stack and API {#Building-a-Multithreaded-Stack-and-API}

The client stack we've used so far isn't smart enough to handle this protocol properly. As soon as we start doing heartbeats, we need a client stack that can run in a background thread. In the Freelance pattern at the end of [Chapter 4 - Reliable Request-Reply Patterns](chapter4#reliable-request-reply) we used a multithreaded API but didn't explain it in detail. It turns out that multithreaded APIs are quite useful when you start to make more complex ZeroMQ protocols like CHP.

{{< textdiagram name="fig63.png" figno="63" title="Multithreaded API" >}}
         #--------------#
         |   Calling    |
         | Application  |
.------- +--------------+ --------.
:        |  Frontend    |         :
:        |   Object     |         :
:        +--------------+         :
:        |    PAIR      |         :
:        '------+-------'         :
:               |                 :
:               |                 :
:               v                 :
:        .--------------.         :
:        |    PAIR      |         :
:        +--------------+         :
:        |   Backend    |         :
:        |    Agent     |         :
:        +--------+-----+         :
:        | DEALER | SUB |         :
:        #--------+-----#         :
:                                 :
'---------------------------------'
            Clone class
{{< /textdiagram >}}

If you make a nontrivial protocol and you expect applications to implement it properly, most developers will get it wrong most of the time. You're going to be left with a lot of unhappy people complaining that your protocol is too complex, too fragile, and too hard to use. Whereas if you give them a simple API to call, you have some chance of them buying in.

Our multithreaded API consists of a frontend object and a background agent, connected by two PAIR sockets. Connecting two PAIR sockets like this is so useful that your high-level binding should probably do what CZMQ does, which is package a "create new thread with a pipe that I can use to send messages to it" method.

The multithreaded APIs that we see in this book all take the same form:

* The constructor for the object (<tt>clone_new</tt>) creates a context and starts a background thread connected with a pipe. It holds onto one end of the pipe so it can send commands to the background thread.

* The background thread starts an *agent* that is essentially a <tt>zmq_poll</tt> loop reading from the pipe socket and any other sockets (here, the DEALER and SUB sockets).

* The main application thread and the background thread now communicate only via ZeroMQ messages. By convention, the frontend sends string commands so that each method on the class turns into a message sent to the backend agent, like this:

{{< fragment name="connect-command" >}}
void
clone_connect (clone_t *self, char *address, char *service)
{
    assert (self);
    zmsg_t *msg = zmsg_new ();
    zmsg_addstr (msg, "CONNECT");
    zmsg_addstr (msg, address);
    zmsg_addstr (msg, service);
    zmsg_send (&msg, self->pipe);
}
{{< /fragment >}}

* If the method needs a return code, it can wait for a reply message from the agent.

* If the agent needs to send asynchronous events back to the frontend, we add a <tt>recv</tt> method to the class, which waits for messages on the frontend pipe.

* We may want to expose the frontend pipe socket handle to allow the class to be integrated into further poll loops. Otherwise any <tt>recv</tt> method would block the application.

The clone class has the same structure as the <tt>flcliapi</tt> class from [Chapter 4 - Reliable Request-Reply Patterns](chapter4#reliable-request-reply) and adds the logic from the last model of the Clone client. Without ZeroMQ, this kind of multithreaded API design would be weeks of really hard work. With ZeroMQ, it was a day or two of work.

The actual API methods for the clone class are quite simple:

{{< fragment name="clone-methods" >}}
//  Create a new clone class instance
clone_t *
    clone_new (void);

//  Destroy a clone class instance
void
    clone_destroy (clone_t **self_p);

//  Define the subtree, if any, for this clone class
void
    clone_subtree (clone_t *self, char *subtree);

//  Connect the clone class to one server
void
    clone_connect (clone_t *self, char *address, char *service);

//  Set a value in the shared hashmap
void
    clone_set (clone_t *self, char *key, char *value, int ttl);

//  Get a value from the shared hashmap
char *
    clone_get (clone_t *self, char *key);
{{< /fragment >}}

So here is Model Six of the clone client, which has now become just a thin shell using the clone class:

{{< examples name="clonecli6" title="Clone client, Model Six" >}}

Note the connect method, which specifies one server endpoint. Under the hood, we're in fact talking to three ports. However, as the CHP protocol says, the three ports are on consecutive port numbers:

* The server state router (ROUTER) is at port P.
* The server updates publisher (PUB) is at port P + 1.
* The server updates subscriber (SUB) is at port P + 2.

So we can fold the three connections into one logical operation (which we implement as three separate ZeroMQ connect calls).

Let's end with the source code for the clone stack. This is a complex piece of code, but easier to understand when you break it into the frontend object class and the backend agent. The frontend sends string commands ("SUBTREE", "CONNECT", "SET", "GET") to the agent, which handles these commands as well as talking to the server(s). Here is the agent's logic:

1. Start up by getting a snapshot from the first server
1. When we get a snapshot switch to reading from the subscriber socket.
1. If we don't get a snapshot then fail over to the second server.
1. Poll on the pipe and the subscriber socket.
1. If we got input on the pipe, handle the control message from the frontend object.
1. If we got input on the subscriber, store or apply the update.
1. If we didn't get anything from the server within a certain time, fail over.
1. Repeat until the process is interrupted by Ctrl-C.

And here is the actual clone class implementation:

{{< examples name="clone" title="Clone class" >}}

