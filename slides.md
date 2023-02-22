---
title: ZIO Promises
description: ZIO Promises
author: Stephen Marsh
keywords: scala,zio
marp: true
---

<style>section { font-size: 20px; }</style>

# ZIO Promise

Last time we learned about Ref, used to exchange data between ZIO fibers.

Today we will learn about ZIO's Promise, a powerful piece of functionality used to provide synchronization between ZIO fibers.

ZIO Promises are similiar to the Promises and Futures provided by the Scala standard library, but more powerful.

---

## Basic Example

Unlike Ref, Promises don't start with a value inside. Consumers of Promises can call `await` to block until a value becomes available. Producers can call `succeed` to provide a value.


```scala
for {
  promise <- Promise.make[Throwable, String]
  // Fork a fiber to print the result of the promise once it has one.
  _ <- promise.await.flatMap(s => Console.printLine(s"Received $s from promise")).fork
  // Main fiber will complete promise
  _ <- Console.printLine("Taking my time...")
  _ <- ZIO.sleep(1.seconds)
  _ <- promise.succeed("FOOBAR")
} yield ()
```

## Output

```
Taking my time...
Received FOOBAR from promise
```

---

## Unhappy Path

Promises can also contain errors as well. You can also take advantage of ZIO's unchecked error channel to fill them with death.

```scala
for {
  errorPromise <- Promise.make[Throwable, String]
  deathPromise <- Promise.make[Nothing, String]
  _ <- errorPromise.await.flip.flatMap(e => Console.printLine(s"Promise errored with $e")).fork
  _ <- deathPromise.await.cause.flatMap(cause => Console.printLine(s"Promise died with $cause")).fork
  _ <- errorPromise.fail(new Exception("BOOM!"))
  _ <- deathPromise.die(new Exception("KABOOM!"))
} yield ()
```

## Output

```
Promise errored with java.lang.Exception: BOOM!
Promise died with Die(java.lang.Exception: KABOOM!,Stack trace for thread "zio-fiber-79":
	at <empty>.PromiseSpec.spec(PromiseSpec.scala:25)
	at <empty>.PromiseSpec.spec(PromiseSpec.scala:23))
```

---

## Complete

The `complete` method offers a general way of "completing" a promise. `complete` will run a ZIO and put the result (success, error, or death) into the promise.

```scala
for {
  promise <- Promise.make[Throwable, Int]
  _ <- promise.complete(ZIO.succeed(5).delay(1.seconds))
  _ <- Console.printLine("About to await promise")
  result <- promise.await
  _ <- Console.printLine(s"Finished awaiting promise, got $result")
} yield ()
```

## Output

```
About to await promise
Finished awaiting promise, got 5
```

---

## CompleteWith

The `completeWith` method will store a ZIO inside of a Promise without running it. Every time the Promise is awaited, the ZIO will be run and the result provided. In other words, `completeWith` is lazy.

```scala
for {
  promise <- Promise.make[Throwable, Unit]
  _ <- promise.completeWith(Console.printLine("Value inside promise").unit)
  _ <- Console.printLine("About to await promise")
  _ <- promise.await
  _ <- promise.await
  _ <- Console.printLine("Finished awaiting promise")
} yield assertCompletes
```

## Output

```
About to await promise
Value inside promise
Value inside promise
Finished awaiting promise
```

---

## Monitoring

Promises can be polled and monitored, using the `isDone` and `poll` methods.

```scala
def awaitPromise(promise: Promise[Throwable, Int]): Task[Unit] = for {
  isDone <- promise.isDone
  _ <- if (isDone) {
    Console.printLine(s"Promise complete!")
  } else {
    for {
      _ <- Console.printLine(s"Promise not yet complete...")
      _ <- ZIO.sleep(1.seconds)
      _ <- awaitPromise(promise)
    } yield ()
  }
} yield ()
for {
  promise <- Promise.make[Throwable, Int]
  _ <- promise.complete(ZIO.succeed(5).delay(5.seconds)).fork
  _ <- awaitPromise(promise)
} yield ()
```

## Output
```
Promise not yet complete...
Promise not yet complete...
Promise not yet complete...
Promise not yet complete...
Promise not yet complete...
Promise complete!
```

---

## Combined with Ref

Ref and Promise combine in useful ways.

```scala
class Cache[K, V](ref: Ref[Map[K, Promise[Throwable, V]]], lookup: K => Task[V]) {
  def get(key: K): Task[V] = for {
    newPromise <- Promise.make[Throwable, V]
    memoizedLookup <- lookup(key).memoize
    _ <- newPromise.completeWith(memoizedLookup)
    promise <- ref.modify { map =>
      map.get(key) match {
        case Some(oldPromise) => (oldPromise, map)
        case None => (newPromise, map.updated(key, newPromise))
      }
    }
    result <- promise.await
  } yield result
}
```

---

## Combined with Ref

```scala
def lookup(s: String): Task[String] = for {
  _ <- Console.printLine(s"Computing reverse of $s")
} yield s.reverse

for {
  ref <- Ref.make[Map[String, Promise[Throwable, String]]](Map.empty)
  cache = new Cache[String, String](ref, lookup)
  r <- cache.get("HELLO")
  _ <- Console.printLine(r)
  r <- cache.get("HELLO")
  _ <- Console.printLine(r)
  r <- cache.get("WORLD")
  _ <- Console.printLine(r)
} yield ()
```

## Output

```
Computing reverse of HELLO
OLLEH
OLLEH
Computing reverse of WORLD
DLROW
```

---

# ZIO Queue

Like Promises, but can hold multiple values.

Never holds errors or death.

---

## Basic Example

```scala
for {
  queue <- Queue.unbounded[String]
  // Fork a fiber to continually read from the Queue
  _ <- queue.take.flatMap(s => Console.printLine(s"Received $s from queue")).forever.fork
  // Main fiber will put values in queue
  _ <- queue.offer("FOO")
  _ <- queue.offer("BAR")
} yield ()
```

```
Received FOO from queue
Received BAR from queue
```

---

## Multiple Readers and Writers

Any number of fibers can be reading and writing from a queue concurrently.

```scala
for {
  queue <- Queue.unbounded[String]
  _ <- queue.take.flatMap(s => Console.printLine(s"Worker 1: $s")).forever.fork
  _ <- queue.take.flatMap(s => Console.printLine(s"Worker 2: $s")).forever.fork
  // Main fiber will put values in queue
  _ <- queue.offerAll(Seq("FOO", "BAR", "ONE", "TWO"))
} yield ()
```

```
Worker 2: BAR
Worker 1: FOO
Worker 2: ONE
Worker 1: TWO
```

---

## Bounded blocking queues

Queues can be created with limited capacity. With a bounded queue, the `offer` call will block until space is available in the queue.

```scala
for {
  queue <- Queue.bounded[Int](2)
  _ <- ZIO.foreach(1 to 4) { s => for {
    _ <- queue.offer(s)
    _ <- Console.printLine(s"Wrote $s to queue")
  } yield ()}.fork
  _ <- ZIO.replicateZIO(4) { for {
    _ <- ZIO.sleep(1.seconds)
    _ <- Console.printLine("About to read from queue...")
    _ <- queue.take
  } yield ()}
} yield ()
```

```
Wrote 1 to queue
Wrote 2 to queue
About to read from queue...
Wrote 3 to queue
About to read from queue...
Wrote 4 to queue
About to read from queue...
About to read from queue...
```

---

## Bounded sliding queues

ZIO's sliding queues also have limited capacity, but will not block the `offer` call. Instead, older values will be removed from the queue to make room for new ones.

```scala
for {
  queue <- Queue.sliding[Int](2)
  _ <- queue.offerAll(1 to 4)
  result <- queue.takeAll
  _ <- Console.printLine(s"Read $result from queue")
} yield ()
```

```
Read Chunk(3,4) from queue
```

---

## Bounded dropping queues

ZIO's dropping queues are similiar, but instead the new values will be dropped without ever entering the queue.

```scala
for {
  queue <- Queue.dropping[Int](2)
  _ <- queue.offerAll(1 to 4)
  result <- queue.takeAll
  _ <- Console.printLine(s"Read $result from queue")
} yield ()
```

```
Read Chunk(1,2) from queue
```

---

## Shutdown

Queues can be shutdown, which will interrupt all fibers waiting to take or put values into the queue.

```scala
for {
  queue <- Queue.unbounded[Int]
  fiber <- queue.take.flatMap(s => Console.printLine(s"Received $s from queue")).forever.fork
  _ <- queue.offer(1)
  _ <- ZIO.sleep(1.seconds)
  _ <- queue.offer(2)
  _ <- ZIO.sleep(1.seconds)
  _ <- queue.shutdown
  _ <- fiber.await
} yield assertCompletes
```

```
Received 1 from queue
Received 2 from queue
```
