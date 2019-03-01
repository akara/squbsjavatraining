## Day 2: Functional Composition

### Container Types
* We deal with container types every day: `List<T>`, `Set<T>`
* Even an array can be considered `Array<T>` (in Scala it indeed is)
* Java 8 adds a few: `Optional<T>`, `CompletionStage<T>`, `CompletableFuture<T>`
* A `List<T>` or `Set<T>` contains 0 or more elements
* A `Optional<T>` contains 0 (empty) or 1 element
* A `Future<T>` (`CompletionStage<T>`, `CompletableFuture<T>`) contains an element or an error that may be filled in the future.

### Transformation the Old Way

```java
    public List<String> intListToString(List<Integer> intList) {
        ArrayList<String> stringList = new ArrayList<>(intList.size());
        for (Integer i : intList) {
            stringList.add("count: " + i);
        }
        return stringList;
    }
```

### No! We can do better!
* Given a `List<Integer>` and a function that transforms `Integer` -> `String`, we should be able to apply this transformation to the whole `List` in one shot.

Lets take a brief look at some Scala code (easier to understand):

```scala
def intListToString(myList: List[Int]) = myList.map(i => "count: " + i)
```

So this converts a `List(1, 2, 3)` to a `List("count: 1", "count: 2", "count: 3")`.

Yes, we can do something like this in Java, too! But only with `Streams`. You have to convert `List` to `Stream` and back if you want to work with list.

```java
    public Stream<String> intStreamToString(Stream<Integer> intStream) {
        return intStream.map(i -> "count:" + i);
    }
```

### Some theory on `map` operation

Note: Trying to use Java notation. This is not standard notation.

* `Set`, `Set`, etc. are container types.
* Lets container type: `C` and the elements inside the container type: `E`.
* We can now say that container is of type `C<E>`
* If we have a function of Java type `Function<E, F>`
  * easier to understand rewritten as `E -> F` and apply this function to `C<E>`
  * the result will be of type `C<F>`
* This application is called the `map` operation.
* So: `C<E>.map(E -> F) =>> C<F>`

This should be easy!

### Compose using `flatMap`

* For type `C<E>`, if we apply function `E -> C<F>`, what will we get?
* More concrete example: If we have `List<String>` and a function `String -> List<Character>` i.e. split the String into a list of characters, what will a `map` operation do?
* Answer, `List<List<Character>>`
* Or `C<C<F>>`
* So what if we want a list of all characters in all strings? We have to flatten it.
* Here's where `flatMap` comes in handy.
* So, in theory: `C<E>.flatMap(E -> C<F>) =>> C<F>`
* We flatMap the container `C<E>` with a container type `C<F>` and get a new combined result of type `C<F>`.

Clear?

### Try to `flatMap` two lists:

Sorry for using the Scala REPL, just less cumbersome:

```scala
scala> val l1 = List("one", "two", "three")
l1: List[String] = List(one, two, three)

scala> val l2 = l1.flatMap(s => s.toList)
l2: List[Char] = List(o, n, e, t, w, o, t, h, r, e, e)

scala> val l3 = l1.map(s => s.toList)
l3: List[List[Char]] = List(List(o, n, e), List(t, w, o), List(t, h, r, e, e))
```
### Combining with `map` and `flatMap`

* What if we want to operate on multiple containers together, combining them.
* Given containers `C<E>` and `C<F>`, we want to apply a function `(E, F) -> T` and get a `C<T>` out of it.
* The combining would look like this: `C<E>.flatMap(E -> C<F>.map(F -> [(E, F) -> T]))`
* Similarly, combining `C<E>`, `C<F>`, `C<G>` applying `(E, F, G) -> T` looks like this: `C<E>.flatMap(E -> C<F>.flatMap(F -> C<G>.map(G -> [(E, F, G) -> T])))` and gives you a `C<T>`.

### Combining & Processing example with `Optional`

```java
    public static Optional<String> combine(Optional<String> o1, Optional<String> o2) {
        return o1.flatMap(s1 -> o2.map(s2 -> s1 + s2));
    }
```

* Now try `combine(Optional.of("Hello"), Optional.of("World"))`
* And try to make one or the other empty.

### Combining Futures

* Why? Because you want to combine futures and not block on each or any future.
* You need to operate on the content of the future.
* And Future is just another container type. Well, in Java 8 it is called `CompletionStage` and a more concrete implementation called `CompletableFuture`.
* Same composition rules applies, but...
* The Java 8 names are weird.

### Some name mappings:

* `thenAccept` : `foreach`
* `thenApply` : `map`
* `thenCompose` : `flatMap`

The `CompletionStage` provides a set of `*Async` signatures to run these operations on a different thread pool. You can pass the thread pool or it will run on a default thread pool.

`CompletionStage` provides the `thenCombine` signature allowing combination of two `CompletionStages`. We need to cascade `thenCombine` to combine more than two.

We can also combine `flatMap` and `map` as seen above with `Optional`, using `thenCompose... thenCompose...thenApply`.

### Using Combined Futures

* Sync CompletionStage operations can be used safely inside an Actor as long as it is satisfied by messages to this same actor.
* Async CompletionStage operations need to use the `pipe` operation to send an actor message to this or a different actor, without blocking.

### Exercise:

Write a test case that calls `PaymentActor` by `ask` multiple times (multiple `PaymentRequests`) and combine the resulting `CompletionStage`s into a single one. Then block on this resulting `CompletionStage` to finish the test case. The `CompletionStage` should contain a `List` of all results.
