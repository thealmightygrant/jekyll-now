---
layout: post
title: Notes on High Performance Browser Networking
permalink: Notes-on-High-Performance-Browser-Networkin Notes-on-High-Performance-Browser-Networkingg
tags: [networking, http, performance]
---

I've started reading this book by [Ilya Grigorik](https://www.igvita.com/) that I found while strolling along in Twitter-land for smart/clever web engineers that I can learn from. I've always been really interested in the underpinnings of the web and networking. Now-a-days, terms like progressive web apps (PWAs) and performance (PERFORMANCE) and really really fast web app (RRFWAs) are commonplace, but I think that very few people working in the technology industry truly understand the technologies behind browser performance. I'm going to be one of those very few people :-D.

I'll turn this into a series of blog posts eventually. I think I'm going to call it "Networking for the Curious" or "Networking for the Dumb and/or Lazy". ;-) For now, it's just a bunch of notes, but if it's helpful to anyone, then enjoy reading them.
<br><br>

## Chapter 1: [Latency and Bandwidth](https://hpbn.co/primer-on-latency-and-bandwidth/)
<br>
Latency -- The time from the source receiving a packet to the destination receiving it
Bandwidth -- Maximum throughput of a logical or physical communication path
<br><br>
Latency Components:
<br><br>
Every system contains many sources/components that affect the time it takes for a message to be delivered. A router that relays messages between the client and server typically has the following contributing components:

Propagation Delay -- Amount of time required to travel from the sender to the receiver. A function of distance over speed.

Transmission Delay -- Amount of time to push all the packet's bits into the link. A function of the packet's length and data rate.

Processing Delay -- Amount of time to process the packet header, check for bit-level errors, and determine the packet's destination.

Queueing Delay -- Amount of time the packet is waiting in the queue until it can be processed.

The total latency is the sum of all of these delays. Propagation rate is usally some fraction of the speed of light and depends upon the medium that the signal travels through. Transmission delay is dictated by the available data rate of the transmitting link. 10 Mb transferred at 1Mbps takes 10s, and at 100Mbps takes .1s.

Once the packet arrives at the router, the router must examine the packet header to determine it's outgoing route. Much of this checking is done in hardware, so there are very minimal delays. Additionally, if the packets arrive at a faster rate to the router than it can handle, it must queue them inside an incoming buffer. This time in the incoming buffer is known as queueing delay.

An increase in distance between client and server means a larger propagation delay. The more intermediate routers in between the client and server, the more transmission and processing delay. The more traffic along the path, the higher the chance of queueing delay.

Fiber optic cable has a refractive index between 1.4 and 1.6, meaning that data can travel through it at about 200 million meters per second. From NYC to San Francisco, then takes 21ms one way and 42ms round trip. This means that this is the absolute minimum for a packet to travel from NYC to San Francisco, not taking into account the many other delays.

Humans will notice a delay at 100-200 milliseconds. After 300 milliseconds many will report the connection as sluggish. After 1 second, most humans will be doing or thinking of something else.

To succeed (as a web enginerd), network latency has to be carefully managed and be an explicit design criteria at all stages of development.

CDN services provide many benefits, but chief among them is distributing content around the globe to reduce propagation delay. By serving content from a place that is near a client any noticable latency can be significantly reduced.

There is also a major issue with the last few miles to get to clients. Significant latency is introduced by the last few miles of cables that are strung through cities/neighborhoods. Fiber (10-20ms), Cable (15-40ms), and DSL (30-65ms).

Interesting Note: fiber optics are not better than metal based wire for transmitting data because of speed. They have some disadvatanges in higher lifetime maintenance, electromagnetic interference, and signal loss. But, in general, fiber optic cables are better because of the ability to multiplex up to 400 signals. At a peak capacity of 171Gb/s, that's 70Tb/s of total bandwidth. It would require thousands of copper wires to perform at the same throughput.

<br><br>

## Chapter 2: [Building Blocks of TCP](https://hpbn.co/building-blocks-of-tcp/)
<br>
The Internet consists mainly of two protocols, IP and TCP. IP provides host-to-host routing and addressing. TCP provides the appearance (or abstraction) of a reliable network running over what is actually an unreliable channel. TCP/IP is commonly referred to as the Internet Protocol Suite. In 1981, the 4th version was published as two RFCs, RFC 791 -- the Internet Protocol and RFC 793 -- Transmission Control Protocol. TCP takes care of and hides retransmission of lost data, in-order delivery congestion control and avoidance, and data integrity.

When working with TCP, bytes that are received are guaranteed to be the same as the bytes that are sent and they will arrive in the same order. However, this does not mean that they will be received quickly.

HTTP does not require the use of TCP as its transport protocol. This data could be delivered via a datagram socket or UDP (User Datagram Protocol) or any other transport protocol. However, the vast majority of internet traffic is sent via TCP.

Before the client and server can exchange data, they must agree on the starting packet sequence numbers and some other connection specific variables. The sequence numbers are picked randomly on both sides for security reasons.

SYN -- client picks a random sequence number and sends a SYN packet. This may contain other flags and options.

SYN ACK -- the server increments this number by one and appends its own random number along with some flags and options.

ACK -- the client increments both random numbers (from SYN and SYN ACK) by one and completes the handshake, by sending the ACK packet back to the server.

Once the ACK packet has been received, the client can immediately start sending data to the server. The server cannot send anything to the client until this ACK packet has been received. Therefore, __every new TCP connection requires at least one roundtrip's worth of latency (plus an additional single leg for the ACK packet)__.

But, every connection does not have to be a *new* connection...connection reuse is a critical optimization for any application running over TCP.

### TCP Fast Open

TCP Fast Open (TFO) is a mechanism that allows for data transfer to occur within the SYN packet. There are limits on the payload that can be carried, only certain types of requests can be sent, and it requires the use of a cryptographic cookie. This can decrease overall latency as much as 15% and in high latency situations, where the handshake is much more noticeable, as much as 40%.

### Flow Control

Flow control prevents the sender from overwhelming the receiver with data that they can't process. This may be because the receiver is busy, under heavy load, or may only be willing to allocate a fixed amount of buffer space. To address this, each side of a TCP connection advertises a receive window (rwnd) so that the buffer space available to hold incoming data is well known.

At the start of a TCP transmission, both sides initiate rwnd values by using system defaults. A typical website will stream data from the server to the client. The client's window is then the bottleneck, but for videos or image uploads, the server receiving window could become the limiting factor.

If for some reason, either side can't keep up, then it advertises a smaller window size in the ACK packet. This window size is specified in 16 bits, but recent additions have provided a window scaling option that left-shifts the 16 bit window size to increase the maximum size from 65KB to 1GB.

Issues can occur as a result of the network limitations in bandwidth between the client and server. Such as when a new connection opens, which would cause the available bandwidth for the original connection to shrink. This causes the data to pile up at an intermediate gateway and leads towards the "stuck" packets being dropped before reaching the client.

Several algorithms were proposed to combat these issues: slow-start, congestion avoidance, and fast recovery.

#### Slow-start

Slow-start is an algorithm where the server sets the size of the congestion window (cwnd) at a conservative value and increases from there. The maximum amount of data that is in flight (not ACKed) between the client and the server is the minimum of the rwnd and the cwnd variables. This value is 10 network segments in a modern server. For every received ACK, the slow-start algorithm increments the cwnd size by one segment -- for every ACKed packet two new packets can be sent.

It is important to keep in mind that every application must go through this algorithm. Every connection must go through the slow-start phase. The full capacity of a link cannot be used until slow-start has agreed that it can be used.

A data segment has the size of 1460 bytes. MTU, maximum transmission unit, is the maximum Ethernet v2 packet size (1500 bytes), IP overhead takes 20 of these bytes and TCP overhead takes another 20 bytes.

Each round trip provides the ability to increase the number of packets in the air by a factor of two. Decreasing the roundtrip time between the client and server decreases this startup time. For long running connections, there are no issues, but for connections that are less than a few hundred milliseconds, the maximum congestion window size is never reached.

#### Slow-start Restart

TCP also implements a slow-start restart in order to reset the congestion window of a connection after it has been idle for a defined period of time. This is to avoid congestion while the connection has been idle. For long-lived TCP connections, this can have a significant negative impact. On servers, SSR is typically disabled to improve performance of long-lived HTTP/TCP connections.

#### Congestion Avoidance

TCP is specifically designed to use packet loss as a feedback mechanism to regulate its performance. Therefore, packet loss is a given. Slow-start will continue to double the amount of segments being sent until one of several things happens: the receiver's flow control window is hit, a system-configured congestion threshold (ssthresh) window is hit, or until a packet is lost. If a packet is lost, then the congestion avoidance algorithm takes over.

The congestion avoidance algorithn assumes that packet loss is the result of network congestion (i.e. somewhere there is a router or link that was forced to drop a packet so the window size should be reduced to decrease packet loss).

An example of this is Multiplicative Decrease and Additive Increase, where each time that packet loss occurs, the congestion window size is halved and then slowly increased. This has seen to be too conserverative and been replaced by alternatives such as Proportional Rate Reduction (PRR).

#### Bandwidth Delay Product

Because of congestion control and avoidance mechanisms within TCP, the optimal sender and receiver window sizes must vary based on the roundtrip time and the target data rate between them. The current receive window sizes are communicated in every ACK communication (after a handshake or after data is sent). If either the sender or the receiver exceeds the maximum amount of data segments "in the air" specified in the rwnd/cwnd, then it must stop and wait for some data to be ACKed.  The time that they might have to wait is determined by the roundtrip time between the two. This is known as the Bandwidth-delay product, the product of a data-link's capacity and its end-to-end delay. This is the maximum amount of unackknowledged data that can be inflight at any point in time.

If either sender or receiver is forced to stop and wait, then this will create gaps in the data flow. This will limit maximum throughput/bandwidth. The window sizes should be made just big enough so that each side can continue sending data until an ACK arrives back from a previous packet -- no gaps implies maximum throughput. The optimal window size depends upon the roundtrip time!

If rwnd is set to be 16KB, then with a 100 millisecond roundtrip time (completely reasonable), the maximum throughput will be 1.31Mbps regardless of the available bandwidth.

The ideal window size to saturate a 10Mbps connection is 122.1 KB. Window scaling is important for streaming videos!

Bad network weather, a saturated peer advertising a low receive window, explicit traffic shaping, or high packet loss resetting the congestion window can all negatively affect throughput.


### Is TCP the best choice?

Packet error-checking and correction, in-order delivery, retransmission of lost packets, flow control, congestion control, and congestion avoidance all combine to make TCP a great method of transport for most applications. However, in-order and reliable packet delivery create considerable overhead that may not always be necessary in some applications.

If one packet is delayed in transit, then the subsequent packets are held in the receiver's TCP buffer until the packet has been retransmitted and put into the correct position. The application has no access to the transport layer (typically), but it can see a delay when trying to access the data. This is known as TCP head-of-line (HOL) blocking.

No packet reordering or reassembly required at the application level! Buuuuuuuut...latency becomes unpredictable. This unpredictability is known as "jitter".

Some applications may not need reliable or in order delivery. If every packet is a standalone message, then in-order delivery becomes unnecessary. If every message overrides the previous one, then who cares if some of them are unreliable (like for a movie...or a game). TCP provides no such configuration. Everything is sequenced and delivered in order.

### Optimizing for TCP

The best way to optimize TCP is to tune how TCP senses the current network conditions and adapts its behavior based on the type and the requirements of the layers below and above it.

Core takeaways:

-- TCP three-way handshake introduces a full roundtrip of latency.<br>
-- TCP slow-start is applied to every new connection.<br>
-- TCP flow and congestion control via rwnd/cwnd regulate throughput of all connections.<br>
-- TCP is generally limited by the roundtrip connection time between receiver and sender.<br>
-- In most cases, latency and not bandwidth is the bottleneck for TCP.<br>

Upgrading hosts to their latest system version (i.e. kernels in Linux) is a great way to ensure that the latest and greatest TCP advantages are available to web applications.

Increasing TCP's initial congestion window to the maximum value, disabling slow-start restart, enabling window scaling, and enabling TCP fast open are some good things to check out and possibly implement on application servers.
<br><br>

## Chapter 3: [Building Blocks of UDP](https://hpbn.co/building-blocks-of-udp/)
<br>
The UDP, or User Datagram Protocol, came about right after the time that the TCP/IP RFC split happened. The UDP protocol is sometimes described as a null protocol because of all of the features that it omits.

Datagram -- a self-contained independent entity of data carrying sufficient information to be routed from the source to the destination without reliance on earlier exchanges.

Packet -- any formatted block of data.

Sometimes, datagram and packet are used interchangeably, but datagrams generally refers to packets delivered via an unreliable service. No delivery guarantees, no failure notifications.




[TCP Tuning for HTTP](https://hpbn.co/http-tcp),
[Optimizing TLS over TCP with nginx](https://blog.cloudflare.com/optimizing-tls-over-tcp-to-reduce-latency/), more links to come...

Take it easy,<br>
From your buddy,<br>
Grant
