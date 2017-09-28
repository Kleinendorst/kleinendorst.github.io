---
layout: post
title:  "Asynchronous Java"
date:   2017-09-22 14:30:00
categories: main
comments: true
---
In order to properly scale applications, asynchronous task handling is key. The principal of asynchronous systems can be achieved on both application and architectural level. In this article we will discuss blocking IO and its disadvantages, as well as how to implement nonblocking IO. The examples in this article will be written in Java, but apply to most modern programming languages.

## Blocking IO
In source code the compiler searches for an order in which to resolve expressions. The compiler keeps track of its current tasks in a FILO (first in last out) array called the "call stack". Let’s look at how this works: 

{% highlight java %}
int a = 2 * 60 + 3;
// --> 2 * 60 = 120
// --> 120 + 3 = 123
// --> writes 123 to space a in memory
{% endhighlight %}

Ordering in this example is resolved in a mathematically correct fashion, but languages also have some rules of their own[^1]. Whenever an expression is finished, the next runnable line of code is executed. Notice that this next line of code can only be executed once the first line of code has finished executing, even though the second line isn’t dependent on the first line. 
In simple examples, like the first example, this doesn’t pose much of a problem, but consider the following example:

{% highlight java %}
String location = "Utrecht";
WeatherService ws = new WeatherService();

double actual = ws.fetchActualTemperature(location);
double predicted = ws.fetchPredictedTemperature(location);

if (actual < predicted) {
   System.out.println("It's colder than anticipated!");
}
{% endhighlight %}

The weather service will make a REST API request to retrieve information. Retrieving information via REST is a (relatively) time consuming task. In this example the predicted temperature may only be retrieved once the current temperature is retrieved. These tasks could easily be retrieved in parallel.

When the call stack is waiting for a return value, in this case from the weather service, it can’t execute additional code. In our case this means that our task will take more time. In the case of a visual application, the GUI will freeze and not respond to user input. To solve these kind of issues, we use multi-threading.

## Multi-threading NIO
NIO (non blocking IO) is achieved by reserving software threads[^2]. Reserving threads is not supported by all modern programming languages[^3], but it is supported in Java. Let’s look at an example:

{% highlight java %}
class Weather {
   private Double actual;
   private Double predicted;

   void setActual(double actual) {
       this.actual = actual;
       if (predicted != null) {
           sendStatus();
       }
   }

   void setPredicted(double predicted) {
       this.predicted = predicted;
       if (actual != null) {
           sendStatus();
       }
   }

   private void sendStatus() {
       String message;
       if (actual < predicted) {
           message = "It's colder than predicted.";
       } else if (actual > predicted) {
           message = "It's warmer than predicted.";
       } else {
           message = "Predictions were spot on.";
       }
       Main.reportWithDuration(message);
   }
}
{% endhighlight %}

This weather object has setters for both the actual and the predicted value. Whenever both values are not null, it will access its values. Now look at the fake implementation of the weather service:

{% highlight java %}
class WeatherService {
   double fetchPredictedTemperature(String location) {
       return fakeFetch(5000, 30);
   }

   double fetchActualTemperature(String location) {
       return fakeFetch(7000, 30);
   }

   private double fakeFetch(long latency,
                            double maxTemperature) {
       try {
           Thread.sleep(latency);
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       return Math.random() * maxTemperature;
   }
}
{% endhighlight %}

In order to simulate a long response time we call `Thread.sleep()`, which just stalls all code execution for the provided number of milliseconds. Now let’s run this code in a blocking way first:

{% highlight java %}
public class Main {
   private static LocalTime start;

   public static void reportWithDuration(String message) {
       System.out.println(message + ":: running for: "
               + Duration.between(start, LocalTime.now()).toMillis()
               + "ms");
   }

   public static void main(String[] args) {
       start = LocalTime.now();
       String location = "Utrecht";
       WeatherService ws = new WeatherService();

       reportWithDuration(
               "Start getting the actual temperature");
       double actual = ws.fetchActualTemperature(location);

       reportWithDuration(
               "Start getting the predicted temperature");
       double predicted = ws.fetchPredictedTemperature(location);

       if (actual < predicted) {
           System.out.println("It's colder than anticipated!");
       }
   }
}
{% endhighlight %}

The code doesn’t use threading in any way and produces the following output:

`Start getting the actual temperature:: running for: 11ms`

`Start getting the predicted temperature:: running for: 7016ms`

`It's colder than predicted.:: running for: 12016ms`

Notice that it starts fetching the predicted temperature only when the current temperature was already fetched. Now let’s rewrite the `Main` class so that the fetching can happen in parallel:

{% highlight java %}
public class Main {
   private static LocalTime start;

   static void reportWithDuration(String message) {
       System.out.println(message + ":: running for: "
               + Duration.between(start, LocalTime.now()).toMillis()
               + "ms");
   }

   public static void main(String[] args) throws IOException {
       start = LocalTime.now();
       String location = "Utrecht";
       WeatherService ws = new WeatherService();
       Weather weather = new Weather();

       reportWithDuration(
               "Start getting the actual temperature");
       Runnable actualRunnable = () -> {
           weather.setActual(ws.fetchActualTemperature(location));
       };

       reportWithDuration(
               "Start getting the predicted temperature");
       Runnable predictedRunnable = () -> {
       weather.setPredicted(ws.fetchPredictedTemperature(location));
       };

       new Thread(actualRunnable).start();
       new Thread(predictedRunnable).start();

       System.out.println(
               "Press any key to stop.");
       System.in.read();
   }
}
{% endhighlight %}

We create `Runnable` instances which will be the input for a new `Thread` object we create. These runnables will perform their task inside a thread allocated by the JVM[^4], and therefore, will be ran in parallel. Now let’s compare the console output and see if this code was faster to execute:


`Start getting the actual temperature:: running for: 12ms`

`Start getting the predicted temperature:: running for: 16ms`

`Press any key to stop.`

`It's colder than predicted.:: running for: 7017ms`

There are also some downsides to using multithreading. We have to check after each runnable whether both values have been returned, because we don't know the finishing order of runnables.

Also mutable variables may become faulty if accessed by threads at the same time. In order to prevent corrupt data, one can use Java’s atomic variable types[^5] or implement locking[^6]. Both increase complexity in large systems. Also, bugs that result from accessing variables on different threads are hard to find.

In the [next article]({{ page.next.url }}) we’ll have a look at the basics of the actor system, a model that was created specially for asynchronous computing. We’ll discover how the system works, how it solves concurrency problems and how to implement it in Java.

----------------
[^1]: Some more associativity rules for Java can be found [here](http://introcs.cs.princeton.edu/java/11precedence/).
[^2]: There is a difference between software and hardware threads: see [this](https://stackoverflow.com/questions/5593328/software-threads-vs-hardware-threads).
[^3]: Like Javascript that uses promises to handle asynchronous tasks. [MDN page of promises](https://developer.mozilla.org/nl/docs/Web/JavaScript/Reference/Global_Objects/Promise)
[^4]: [Java virtual machine](https://nl.wikipedia.org/wiki/Java_Virtual_Machine)
[^5]: [Atomic variables](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomicvars.html)
[^6]: [Locking](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/Lock.html)