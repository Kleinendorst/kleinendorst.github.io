---
layout: post
title:  "Microservices - Message solutions"
date:   2017-10-03 15:00:00
categories: main
comments: true
---

In the last articles we looked at messaging between software entities in a single running process (inside the JVM).
When using a microservice based architecture, we also need to think about the way messages are
delivered between our services. 

In this article we will discuss the different ways of implementing messaging: direct communication via REST, 
Akka Cluster and finally enterprise service buses.

## REST
The most simple form of inter-service communication would be direct communication with REST[^1].
Each service provides endpoints to which other services can talk. When a service sends a message to 
another service it creates an HTTP client object that sends the message and expects a synchronous response back.

One advantage of REST is that we get a response from the receiver. This response
can hold the result of a task or its status. Receiving the response directly reduces complexity 
because we don't have to worry about managing requests sent and responses received. 

Another advantage of REST is interoperability. REST communication happens via
open text-based standards, such as XML and JSON. Libraries for parsing these messages, and receiving HTTP requests
are available in practically all languages. 

> The Marshalling and unmarshalling of text formats like JSON and XML is a costly operation[^2].

Inter-service communication via REST is similar to the way way we made 
[direct calls  between Java methods]({% post_url 2017-09-22-asynchronous-java %}#blocking-io) and shares its disadvantages,
because of its synchronous nature threads at multiple services need to block in order to handle a single task.

> Although you could technically run requests asynchronously, this is often not done. 

Another disadvantage is that decoupling services is harder.
In order to send messages between services, the sender needs to know the address and endpoints of the receiver. When we have a lot
of Inter-service communication, it quickly becomes hard to map out communication lines between individual services.
It also prevents us from easily deploying new instances of a service because we would have to register its address in all 
other services. 

> Service discovery is possible with tools such as [Consul](https://www.consul.io/) but adds complexity.

## Akka cluster
We looked at the actor system [earlier]({% post_url 2017-09-22-actor-model-introduction %}), 
which we implemented using [Akka actors](https://doc.akka.io/docs/akka/current/scala/actors.html).
Akka also provides a way for actors to communicate over 
the network using [Akka remoting](https://doc.akka.io/docs/akka/snapshot/scala/remoting.html).
Actors in the actor system communicate with each other through messages. 
The idea behind Akka remoting is that these messages might just as well travel across the network 
as opposed to just locally in the JVM. 

Akka remoting can start Actors on different hosts. The complexity of communicating over TCP is abstracted away from the developer, 
it is handled by Akka remoting behind the scenes. 
The actor addresses we saw [earlier]({% post_url 2017-09-22-actor-model-introduction %}#implementation) can also be 
used to get a reference to a remote actor. 

Before we can start using remoting, we need to add the 
[maven dependency](https://mvnrepository.com/artifact/com.typesafe.akka/akka-remote_2.11/2.5.6) 
and configure [some properties](https://doc.akka.io/docs/akka/snapshot/java/remoting.html#remote-configuration).

Let's see how we can get an `ActorRef` to a remote actor and how we create new actors on a remote.

```java
// getting a reference to a remote actor
ActorRef weatherManager = getContext().actorSelection("akka.tcp://weather/user/weather-manager");
// creating an actor on a remote host
String address = "akka.tcp://weather/user/weather-manager";
ActorRef remoteActorRef = getContext().actorOf(MyActor.props()
			.widthDeploy(new Deploy(new RemoteScope(address))));

// communicating to the remote actor
remoteActorRef.tell("I am serialized behind the scenes", getSelf());
```

The problem with using just Akka remoting is that actor and host addresses are still hardcoded, 
this is where Akka cluster comes in.
To enable Akka cluster we should add another 
[maven dependency](https://mvnrepository.com/artifact/com.typesafe.akka/akka-cluster_2.11) 
and add additional configuration. In these settings we have to provide a single host active in the cluster, after which 
other hosts in the cluster will be discovered using the [gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol). 

In order to start actors remotely via Akka cluster, we should first create a cluster aware router. 
A router is an `Akka Core` concept, in which we create a router actor than can creates and manages a pool 
of actors to which work can be delegated. 
Different [strategies](https://doc.akka.io/docs/akka/2.5/java/routing.html#a-simple-router) for delegating messages are provided. 
By creating a special kind of router added by `Akka Cluster`, we can configure it in order to allow creating
children on other nodes in the cluster.

```java
// In our configuration we set the hosts associated with the path
Iterable<String> routeesPaths = Collections
        .singletonList("/weatherManager/weather-service");
ActorRef workerRouter = getContext().actorOf(
        new ClusterRouterGroup(new RoundRobinGroup(routeesPaths),
                new ClusterRouterGroupSettings(4, routeesPaths,
                        true, new HashSet<String>())).props(),
        "weatherServiceRouters");

workerRouter.tell(new RequestWeather("Utrecht"), getSelf());
workerRouter.tell(new RequestWeather("Eindhoven"), getSelf());
workerRouter.tell(new RequestWeather("Amsterdam"), getSelf());
workerRouter.tell(new RequestWeather("Groningen"), getSelf());
```

In this example each message will be sent in round-robin style over the 4 actors we create.
Notice that the `workerRouter` is an actor itself, which will automatically forward messages
it receives to child actors it manages.

### Using Akka cluster
Akka cluster is a nice way to enable our actor system as a whole to not only utilize 
all threads on a system, but also threads of multiple systems in the cluster.

Systems with distributed actors are extremely scalable, but they don't 
decouple services as a whole, which was the point of creating microservices. 
Another disadvantage is that it adds loads of complexity,
especially when the existing services don't use actors. 
All microservices need to be written in a JVM language to use Akka Clustering. 

## Enterprise service buses
Enterprise service buses (ESBs) are services that are deployed within the existing infrastructure.
Such services are called message brokers, and route received messages to destination services.

Because inter-service communication is proxied through message brokers, the services only have to
know about network addresses of the brokers. This decouples services. The brokers define
different strategies for routing messages, they can for example employ a round-robin strategy. 
New instances of services can be added easily, they just have to register to the
broker in order to take workload of its peers. 

Just like with actors, asynchronous messaging is used so services don't block. 
Failure strategies are configured at the broker, which can for example ask for acknowledgments from the receiving service.

Another advantage of using message brokers is that they are (mostly) language agnostic.
The payload of messages in most brokers consists only of binary data. The serialization strategy for messages
is up to the developer, they can for example serialize to JSON using common encodings like UTF-8, or use 
[Protocol Buffers](https://developers.google.com/protocol-buffers/).

A disadvantage of message brokers is that they present a single point of failure.
Some message brokers, like [Apache Kafka](https://kafka.apache.org/) and [Pivotal RabbitMQ](https://www.rabbitmq.com/),
offer distributed deployment in order to be more resilient. 

Placing an ESB system in the architecture increases complexity in a system, and should thus only be considered
when dealing with systems that have lots of inter-service communication.

## Conclusion
There are different ways to handle inter-service communication, each with its own advantages and disadvantages.
Sometimes it's worth the extra complexity for a more scalable solution, and sometimes the overhead causes 
more problems than it solves. We've seen that direct communication becomes 
harder to scale properly when our systems becomes bigger. 

Our knowledge of different methods of communication should be enough to decide which 
approach we should take. When we choose for a ESB, our research is not done: There are lots
of different messaging platforms. In the [next article]({{ page.next.url }}) 
we'll take a look at the differences between these message brokers.

---------
[^1]: Representational state transfer [Wikipedia](https://en.wikipedia.org/wiki/Representational_state_transfer) 
[^2]: [Protocol Buffers, a binary format by google, is made espascially to decode/encode faster.](https://developers.google.com/protocol-buffers/docs/overview#whynotxml)
