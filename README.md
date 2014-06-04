# Reactive Streams #

The purpose of Reactive Streams is to provide a standard for asynchronous stream processing with non-blocking backpressure.

The latest preview release is available on Maven Central as

    <dependency>
      <groupId>org.reactivestreams</groupId>
      <artifactId>reactive-streams</artifactId>
      <version>0.4.0.M1</version>
    </dependency>
    <dependency>
      <groupId>org.reactivestreams</groupId>
      <artifactId>reactive-streams-tck</artifactId>
      <version>0.4.0.M1</version>
    </dependency>

## Goals, Design and Scope ##

Handling streams of data—especially “live” data whose volume is not predetermined—requires special care in an asynchronous system. The most prominent issue is that resource consumption needs to be carefully controlled such that a fast data source does not overwhelm the stream destination. Asynchrony is needed in order to enable the parallel use of computing resources, on collaborating network hosts or multiple CPU cores within a single machine.

The main goal of Reactive Streams is to govern the exchange of stream data across an asynchronous boundary – think passing  elements on to another thread or thread-pool — while ensuring that the receiving side is not forced to buffer arbitrary amounts of data. In other words, backpressure is an integral part of this model in order to allow the queues which mediate between threads to be bounded. The benefits of asynchronous processing would be negated if the communication of backpressure were synchronous (see also the [Reactive Manifesto](http://reactivemanifesto.org/)), therefore care has been taken to mandate fully non-blocking and asynchronous behavior of all aspects of a Reactive Streams implementation.

It is the intention of this specification to allow the creation of many conforming implementations, which by virtue of abiding by the rules will be able to interoperate smoothly, preserving the aforementioned benefits and characteristics across the whole processing graph of a stream application.

It should be noted that the precise nature of stream manipulations (transformation, splitting, merging, etc.) is not covered by this specification. Reactive Streams are only concerned with mediating the stream of data between different processing elements. In their development care has been taken to ensure that all basic ways of combining streams can be expressed.

In summary, Reactive Streams is a standard and specification for Stream-oriented libraries for the JVM that

 - process a potentially unbounded number of elements 
 - in sequence,
 - asynchronously passing elements between components,
 - with mandatory non-blocking backpressure.

The Reactive Streams specification consists of the following parts:

**The API** specifies the types to implement Reactive Streams and achieve interoperablility between different implementations.

***The Technology Compatibility Kit (TCK)*** is a standard test suite for conformance testing of implementations.

Implementations are free to implement additional features not covered by the specification as long as they conform to the API requirements and pass the tests in the TCK.

### API Components ###

The API consists of the following components that are required to be provided by Reactive Stream implementations:

1. Publisher
2. Subscriber 
3. Subscription
4. Processor

A *Publisher* is a provider of a potentially unbounded number of sequenced elements, publishing them according to the demand received from its Subscriber(s). 

In response to a call to `Publisher.subscribe(Subscriber)` the possible invocation sequences for methods on the `Subscriber` are given by the following protocol:

```
onError | (onSubscribe onNext* (onError | onComplete)?)
```

#### NOTES

- The specifications below use binding words in CAPLOCKS from https://www.ietf.org/rfc/rfc2119.txt
- The terms emit, signal or send are interchangeable. The specifications below will use `signal`.
- The terms `synchronously` or `synchronous` refer to executing in the calling `Thread`.

#### Under Discussion
- 1.10
- 1.14
- 1.18
- 2.3
- 3.3


### SPECIFICATION

#### 1. Publisher ([Code](https://github.com/reactive-streams/reactive-streams/blob/master/api/src/main/java/org/reactivestreams/Publisher.java))

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<T> s);
}
````

1. The number of `onNext` signaled by a `Publisher` to a `Subscriber` MUST NOT exceed the cumulative demand that has been signaled via that `Subscriber`’s `Subscription`.
2. A `Publisher` MAY signal less `onNext` than requested and terminate the `Subscription` by calling `onComplete` or `onError`.
3. Events signaled to a `Subscriber` MUST be signaled sequentially (no concurrent notifications).
4. If a `Publisher` fails it MUST signal an `onError`.
5. If a `Publisher` terminates successfully (finite stream) it MUST signal an `onComplete`.
6. If a Publisher signals either `onError` or `onComplete` on a `Subscriber`, that `Subscriber`’s `Subscription` MUST be considered canceled.
7. Once a terminal state has been signaled (`onError`, `onComplete`) it is REQUIRED that no further signals occur.
8. Upon receiving a `Subscription.cancel` request it SHOULD, as soon as it can, stop signaling its `Subscriber`.
9. `Subscription`'s which have been canceled SHOULD NOT receive subsequent `onError` or `onComplete` signals, but implementations will not be able to strictly guarantee this in all cases due to the intrinsic race condition between actions taken concurrently by `Publisher` and `Subscriber`.
10. A `Publisher` SHOULD NOT throw an `Exception`. The only legal way to signal failure (or reject a `Subscription`) is via the `Subscriber.onError` method. _[Under Discussion]_
11. The `Subscriber.onSubscribe` method on a given `Subscriber` instance MUST NOT be called more than once (based on object equality).
12. The `Publisher.subscribe` method MAY be called as many times as wanted but MUST be with a different Subscriber each time [see 1.11]. It MUST reject the Subscription with a `java.lang.IllegalStateException` if the same Subscriber already has an active `Subscription` with this `Publisher`. The cause message MUST include a reference to this rule and/or quote the full rule.
13. A `Publisher` MAY support multi-subscribe and choose whether each `Subscription` is unicast or multicast.
14. A `Publisher` MAY reject calls to its `subscribe` method if it is unable or unwilling to serve them (e.g. because it is overwhelmed or bounded by a finite number of underlying resources, etc...). If rejecting it MUST do this by calling `onError` on the `Subscriber` passed to `Publisher.subscribe` instead of calling `onSubscribe`". _[Under Discussion]_
15. A `Publisher` in `completed` state MUST NOT call `Subscriber.onSubscribe` and MUST signal an `Subscriber.onComplete` on the given `Subscriber`
16. A `Publisher` in `error` state MUST NOT call `Subscriber.onSubscribe` and MUST signal an `Subscriber.onError` with the error cause on the given `Subscriber`
17. A `Publisher` in `shut-down` state MUST NOT call `Subscriber.onSubscribe` and MUST signal an `Subscriber.onError` with `java.lang.IllegalStateException` on the given `Subscriber`. The cause message MUST include a reference to this rule and/or quote the full rule.
18. A `Publisher` MUST support a pending element count up to 2^63-1 (java.lang.Long.MAX_VALUE) and provide for overflow protection. _[Under Discussion]_
19. A `Publisher` MUST produce the same elements in the same sequence for all its subscribers. Producing the stream elements at (temporarily) differing rates to different subscribers is allowed.  
20. A `Publisher` MUST start producing with the oldest element still available for a new `Subscription`.

#### 2. Subscriber ([Code](https://github.com/reactive-streams/reactive-streams/blob/master/api/src/main/java/org/reactivestreams/Subscriber.java))

```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
````

1. A `Subscriber` MUST signal demand via `Subscription.request(int n)` to receive onNext signals.
2. If a `Subscriber` suspects that its processing of events will negatively impact its `Publisher`'s responsivity, it is RECOMMENDED that it asynchronously dispatches its signals.
3. A `Subscriber.onComplete()` and `Subscriber.onError(Throwable t)` MUST NOT call any methods on the `Subscription`, the `Publisher` or any other `Publishers` or `Subscribers`. [Under Discussion]
4. A `Subscriber.onComplete()` and `Subscriber.onError(Throwable t)` MUST consider the Subscription cancelled after having received the signal.
5. A `Subscriber` MUST NOT accept an `onSubscribe` signal if it already has an active `Subscription`. What exactly "not accepting" means is left to the implementation but should include behavior that makes the user aware of the usage error (e.g. by logging, throwing an exception or similar).
6. A `Subscriber` MUST call `Subscription.cancel()` if it is no longer valid to the `Publisher` without the `Publisher` having signaled `onError` or `onComplete`.
7. A `Subscriber` MUST ensure that all calls on its `Subscription` take place from the same thread or provide for respective external synchronization.
8. A `Subscriber` MUST be prepared to receive one or more `onNext` signals after having called `Subscription.cancel()` if there are still requested elements pending [see 3.12]. `Subscription.cancel()` does not guarantee to perform the underlying cleaning operations immediately.
9. A `Subscriber` MUST be prepared to receive an `onComplete` signal with or without a preceding `Subscription.request(int n)` call.
10. A `Subscriber` MUST be prepared to receive an `onError` signal with or without a preceding `Subscription.request(int n)` call.
11. A `Subscriber` MUST make sure that all calls on its `onXXX` methods happen-before the processing of the respective signals. I.e. the Subscriber must take care of properly publishing the signal to its processing logic.

#### 3. Subscription ([Code](https://github.com/reactive-streams/reactive-streams/blob/master/api/src/main/java/org/reactivestreams/Subscription.java))

```java
public interface Subscription {
    public void request(int n);
    public void cancel();
}
````

1. A `Subscription.request` and `Subscription.cancel` MUST not be called outside its `Subscriber` context. A `Subscription` represents the unique relationship between a `Subscriber` and a `Publisher` [see 1.11].
2. Calls from a `Subscriber` to `Subscription.request(int n)` can be made directly since it is the responsibility of `Subscription` to handle async dispatching.
3. The `Subscription.request` method MUST assume that it will be invoked synchronously and MUST NOT allow unbounded recursion such as `Subscriber.onNext` -> `Subscription.request` -> `Subscriber.onNext`.  _[Under Discussion]_
4. The `Subscription.request` method SHOULD NOT synchronously perform heavy computations.
5. The `Subscription.cancel` method MUST assume that it will be invoked synchronously and SHOULD NOT synchronously perform heavy computations.
6. After the `Subscription` is cancelled, additional `Subscription.request(int n)` MUST be NOPs.
7. After the `Subscription` is cancelled, additional `Subscription.cancel()` MUST be NOPs.
8. While the `Subscription` is not cancelled, `Subscription.request(int n)` MUST register the given number of additional elements to be produced to the respective subscriber.
9. While the `Subscription` is not cancelled, `Subscription.request(int n)` MUST throw a `java.lang.IllegalArgumentException` if the argument is <= 0. The cause message MUST include a reference to this rule and/or quote the full rule.
10. While the `Subscription` is not cancelled, `Subscription.request(int n)` MAY synchronously call `onNext` on this (or other) subscriber(s) if and only if the next element is already available.
11. While the `Subscription` is not cancelled, `Subscription.request(int n)` MAY synchronously call `onComplete` or `onError` on this (or other) subscriber(s).
12. While the `Subscription` is not cancelled, `Subscription.cancel()` the `Publisher` MUST eventually cease to call any methods on the corresponding subscriber.
13. While the `Subscription` is not cancelled, `Subscription.cancel()` the `Publisher` MUST eventually drop any references to the corresponding subscriber. Re-subscribing with the same `Subscriber` object is discouraged [see 1.11], but this specification does not mandate that it is disallowed since that would mean having to store previously canceled subscriptions indefinitely.
14. While the `Subscription` is not cancelled, `Subscription.cancel()` the `Publisher` MUST transition to a `shut-down` state [see 1.17] if the given `Subscription` is the last downstream `Subscription`. Explicitly adding "keep-alive" Subscribers SHOULD prevent automatic shutdown if required.

A `Subscription` is shared by exactly one `Publisher` and one `Subscriber` for the purpose of mediating the data exchange between this pair. This is the reason why the `subscribe()` method does not return the created `Subscription`, but instead returns `void`; the `Subscription` is only passed to the `Subscriber` via the `onSubscribe` callback.

#### 4.Processor ([Code](https://github.com/reactive-streams/reactive-streams/blob/master/api/src/main/java/org/reactivestreams/Processor.java))

```java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
````

1. A `Processor` represents a processing stage—which is both a `Subscriber` and a `Publisher` and MUST obey the contracts of both.
2. A `Processor` MUST cancel its upstream Subscription if its last downstream Subscription has been cancelled.
3. A `Processor` MUST immediately pass on `onError` signals received from its upstream to its downstream.
4. A `Processor` MUST be prepared to receive incoming elements from its upstream even if a downstream subscriber has not requested anything yet.

### Asynchronous vs Synchronous Processing ###

The Reactive Streams API prescribes that all processing of elements (`onNext`) or termination signals (`onError`, `onComplete`) MUST NOT *block* the `Publisher`. However, each of the `on*` handlers can process the events synchronously or asynchronously. 

Take this example:

```
nioSelectorThreadOrigin map(f) filter(p) consumeTo(toNioSelectorOutput)
```

It has an async origin and an async destination. Let's assume that both origin and destination are selector event loops. The `Subscription.request(n)` must be chained from the destination to the origin. This is now where each implementation can choose how do do this.

The following uses the pipe `|` character to signal async boundaries (queue and schedule) and `R#` to represent resources (possibly threads).

```
nioSelectorThreadOrigin | map(f) | filter(p) | consumeTo(toNioSelectorOutput)
-------------- R1 ----  | - R2 - | -- R3 --- | ---------- R4 ----------------
```

In this example each of the 3 consumers, `map`, `filter` and `consumeTo` asynchronously schedule the work. It could be on the same event loop (trampoline), separate threads, whatever.

```
nioSelectorThreadOrigin map(f) filter(p) | consumeTo(toNioSelectorOutput)
------------------- R1 ----------------- | ---------- R2 ----------------
```

Here it is only the final step that asynchronously schedules, by adding work to the NioSelectorOutput event loop. The `map` and `filter` steps are synchronously performed on the origin thread.

Or another implementation could fuse the operations to the final consumer:

```
nioSelectorThreadOrigin | map(f) filter(p) consumeTo(toNioSelectorOutput)
--------- R1 ---------- | ------------------ R2 -------------------------
```

All of these variants are "asynchronous streams". They all have their place and each has different tradeoffs including performance and implementation complexity.

The Reactive Streams contract allows implementations the flexibility to manage resources and scheduling and mix asynchronous and synchronous processing within the bounds of a non-blocking, asynchronous, push-based stream.

In order to allow fully asynchronous implementations of all participating SPI elements—`Publisher`/`Subscription`/`Subscriber`—all methods defined by these interfaces return `void`.


### Subscriber controlled queue bounds ###

One of the underlying design principles is that all buffer sizes are to be bounded and these bounds must be *known* and *controlled* by the subscribers. These bounds are expressed in terms of *element count* (which in turn translates to the invocation count of onNext). Any implementation that aims to support infinite streams (especially high output rate streams) needs to enforce bounds all along the way to avoid out-of-memory errors and constrain resource usage in general.

Since back-pressure is mandatory the use of unbounded buffers can be avoided. In general, the only time when a queue might grow without bounds is when the publisher side maintains a higher rate than the subscriber for an extended period of time, but this scenario is handled by backpressure instead.

Queue bounds can be controlled by a subscriber by signaling demand for the appropriate number of elements. At any point in time the subscriber knows:

 - the total number of elements requested: `P`
 - the number of elements that have been processed: `N`

Then the maximum number of elements that may arrive—until more demand is signaled to the Publisher—is `P - N`. In the case that the subscriber also knows the number of elements B in its input buffer then this bound can be refined to `P - B - N`.

These bounds must be respected by a publisher independent of whether the source it represents can be backpressured or not. In the case of sources whose production rate cannot be influenced—for example clock ticks or mouse movement—the publisher must choose to either buffer or drop elements to obey the imposed bounds.

Subscribers signaling a demand for one element after the reception of an element effectively implement a Stop-and-Wait protocol where the demand signal is equivalent to acknowledgement. By providing demand for multiple elements the cost of acknowledgement is amortized. It is worth noting that the subscriber is allowed to signal demand at any point in time, allowing it to avoid unnecessary delays between the publisher and the subscriber (i.e. keeping its input buffer filled without having to wait for full round-trips).

## Legal

This project is a collaboration between engineers from Netflix, Pivotal, RedHat, Twitter, Typesafe and many others. The code is offered to the Public Domain in order to allow free use by interested parties who want to create compatible implementations. For details see `COPYING`.
