---
layout: post
title:  "Microservices - Message solutions"
date:   2017-10-03 15:00:00
categories: main
comments: true
---

In the last articles we looked at messaging between software entities in a single running process.
When using a microservice based architecture, we also need to think about the way messages are
delivered between our services. 

In this article we will discuss the different ways of implementing messaging: direct communication via REST, 
Akka HTTP, which allows us to connect actors between different services and finally message brokers.

## REST
The most simple form of inter-service communication would be direct communication with REST.
Each service provides endpoints to which other services can talk. When a service sends a message to 
another service it creates an HTTP client object that sends the message and expects a synchronous response back.

One advantage of REST (which uses HTTP) is that we get a response from the receiver. This response
can hold the result of a task or simply if it succeeded. Receiving the response directly reduces complexity 
because we don't have to worry about managing requests send and responses received. 

Another advantage of REST is interoperability. REST communication happens mostly via
open, text-based, standards such as XML and JSON. Libraries for parsing these messages, and receiving HTTP requests
is available in most languages. 

> Marshalling and unmarshalling of text formats like JSON is a costly
> operation. This results in low throughput systems, but is not problematic
> in most systems. 

Inter-service communication via REST resembles the way we made 
[direct calls  between Java methods]({% post_url 2017-09-22-asynchronous-java %}#blocking-io) and shares its disadvantages.
Threads at multiple services need to block in order to handle a single task.

Another disadvantage, which prevents this solution from scaling properly, is the direct coupling. Before
a message could be send to a system, the sender needs to know the address of the receiver. When we have a lot
of Inter-service communication, it quickly becomes hard to map out communication lines between individual services.
It also prevents us from easily deploying new instances of a service because we would have to
register its address in all other services. 

> Service discovery is easier with tools such as [Consul](https://www.consul.io/) but still requires work in your services.

## Akka Cluster
