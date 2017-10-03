---
layout: post
title:  "Reactive streams"
date:   2017-09-29 13:15:00
categories: main
comments: true
---
> Source code for this article can be found on [Github](https://github.com/Kleinendorst/reactive-example).

In this article we shall take a look at reactive streams. First we’ll discover how reactive streams work, what problems they solve and how this fits into our asynchronous journey. We’ll finish this article by rewriting our weather example to use reactive streams. 

# Reactive streams
Reactive streams are an embodiment of the publish-subscribe pattern, which passes data in a  type safe manner to build pipelines in which data can be send over an asynchronous boundary. This definition is a bit abstract, so we'll look at these streams piece by piece. 

## Origin
The idea of reactive streams originates from somewhere in 2013 and was pioneered by Netflix, Pivotal and Lightbend[^1]. This group of companies created an open format which reactive streams libraries can implement in order to stay interoperable with each other. The original specification can be found [here](http://www.reactive-streams.org/) and consists of a lightweight jar with 4 interfaces. This interface specification is now part of Java 9 as the [Flow class](http://download.java.net/java/jdk9/docs/api/java/util/concurrent/Flow.html). 

Implementations of the reactive streams specification includes Netflix [RXJava](http://reactivex.io/) (as part of the ReactiveX project), Pivotals [Project Reactor](http://projectreactor.io/) and Lightbends [Akka Streams](http://doc.akka.io/docs/akka/2.5/scala/stream/index.html). These libraries each provide their own publishers, operators and subscribers. Besides the reactive streams libraries themselves, some new versions of technologies added libraries that support working with publishers and subscribers, most notably is Spring 5 which enables web programming using reactive streams.

## So what are these streams?
The reactive streams specification exists of four elements: 

1. **Publisher**: a publisher is an entity that will emit objects.
2. **Operator**: an operator is created from a publisher and it will return a new publisher. It can perform operation such as transforming objects, delaying emitted objects, filtering object, etc. Because operators always return a publisher, operators can be chained. Beware that this is not the [builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) as the order of operators matters. 
3. **Subscriber**: A subscriber handles messages been sent by a publisher. 
4. **Subscription**: An object holding the subscription itself.

Let’s try a quick example to get a feel for reactive streams, in our example we’ll use [Project Reactor](http://projectreactor.io/docs). We can start using Project Reactor by importing the [reactor-core](https://mvnrepository.com/artifact/io.projectreactor/reactor-core) Maven dependency.

``` java
// 1
Flux<String> publisher = Flux.just("one", "two", "three");
// 2
Flux<String> upper = publisher.map(String::toUpperCase);
// 3
upper.subscribe(System.out::println);
```

1. We create a `Publisher`. Our `Publisher` is of type `Flux<T>` which is one of the `Publisher` types provided by Project Reactor. A number of simple factory functions are provided, like `just()` which creates a `Flux<T>` which emits its parameters as values.
2. `map()` is an operator function which means it returns `Flux<T>` of it’s own. The `map()` function takes all objects coming from our `Publisher` and transforms it using a lambda, in our case applying the `toUpperCase()` function to each `String`. 
3. One of the advantages of reactive streams is that, most of them[^2], won’t perform any logic until subscribed to. If we leave out our `subscribe()` function, our program wouldn’t do anything at all. Here we subscribe and call `System.out.println()` for each `String` object received. 

Running this example results in the following output:
```
ONE
TWO
THREE
```

A key feature of reactive streams is that we don’t care about latency of messages. We program the actions that should be performed on data coming in and don’t care if our functions are performed synchronously or asynchronously. The `map()` function we wrote would be performed in a blocking way in our example, but if we rewrite our code to include a `delay()` (an operator that, as a side effect, defines that actions after that should be processed asynchronously) it’s subscribe block will finish asynchronously.

This concept of asynchronous abstraction can become complex to wrap our heads around so let’s practice by doing this by example:

``` java
String blockingResult = "";
String nonBlockingResult = "";

// blocking
Flux.just("one", "two", "three")
   .map(String::toUpperCase)
   .subscribe(val -> blockingResult += val + " + ");

System.out.println("blocking: " + blockingResult);

Flux.just("four", "five", "six")
   .delayElements(Duration.ofMillis(500))
   .map(String::toUpperCase)
   .subscribe(val -> {
       nonBlockingResult += val + " + ";
       System.out.println("updated non-blocking: " + val);
   });

System.out.println("non-blocking: " + nonBlockingResult);
```

This will print the following output:

```
blocking: ONE + TWO + THREE + 
non-blocking: 
updated non-blocking: FOUR
updated non-blocking: FIVE
updated non-blocking: SIX
```

So what happens in this example? Reactive streams operators don’t automatically switch to other threads. The truth is that communicating and managing threads causes some overhead, so some calls are actually faster in a synchronous context. Our `map()` operator alone doesn’t start the subscriber in a new thread and we thus see that the result of the blocking operator as the first log entry. 

In contrary our stream containing the `delayElements()` function will publish its data in a new thread, thus our main thread does not block when creating and subscribing to that `Flux<T>`. This means that our second log entry doesn’t include any results and print an empty result. 

> Q. But isn’t there a the possibility the first statement also ran asynchronously and it simply returned all values fast enough to be included in our log call?
> 
> A. You can check this by cloning the [code on Github](https://github.com/Kleinendorst/reactive-example) and adding: `System.out.println(Thread.currentThread());` in both subscribe callbacks and you’ll see that this isn’t the case. 

## Operators
The Reactive libraries come with operators that handle common operations that might take multiple lines of code normally. Some operators are the same as [Java 8’s streaming API]( http://www.oracle.com/technetwork/articles/java/ma14-java-se-8-streams-2177646.html), which is not surprising because lists of items can sometimes be handled the same way as items coming in asynchronously. Other operators are specific to the Streaming semantics, such as [`onBackpressureBuffer`](http://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#onBackpressureBuffer-int-). 

Operators always return a new `Publisher` object which makes it possible to chain operations. Transformations made by operators are best described in marble diagrams:

{% include image.html 
  url="/assets/reactive_streams/delayonnext.png" 
  description="source: <a href=\"https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html#delayElements-java.time.Duration-\">Project Reactor Docs</a>" %}

Marble diagrams show publishers as arrows. The line shows time from left to right. The perpendicular line shows when a publisher is complete. Each marble shows an emitted message. In the middle we see the function, including provided parameters. At the bottom of the diagram we see the resulting publisher outputted by the operator function. 

These marble diagrams are actually so helpful that they are provided for most operator in the [JavaDoc](http://projectreactor.io/docs/core/release/api/). 

## Using operators
Let’s look at an example where the use of operators shine: 

Imagine us wanting to parse [Markdown](https://nl.wikipedia.org/wiki/Markdown) input to HTML in real time. We want to render our input in a different thread so we don’t block the user from typing. Also parsing on each keypress would be wasteful, so instead we parse only when the user didn’t type anything for some time. 

In our example `fakeInput` will be a reactive stream emitting 5 random `char` objects, each 100 ms apart and then waiting 3 seconds until we send the next chars. Let’s see this in a marble diagram.

![example marble](/assets/reactive_streams/marble_diagram.svg "example marble")

Our goal is to not store the value somewhere but to combine them in the objects the stream returns, kind of like a `reduce` function. Then we only want to subscribe to our total value when no object came through for 200 ms, let’s look at our implementation:

``` java
fakeInput
       .scan("", (acc, val) -> acc += val)
       .sampleTimeout(c -> Mono.delay(Duration.ofMillis(200)))
       .subscribe(System.out::println);
```

Our scan operation will work like this:

{% include image.html 
  url="/assets/reactive_streams/scan.svg" 
  description="scan"
  width="600px" %}

`scan()` works just like `reduce()`, but instead of returning one value it returns the accumulated value each time a message comes in. 

Now let’s take a look at `sampleTimeout()`:

![sampleTimeout](/assets/reactive_streams/sampleTimeout.svg "sampleTimeout")

Each time a new message comes in, it starts a new publisher (`Mono<T>`) which completes in 200 ms. When all `Mono<T>` objects complete it send **only** its last message to it’s new publisher. The only thing we have to do now is subscribe to this new stream and we have the functionality we wanted. The actual publisher we subscribe to will look like this:

![end_result](/assets/reactive_streams/end_result.png "end_result")

These 2 messages are exactly what we wanted to subscribe to. This took just 4 lines of code to achieve thanks to the operators provided by Project Reactor. To appreciate the true power of operators you really have to get your hands dirty with them. In fact, the stream that provided our input data for this example was created using just a few operators:

``` java
Flux<Character> fakeInput = Flux.interval(Duration.ofMillis(5000))
       .flatMap(i -> Flux.range(1, 5).delayElements(Duration.ofMillis(100)))
       .map(i -> getRandomChar());
```

To understand this code, just try to decipher it using the [API page](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html) as an exercise. 

## Back pressure
Back pressure is something that was recently added as part of the reactive streams specification. Back pressure means that subscribers only relieve publishers of messages once the subscribers are ready to receive new messages. 

Without back pressure message 4 will be put on top of the subscriber buffer. If the subscribers handles messages quicker than the publisher publishes them, the buffer will remain small.

![push_model](/assets/reactive_streams/push_model.png "push_model")

Problems arise when the publisher pushes messages quicker than the subscriber can handle. Messages will pile up in the buffer till eventually the system runs out of memory and crashes. 

This method is called a push mechanism because the publisher pushes the messages to the subscriber. 

When using back pressure, the buffer is maintained at the publisher. A subscriber will request a message from the publisher whenever it’s done processing its messages.

![pull_model](/assets/reactive_streams/pull_model.png "pull_model")

> Q. So why is this better?
> 
> A. Because the publisher produces the messages, it should implement what to do when something goes wrong. Compare this with an employee (subscriber) of a supermarket that has to stock more shelves than he can manage. The manager (publisher) should fix this problem by hiring new employees or accepting emptier shelves. 

One use case in a microservice architecture is that we can use back pressure to optimize our messaging service. By implementing one of the new [reactive](https://github.com/ScalaConsultants/reactive-rabbit) [libraries](https://github.com/akka/reactive-kafka) we can signal back to the message service if new messages could be processed. Imagine two microservices that can handle the same tasks. Service A starts with a message that takes 1 hour to process whilst Service B starts with a message that takes just 10 minutes. When using round robin dispatching at the message service, the third message would be handed to service A sitting idle in its mailbox for an hour, whilst it could be processed after 10 minutes at service B.

By using back pressure our message service connector could request the message when it’s done processing its last message, thus the third message would be send to Service B. 

Back pressure is called using a pull based mechanism. Pull based mechanisms have the disadvantage of not allowing publishers to work at full speed when the subscribers can handle the total workload. The reactive specification even has an answer to this drawback. Subscribers can request more messages, thus buffering a **bounded** amount of messages themselves. If a subscriber requests more messages than the publisher has available, the publisher switches to a push mechanism for at least the amount of messages that are requested by the publisher. 

## Weather example
Let’s rewrite the asynchronous part of our weather application using these streams. 
We start by writing our `WeatherService` that provides a fake, blocking, method. 

``` java
class WeatherService {
   private long latency = 4000;
   private double maxTemp = 30;


   Mono<WeatherReport> getTemperature(String type, String location) { // <- 2
       return Mono.defer(() -> // <- 3
               Mono.just(new WeatherReport(type, fetchWeather(type, location))))
               .subscribeOn(Schedulers.parallel()); // <- 4
   }

   private Double fetchWeather(String type, String location) { // <- 1
       double temperature = Math.random() * maxTemp;

       try {
           Thread.sleep(latency);
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       return temperature;
   }
}
```

We also store a result in a `WeatherReport` object:

``` java
class WeatherReport {
   public String type;
   public Double temperature;

   WeatherReport(String type, Double temperature) {
       this.type = type;
       this.temperature = temperature;
   }
}
```

Now in our main app we’ll write the code to handle getting both blocking calls asynchronously. 

``` java
LocalTime start = LocalTime.now();
WeatherService weatherService = new WeatherService();
String location = "Eindhoven";

Flux.just("actual", "predicted")
       .flatMap(type -> weatherService.getTemperature(type, location)) // <- 5
       .scan(new HashMap<String, Double>(), (acc, val) -> { // <- 6
           acc.put(val.type, val.temperature);
           return acc;
       })
       .filter(result -> result.size() > 1) // <- 7
       .subscribe(result -> {
           LocalTime end = LocalTime.now();

           System.out.println(result + " finished in "
                   + Duration.between(start, end).toMillis()
                   + "ms");
       });

```

When run, this provides the following output:

```{actual=13.678773126624673, predicted=16.631860503562045} finished in 4216ms```

Each request we made took 4 seconds, so this code actually ran asynchronously. Now let’s examine what we just wrote:

1. Our `fetchWeather()` function doesn’t have any reactive parts, it’s a straight old  blocking Java function call. 
2. We want to wrap our `fetchWeather()` function with something reactive, this means returning a `Mono<T>` (non-blocking) which promises to return a `WeatherReport` somewhere in the future.
3. `defer()` is a trick. It takes a lambda which runs the blocking code. If we wouldn’t use this construct, Java would execute our blocking code first without touching any reactive streams related functions, thus making it impossible for the `getTemperature()` to return non blocking.
4. `subscribeOn()` will make sure that this stream is started on a new thread.
5. The `flatMap()` operator works primarily as a `map()` function, it takes our “actual” and “predicted” values and maps them to reports. The difference with a normal map is that `flatMap()` may take reactive objects as a return type which emitted messages it "flattens" onto a single stream. 
6. We saw the `scan()` function earlier in this article. It start initially with an empty `HashMap<T>`. Each message is then `put()` in the `HashMap` after which the result is sent downstream. 
7. We only want to emit the message when the **full** result is ready. `filter()` only forwards a message downstream once it meets its requirements.

Finally, let’s visualize our data being transformed through our stream:

![complete marble](/assets/reactive_streams/marble_diagram_complete_weather.png "complete marble")

## Conclusion
Using reactive streams allows us to write code that handles concurrency problems with elegance. We can be sure that streams that we write can consume and publish data to other libraries by implementing Java 9s new `Flow` class. Finally, by ensuring back pressure is communicated throughout the stream, we ensure buffer problems are handled at the right level. 

----------------
[^1]: Lightbend is the developer of the Akka streams we saw in an earlier article
[^2]: There are publishers that start emitting when created, but we’ll not focus on these in this article.