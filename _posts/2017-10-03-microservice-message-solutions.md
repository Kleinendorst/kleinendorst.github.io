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
Akka Clustering, which allows us to connect actors between different services and finally enterprise service buses.

## REST
The most simple form of inter-service communication would be direct communication with REST.
Each service provides endpoints to which other services can talk. When a service sends a message to 
another service it creates an HTTP client object that sends the message and expects a synchronous response back.

One advantage of REST (which uses HTTP) is that we get a response from the receiver. This response
can hold the result of a task or simply if it succeeded. Receiving the response directly reduces complexity 
because we don't have to worry about managing requests sent and responses received. 

Another advantage of REST is interoperability. REST communication happens mostly via
open, text-based, standards such as XML and JSON. Libraries for parsing these messages, and receiving HTTP requests
is available for most languages. 

> Marshalling and unmarshalling of text formats like JSON is a costly
> operation. This results in lower throughput, but is not problematic
> in most systems. 

Inter-service communication via REST resembles the way we made 
[direct calls  between Java methods]({% post_url 2017-09-22-asynchronous-java %}#blocking-io) and shares its disadvantages,
because the synchronous nature: threads at multiple services need to block in order to handle a single task. 

Another disadvantage, which prevents this solution from scaling properly, is the direct coupling. Before
a message could be sent to a system, the sender needs to know the address and endpoints of the receiver. When we have a lot
of Inter-service communication, it quickly becomes hard to map out communication lines between individual services.
It also prevents us from easily deploying new instances of a service because we would have to register its address in all 
other services. 

> Service discovery is easier with tools such as [Consul](https://www.consul.io/) but adds complexity.

## Akka cluster
We looked at the actor system [earlier]({% post_url 2017-09-22-actor-model-introduction %}), which we implemented using [Akka actors](https://doc.akka.io/docs/akka/current/scala/actors.html).
Akka also provides a way for actors to communicate over the network using [Akka remoting](https://doc.akka.io/docs/akka/snapshot/scala/remoting.html).
Actors in the actor system communicate with each other through messages. The idea behind Akka remoting is that these messages might just as well travel across the network as opposed to just locally in the JVM. 

Akka remoting can start Actors on different hosts. The complexity of communicating over TCP is abstracted away from the developer, it is 
handled by Akka remoting behind the scenes. 
The actor addresses we saw [earlier]({% post_url 2017-09-22-actor-model-introduction %}#implementation) can also be 
used to get a reference to a remote actor. Before we can start using remoting, we need to add the 
[maven dependency](https://mvnrepository.com/artifact/com.typesafe.akka/akka-remote_2.11/2.5.6) and configure some properties[^1].

Let's see how we can easily create an `ActorRef` to a remote actor and how we create new actors on a remote.

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

The problem with using just Akka remoting is that actor and host addresses are hardcoded, this is where Akka cluster comes in.
To enable Akka cluster we should add another [maven dependency](https://mvnrepository.com/artifact/com.typesafe.akka/akka-cluster_2.11) 
and add some configuration. In these settings we have to provide a single host active in the cluster, after which 
other hosts in the cluster will be discovered using the [gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol). 

In order to start actors remotely via Akka cluster, we should first create a cluster aware router. 
A router is an `Akka Core` concept which can creates and manages a pool of actors to which work can be delegated. Different 
[strategies](https://doc.akka.io/docs/akka/2.5/java/routing.html#a-simple-router) for delegating messages are provided. 
By creating a special kind of router added by `Akka Cluster`, we can configure it in order to allow creating
children on other nodes in the cluster.

```java
// In our configuration we set the hosts associated with the path
Iterable<String> routeesPaths = Collections
        .singletonList("/weatherManager/weather-service");
ActorRef workerRouter = getContext().actorOf(
        new ClusterRouterGroup(new RoundRobinGroup(routeesPaths),
                new ClusterRouterGroupSettings(20, routeesPaths,
                        true, new HashSet<String>())).props(),
        "weatherServiceRouters");

workerRouter.tell(new RequestWeather("Utrecht"), getSelf());
workerRouter.tell(new RequestWeather("Eindhoven"), getSelf());
workerRouter.tell(new RequestWeather("Amsterdam"), getSelf());
workerRouter.tell(new RequestWeather("Groningen"), getSelf());
```

In this example each message will be sent in round-robin style over the 20 actors we create.
Notice that the `workerRouter` is an actor itself, which will automatically forward messages
it receives to child actors it manages.

### Using Akka cluster
Akka cluster is a nice way to enable our actor system as a whole to not only utilize 
all threads on our computer, but also threads of multiple computers in a cluster.

Creating systems with distributed actors is extremely scalable, but it doesn't 
decouple our services as a whole, which was the point of creating microservices. Another disadvantage is that it adds loads of complexity,
especially when the existing services don't use actors. 
It's quite a commitment to use Akka cluster: all microservices need to be written in a JVM language to use Akka Clustering. 
Also in order to reap the benefits of using Akka cluster, all our services should use actor systems.

## Enterprise service buses
Enterprise service buses are services that are deployed within our existing infrastructure.
Such a service is called a message broker, because it routes received messages to the destination service. 

Because inter-service communication is proxied through our message broker, the services only has to
know about the network address of the broker. This decouples services. The brokers define
different strategies for routing messages, they can for example employ a round-robin strategy. 
New instances of services can be added easily, they just have to register to the 
broker in order to take workload of its peers. 

Just like with actors, asynchronous messaging is used for communication.
Services don't have to block. Failure strategies are configured at the broker, which can for example ask for acknowledgments from the receiving service. Because the broker gives some delivery guarantees, the services themselves don't have to implement this logic. 

Another advantage of using message brokers is that they are language agnostic.
The payload of messages in most brokers consists only of binary data. The serialization strategy for messages
is up to the developer, they can for example serialize to JSON using common encodings like UTF-8, or use 
[Protocol Buffers](https://developers.google.com/protocol-buffers/).

By letting each message flow through the message broker, we create a single point of failure. 
Some message brokers, like [Apache Kafka](https://kafka.apache.org/) and [Pivotal RabbitMQ](https://www.rabbitmq.com/),
offer distributed deployment which makes this single point highly available.

Placing an ESB system in the architecture increases complexity in a system, en should thus only be considered
when dealing with systems that have lots of inter-service communication.

------------
[^1]: Remoting configuration can be found [here](https://doc.akka.io/docs/akka/snapshot/java/remoting.html#remote-configuration)