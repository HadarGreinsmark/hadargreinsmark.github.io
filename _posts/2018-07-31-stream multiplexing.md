---
layout: post
title:  "Moving from TCP to Stream Multiplexing"
---
In recent years, lots of efforts have been put into moving the web towards HTTP/2

Multi-streaming is the technique of giving a connection multipe independent streams within one connection. By giving the hosts the ability to send messages on different streams, it allows for the network and the reveiving host to process the messages in any order. I.e. messages on independent streams can be processed concurrently. Messages sent in the same stream in contrary has to be processed in the order it was sent. What does such an ease in the processing requirement allow?

# Drawbacks of TCP
Internet have adopted TCP as the primary transmission protocol for streams between two hosts. Though the IP networks aren't really designed for transferring a stream of bytes - but to transmit packets of a practical size - the TCP mechanism gives the illusion of a byte stream. If a packet gets lost, TCP stalls processing of subsequent packets on the receiving host, until the lost packet has been retransmitted. UDP could be used for processing subsequent packets at once, thus avoid stalling. But most applications gets much simpler by relying on an illution of a stream.

Another problem arising when having many connections between two IP addresses, is the fact that each end’s port number is reserved over a time. This is to make sure old packets in the network won’t disturb a new connection on the same port pair. For example, a TCP connection form 1.1.1.1:60100 to 2.2.2.2:80 is in Linux blocked for reuse for 1 minute. New connections have to use another port. 

## First efforts with SCTP
In year 2000, the IETF standardization institute defined the SCTP  transport layer protocol as an alternative to TCP. Among many introduced features, stream multiplexing is one. Each packet sent through the network may contain several data chunks, where each chunks is associated with a stream. The client and server negotiate on the amount of streams should be used, but there's a limit of 65 536 streams at max. If a packet gets dropped, only streams with data chunks in the dropped packet have to wait for retransmission of the packet. 

## Virtualizing streams with HTTP/2
Stantardized in 2015 by IETF, HTTP/2 introduces stream multiplexing in the application layer 7. That way multiple streams is "virtualized" within one TCP connection. Each stream can have a priority as well as a dependency on another stream. I.e. if a client sends two concurrent requests and want the first request to be delivered before the second one, the second stream can refer to the first stream as a dependency. Priorities and stream dependencies give hints to the server in which order to deliver responses.

## Redefining the stack with QUIC
In an effort by Google to replace TCP without having to push for support for new transport protocols on the Internet, they are experimenting with a protocol called QUIC. In order to be easily deployed, QUIC is sending its data segments on top of UDP which is commonly supported by switches and firewalls on the Internet. QUIC partly overlaps with HTTP/2 as both are introducing the notion of streams. Though, HTTP/2's stream handling are more complex as it has stream priorites and stream dependencies.

# Benefited Use Cases
Multi-streaming enables the client to fire off multiple requests at the same time, so the server can prepare all of them in one round-trip time. HTTP 1.1 is notorious for having to sequentially request and wait for the response, before sending a second request. Isolating each request-response pair in its own stream works well if the messages are small, as the round-trip time then is of a greater factor than the data transmission time.

Two nodes with many frequent procedures between them benefits from quickly establish a new stream for performing a remote call. One approch is using a connection pool of active streams, that procedures can queue for getting access to. A common use case is web servers sending SQL queries to a database for concurrent web users. With a multi-streaming approach, the number of concurrent streams is instead depends on back pressure from the server, rather than how many connections the client are configured to create.
One example of managing many concurrent streams is in telephone networks. For each mobile phone connecting to a radio station, the radio station have to communicate with the network provider's central database for authorization and for sending meta-data like the phone's location. The radio station and the central database might have tens-of-thousands of concurrent phones being active. In 4G/LTE this is handled by using SCTP, where each phone gets its own isolated stream. 

## New Problems
Though multi-streaming makes stream creation between client and server more lenient, the multiplexed streams are limited in communication only between the client and server endpoints. In order to communicate with a second server, a completely new connection has to be established or the first server has to route the traffic to the second server. The second approach is what we call a proxy or gateway. This way, the client and gateway server can maintain their connection. The gateway server can do an informed decision on where to route each specific request based on which backend service is needed and do load balancing. The Envoy proxy (LINK!!!) is one example of such a “de-multiplexing” for HTTP/2. (PICTURE!!!) (SHARDING???). 
Thi s model puts more pressure the gateway server as it has to keep track of active clients and available backend servers in order to make an informed routing decision. If the backend network is loaded
dc,cache servers
It also takes a similar role as a one-to-many NAT (Network Address Translation)
