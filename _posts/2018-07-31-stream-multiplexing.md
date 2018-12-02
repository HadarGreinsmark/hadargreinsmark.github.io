---
layout: post
title:  "Moving from TCP to Stream Multiplexing"
---
<img alt="Service de-multiplexing" src="{{site.baseurl}}/assets/stream-multiplexing.png" style="float: right;" />
Multi-streaming is the technique of giving a connection multiple independent streams within one connection. By giving the hosts the ability to send messages on different streams, it allows for the network and the receiving host to process the messages in any order. I.e. messages on independent streams can be processed concurrently. Messages sent in the same stream in contrary has to be processed in the order it was sent. What does such an ease in the processing requirement allow? 



# Drawbacks of TCP
Internet have adopted TCP as the primary transmission protocol for streams between two hosts. The IP networks deliver a stream of bytes by splitting them into a series of packets. If one packet gets dropped in the series, all subsequent packets has to wait for that packet to be retransmitted so the receiving host process each packet in order. This gives the programmer an abstraction of the two hosts being connected with each other by a single stream. Though being simple, it also forces applications to send and receive packets in a defined order. If a packet in the sequence gets dropped, the receiving host is prohibited from processing subsequent packets until retransmit is finished. One could open a second TCP stream between the hosts to avoid stopping all progress if packed drop occurs in one of the connections. Though, this gives overhead as each new stream has to initialize a new TCP handshake and do a slow start to avoid congestion in the network.

In HTTP/1.1 this gets even worse. HTTP is designed to reserve the stream until each request and response pair is finished. A second request have to wait for the first to finish, leading to poor usage of the available stream throughput. Modern browsers tries to go around this problem by starting multiple TCP connections in parallel, leading to another TCP handshake as well as another encryption key exchange if using TLS. A similar strategy used between a web-backend and a database, is to maintain a connection pool of active TCP connections. When the web-backend receives a web request, it asks for an idle connection from the pool. Such an approach reduces request latency, but also requires both hosts to maintain the active pool with keep-alives.

# First efforts with SCTP
As TCP was limited in having only one byte stream per connection along with other disadvantages, the IETF standardization institute in 2000 defined SCTP as an alternative to TCP. One new feature -- stream multiplexing -- gives any of the hosts the ability to start a new stream on the already existing connection. As no new handshake or slow start is needed, stream creation is cheap. Applications can therefore afford to create a new stream per transaction or user. If a packet in one of the streams gets dropped, all other independent stream activities continues.

Although SCTP have been along for a while, it hasn't reached full scale on the Internet. What it has reached though is the telecommunication backhaul. When a mobile phone connects to a cell tower, it has to authenticate the user, access the encryption key and know its promised data rate. In 4G/LTE, this is done through a central controller that communicates with cell towers using the SCTP based `s1-MME` interface. As each phone is independently controlled, every phone is given its own SCTP stream.

# Virtualizing Streams with HTTP/2
Standardized in 2015 by IETF, HTTP/2 introduces stream multiplexing in the application layer. Rather than replacing TCP, multiple streams are "virtualized" within one TCP connection. Stream multiplexing enables the client to fire off multiple requests at the same time, so the server can prepare all of them in one round-trip time. HTTP 1.1 is notorious for having to sequentially request and wait for the response, before sending a second request. 

Each stream can have a priority as well as a dependency on another stream. I.e. if a client sends two requests on different streams and want the response to the first request to be delivered before the second one, the second stream can refer to the first stream as a dependency. Both *priorities* and *stream dependencies* can be seen as the client giving hints to the server in which order to deliver responses on the network; the server could have its own preferred delivery order.

# Moving Quickly from TCP to QUIC
In an effort by Google to move the stream multiplexing from HTTP/2 to the transport layer, they're experimenting with a protocol called QUIC. In order to be easily deployed as well, QUIC is sending its data segments on top of UDP that is commonly supported by switches and firewalls on the Internet. QUIC partly overlaps with HTTP/2 as both are introducing the notion of streams, though QUIC can't give hints on preferred delivery order. In contrast to SCTP, QUIC performs *flow control* both on the entire connection as well as on each single stream. Each stream has its own buffer, and its size limits how much data can be sent before the sender has to wait for the stream's buffer to be processed. A clogged stream therefore can't block other streams from making progress.

# New Problems
Though multi-streaming makes stream creation more lenient, the multiplexed streams are limited in communication only between the two endpoints. In order to communicate with another endpoint, a completely new connection has to be established or one endpoint has to route the traffic to the third endpoint. The second approach can be seen in many data centers where all external connections goes through gateway servers, that in turn routes it to the right service internally. 

![Service de-multiplexing]({{site.baseurl}}/assets/service-de-multiplexing.png)

The [Envoy proxy](https://www.envoyproxy.io/) is one example of such a “de-multiplexing” for HTTP/2, that routes external requests to the right server. One could argue though that this violates the end-to-end principle of doing as much work as possible on the endpoints rather than on intermediate nodes connecting them. In this case, the gateways split the network into the external and internal part potentially leading to . 
!Consul


# Conclusion
Stream-multiplexing enables the developer to cheaply perform concurrent transactions without worrying about blocking or connection pooling. Previously, communication between two hosts have been abstracted to a stream of sequential transactions. Now, it's being abstracted to multiple streams of concurrent transactions. But limiting the client to a single connection also limits its communication to one endpoint; something that might not play well with todays distributed data centers.
