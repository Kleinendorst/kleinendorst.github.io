---
layout: post
title:  "Message brokers"
date:   2017-10-04 16:00:00
categories: main
comments: true
---
When microservice architectures become bigger, it's important to think about the scalability of the system.
When the processing power of the architecture isn't sufficient, it can be scaled up vertically by adding 
more RAM or Faster CPU's. This vertical approach is relatively easy because we don't have to change our
code. At some point the limitations of vertical scaling are met, and at this time, 
it's time to think about horizontal scaling.

> Another disadvantage is downtime caused by vertical scaling.

Horizontal scaling is achieved by adding more servers to perform computations.
Horizontal scaling is considerably more difficult than vertical scaling because the 
architecture must allow horizontal scaling[^1].

In order to achieve horizontal scaling, microservices architectures are often deployed
which make scaling easier[^2]. Indirect messaging is used to ensure that services are decoupled
and [integration patterns](http://www.enterpriseintegrationpatterns.com/) are used in order to achieve horizontal scalability.

In modern microservice based architectures, messaging between services is proxied through message brokers. In this article we will investigate the advantages, differences and need for different messaging platforms. 

## Message brokers
A message broker is a service deployed within the cluster. This service routes messages between services. 
Messages sent between services and the broker are often send through an open protocol[^3], 
and automatically encoded/decoded by client libraries. Often times libraries are provided for multiple popular programming languages, 
which negates the need for language uniformity throughout the distributed system.

There are a lot of platforms on the market and in order to choose, knowing the differences is important. 
Let’s look at some common differences between messaging platforms:

1. **Guarantees**: Some message queues guarantee messages are distributed FIFO while others do not. 
Some queues even guarantee at-once-delivery, at-most-once-delivery or other options.

2. **Persistence**: Most message brokers have the ability to temporarily persist messages to disk 
which makes sure that no messages are lost even if the broker itself fails. 
Apache Kafka takes this to the extreme with its ability to persist messages indefinitely. 
This makes Kafka a suitable solution for [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html).

3. **Distributed**: Some messaging queues have the ability to run multiple broker instances in sync. 
Distribution eliminates single points of failure, thus increasing reliability. 

4. **Cloud**: It is possible to let 3ᵈ parties host broker. 
Services like Amazon SQS and iron.io are  available *only* as cloud platforms, 
while popular open-source solutions like RabbitMQ have [hosted solutions](https://www.cloudamqp.com/). 
For most platforms you’ll have to host your own infrastructure.

5. **Protocols**: Some queues use standardized protocols, whilst others do not.

6. **Throughput**: Some queues are more focused on reliability and features 
while others prioritize low latency and high throughput.

> performance is dependent on a number of factors, for each service we'll use (relatively): LOW, MEDIUM or HIGH as vectors.
> Also note that cloud provisioned services have additional network latency.

7. **Pricing**: Most messaging solutions are open-source and free to implement, 
Pricing is mostly relevant for hosted solutions (and how that compares to hosting it yourself).

## Platforms
### Apache ActiveMQ
[ActiveMQ](http://activemq.apache.org/) is a mature and popular open source messaging platform. 
It features client libraries for a wide array of modern programming languages and
communicates using open protocols. Messages can be persisted in a relational database via the JDBC tools. 
ActiveMQ also supports distributed deployment, although limited.

*Advantages*: stable and proven, client libraries for different languages, 
open messaging protocols, advanced integration patterns build in, 
REST api for retrieving performance metrics, open-source

*Disadvantages*: One of the slower queues, 
client library is part of JMS specification there is also Spring integration but no mature standalone implementation

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">done</i>via JDBC | <i class="material-icons">done</i> |<i class="material-icons">clear</i> | OpenWire, Stomp, AMQP, MQTT | [LOW](http://activemq.apache.org/performance.html) |   -   |

### Amazon Simple Queue Service
[Amazon’s simple queue service](https://aws.amazon.com/sqs/) is a hosted 
queuing solutions which is easy to setup. 
SQS was the first service added in Amazon’s AWS. 
The main selling point for SQS is it’s elimination of operational costs of running own hardware. 

*Advantages*: simple to setup, secure, scales elastically, AWS integration, at least and exactly once guarantees

*Disadvantages*: closed-source, self-hosting is not possible, not very feature rich

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">clear</i> | - | <i class="material-icons">done</i> | HTTPS, TLS | [MEDIUM](https://softwaremill.com/amazon-sqs-performance-latency/) |   [~$.40-$.50 per million requests.](https://aws.amazon.com/sqs/pricing/)  |

### Apache Apollo
[Apache Apollo](https://activemq.apache.org/apollo/) is a message queue which was build from the foundations of Apache ActiveMQ. 
Apollo was made with performance and easy-of-use in mind.  Apollo scales exceptionally well on multi-core systems. 
It can communicate in a wide variety of open protocols and has modern web/REST support. 

*Advantages*: fast in multi-core setups, open protocols, open-source, REST endpoints

*Disadvantages*: lack of popularity, green-fields technology

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">done</i> | <i class="material-icons">done</i> | <i class="material-icons">clear</i> | OpenWire, Stomp, AMQP, MQTT | [HIGH](http://hiramchirino.com/stomp-benchmark/ec2-c1.xlarge/index.html) | - |

### Beanstalkd
[Beanstalkd](http://kr.github.io/beanstalkd/) is a lightweight messaging queue 
which uses its own protocol for message delivery. 
It has clients for most popular languages, and features disk persistence via binlog files (optional). 
Beanstalkd sees each running instance as only one active job queue, 
but can uses so called tubes to support multiple job queues.

*Advantages*: lightweight, fast, easy to configure

*Disadvantages*: Almost no features, Clients and monitoring tools are all third party.

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">done</i> via Binlog | <i class="material-icons">clear</i> | <i class="material-icons">clear</i> | HTTP | [HIGH](https://www.quora.com/What-are-the-advantages-and-disadvantages-of-Beanstalkd-as-a-work-queue-How-have-peoples-experiences-been-with-Beanstalkd) | - |

### Iron MQ
[Iron MQ](https://www.iron.io/platform/ironmq/) is a cloud provider that also delivers queue services. 
The business model is exactly the same as with Amazon SQS, but the queue server
is more feature rich. 

Iron MQ sends messages via HTTP in JSON format and handles authentication. Library
wrappers are available for Go, Ruby, .NET, Java, PHP, Python an JavaScript(Node).
It features: monitoring via a web dashboard, high availability and data persistence.

*Advantages*: Libraries in some popular languages, Different types of queue's (error queues)
*Disadvantages*: Very expensive, latency introduced by being a cloud technology

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">done</i> | - | <i class="material-icons">done</i> | proprietary | [MEDIUM](https://www.iron.io/ironmq-v3-is-10x-faster-than-rabbitmq/) | [at least $24.99/wk](https://try.iron.io/pricing-mq-monthly/) |

### RabbitMQ
[RabbitMQ](https://www.rabbitmq.com/) is made by [Pivotal](https://pivotal.io/) and is a very popular open-source message broker.
RabbitMQ runs on the Erlang VM, and is programmed using the actor model. RabbitMQ is over
10 years old and is considered to be very stable and feature rich.

RabbitMQ provides libraries for most modern languages. It also sends messages via AMQP which is 
one of the open standards, other open protocols are also available. 

*Advantages*: Different guarantee and routing strategies to choose from, can be distributed, open messaging
standards, management UI and tools 

*Disadvantages*: apparently not suited for real-time messaging[^4], not the fastest tool on the market

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">clear</i> | <i class="material-icons">done</i> | <i class="material-icons">clear</i> | AMQP, STOMP, MQTT, HTTP | [MEDIUM](https://www.rabbitmq.com/blog/2012/04/25/rabbitmq-performance-measurements-part-2/) | FREE or [variable prices in the cloud](https://www.cloudamqp.com/plans.html) |

### Apache Kafka
[Apache Kafka](https://kafka.apache.org/) calls itself a distributed streaming platform. The model used by Kafka
is slightly different than other message brokers, but still very feasible to use as one. 

Normal message brokers let you choose either a queuing or publish-subscribe models, 
Apache Kafka combines both[^5], levering advantages of both. This ability makes it possible
to create very resilient systems (through publish-subscribe) that also scale well (through queuing).

Apache Kafka defines one or more brokers as a cluster. 
It's possible that Kafka sends acknowledgments to publishers
once messages are successfully replicated throughout the cluster.
This ensures that messages are not lost when a Kafka node dies. 
New nodes can be easily added to the cluster and load balancing is abstracted away from the developer.

*Advantages*: combines queuing and publish-subscribe, extremely fast, distributed

*Disadvantages*: Only Java API's officially available (unofficial libraries in more language), 
adds complexity if you don't need it's features.

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| <i class="material-icons">done</i> | <i class="material-icons">done</i> | <i class="material-icons">clear</i> | proprietary | [HIGH](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines) | - |

### ZeroMQ
[ZeroMQ](http://zeromq.org/) isn't really a broker system, but is still relevant on our list.
ZeroMQ can be seen as a library that layers on top of standard TCP sockets. Additional features
provided by ZeroMQ are things like: [request/response (with load balancing)](http://api.zeromq.org/2-1:zmq-socket#_request_reply_pattern), [publish-subscribe](http://api.zeromq.org/2-1:zmq-socket#_publish_subscribe_pattern), 
and more.

*advantages*: extremely fast, handles receiving and sending messages asynchronously, doesn't introduce
a single point of failure (broker), open-source, multi-platform, very simple API
	
*disadvantages*: considerably more effort to implement, integration patterns should
be implemented as part of the application, otherwise it's just direct communication.
 
In the [last article]({% post_url 2017-10-03-microservice-message-solutions %}) we looked at the disadvantages of direct communication.

| Persistence | Distributed | Cloud | Protocols | Throughput | Price |
|-------------|-------------|-------|-----------|------------|-------|
| ? | ? | <i class="material-icons">clear</i> | INPROC, IPC, MULTICAST, TCP | [HIGH](http://zeromq.org/whitepapers:measuring-performance) | - |

## Conclusion
In this article we just looked at a few of the [options available](http://queues.io/).
When looking for a messaging platform it's important to think about the requirements and choose accordingly.
This is the end of these blog posts about communication technologies to use for microservice systems.

-----
[^1]: [Scaling](https://en.wikipedia.org/wiki/Scalability)
[^2]: Because microservices separate smaller software components from each other, scaling these smaller parts is easier. This improved granularity enables us to more precisely scale specific parts of the application. 
[^3]: Like [STOMP](https://stomp.github.io/) and [AMQP](https://www.amqp.org/)
[^4]: [see](http://objectzen.com/2016/10/14/not-use-rabbitmq-real-time-messaging/)
[^5]: More information can be found [here](https://kafka.apache.org/intro#kafka_mq)