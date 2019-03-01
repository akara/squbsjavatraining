## Day 3: Intro to Akka Streams

### Core Concepts

* Source - Source stream components
* Flow - Processing components
* Sink - End stream components
* Materializer - Engine that makes the stream tick
* Materialized Values - The resulting value of a stream
* Back-pressure

### Writing your First Stream

We start with writing our first, very simple stream. Again, we'll expand on this first application later to build a more sophisticated, and more useful stream. You do not necessarily have to build a stream inside an `Actor`, but we will as well start with that.

Create Java class `PaymentStreamActor` into your package of choice under `src/main/java` of your cube project with the following content:

```java
import akka.NotUsed;
import akka.actor.AbstractActor;
import akka.stream.ActorMaterializer;
import akka.stream.Materializer;
import akka.stream.javadsl.Source;

public class PaymentStreamActor extends AbstractActor {

    final Materializer mat = ActorMaterializer.create(getContext());

    final Source<Integer, NotUsed> src = Source.range(1, 100);

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .matchAny(o -> run())
                .build();
    }

    private void run() {
        src.runForeach(System.out::println, mat);
    }

}
```

And to make it tick, we also want to write our test code. Create `PaymentStreamTest` into your package of choice under `src/test/java` of your cube project with the following content:

```java
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;
import akka.testkit.javadsl.TestKit;
import org.junit.AfterClass;
import org.junit.BeforeClass;
import org.junit.Test;

public class PaymentStreamTest {

    private static ActorSystem system;
    private static ActorRef streamActor;

    @BeforeClass
    public static void setup() {
        system = ActorSystem.create();
        streamActor = system.actorOf(Props.create(PaymentStreamActor.class), "PaymentStream");
    }

    @AfterClass
    public static void teardown() {
        TestKit.shutdownActorSystem(system);
        system = null;
    }

    @Test
    public void pingStreamActor() {
        new TestKit(system) {{
            streamActor.tell("ping", getRef());
        }};

        // Just temporary, as we want to see the result.
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

Click on the `>>` next to the class declaration and select `Run 'PaymentStreamTest'`. You should see it print out the numbers 1 to 100. You got your first stream!

### Adding the Sink

So far, we used a shortcut method `runForeach` that installs a sink and just runs it in one shot. There are a lot of such shortcut methods in Akka Steams to make sure you do not have to write more code than needed. But now lets make it all very clear. Add a sink just below where the source was declared:

```java
    final Source<Integer, NotUsed> src = Source.range(1, 100);

    final Sink<Integer, CompletionStage<Done>> sink = Sink.foreach(System.out::println);
    // Add this line ^^^^^^^^^^
```

Next we also want to formalize the run a bit better for our own understanding. Lets re-write the `run()` method to be as followings:

```java
    private void run() {
        final RunnableGraph<NotUsed> stream = src.to(sink);
        stream.run(mat);
    }
```

Here we can now see that we essentially create a runnable stream graph `RunnableGraph` where we don't care about the materialized value, so `NotUsed`. And we explicitly run it with a materializer, the piece that makes the stream tick.

### Adding a Flow component

Now that we have the source and sink components, lets add some processing to the middle of the stream. First we'll only take our even numbers using the filter. Lets create a filtering flow:

```java
    final Source<Integer, NotUsed> src = Source.range(1, 100);
    
    final Flow<Integer, Integer, NotUsed> filter = Flow.of(Integer.class).filter(i -> i % 2 == 0);
    // Add this line ^^^^^^^^

    final Sink<Integer, CompletionStage<Done>> sink = Sink.foreach(System.out::println);

```

Then lets put our filter to work. Modify our `run()` method as follows:

```java
        final RunnableGraph<NotUsed> stream = src.via(filter).to(sink);
        stream.run(mat);
```

And run it again. Now you'll notice half the numbers are no longer printed.

We can also add another stream component, another flow, to calculate the square of each number. Lets do so.

```java
    final Flow<Integer, Integer, NotUsed> filter = Flow.of(Integer.class).filter(i -> i % 2 == 0);

    final Flow<Integer, Integer, NotUsed> square = Flow.of(Integer.class).map(i -> i * i);
    // Add this line ^^^^^^^^^

    final Sink<Integer, CompletionStage<Done>> sink = Sink.foreach(System.out::println);
```

Then, also add the `square` to the stream.

```java
    final RunnableGraph<NotUsed> stream = src.via(filter).via(square).to(sink);

```

And just run it. Well, there is nothing unexpected to the results.

Here we just created two separate flow components and then composed them together. We could as well define all these as a single component, such as:

```java
    final Flow<Integer, Integer, NotUsed> squareEven = Flow.of(Integer.class).filter(i -> i % 2 == 0).map(i -> i * i)
```

This definitely works. But the filter and square components are no longer easily re-usable. It may be worth grouping some complex logic together this way, though.

But, we have just learned one more thing. A stream is not always one in and one out. On the contrary, the number of elements might shrink (with `filter`) or might grow from processing stage to processing stage (for instance, with `mapConcat` or `flatMapConcat`).

### Stream Components are Re-usable

In the later examples, you can clearly see we have separated the declaration of the stream to the running of the stream with the `stream.run(mat)`.

It is to be noted that each of the stream components by themselves are reusable. They can be shipped around in any form, returning from a method call, sent around via actor messages, etc. These logic components can then be composed into the final runnable form and run just about anywhere. Even the final `RunnableGraph` is reusable and can be shipped around. We can think of them as templates of stream logic that is being shipped around and manipulated until finally they get to run via the `run` command. The `src.runForeach` call we had in the beginning is just a shortcut of all this.

### Stream Composition & Componentization

Stream components are composable to build more complex components. Each of them can then be shipped around and built into a final `RunnableGraph`. It is important to understand the result types of such composition, though.

* `Source ~> Flow` ==> `Source`
* `Flow ~> Flow` ==> `Flow`
* `Flow ~> Sink` ==> `Sink`

Lets try out some declarations here:

```java
    final Source<Integer, NotUsed> filteredSource = src.via(filter);

    final Flow<Integer, Integer, NotUsed> filterAndSquare = filter.via(square);

    // Use materialized value of flow
    final Sink<Integer, NotUsed> squareAndPrint = square.to(sink);
    
    // Choose materialized value of sink
    final Sink<Integer, CompletionStage<Done>> squareAndPrintMat = square.toMat(sink, Keep.right());
```

Each of these now become more complex stream components that also can be shipped around and re-used. This is very handy in declaring and composing more and more complex components that will become part of the main flow.

### Materialized Value

The materialized value is the resulting value of a stream when it runs. In our examples so far, we really did not care about the materialized value just yet. but lets say we wanted to calculate the sum of squares as a result of the stream, we can do the `println` as a map stage separately. Then we can have a sink that sums it. Lets do that:

```java
    final Flow<Integer, Integer, NotUsed> print = Flow.of(Integer.class).map(i -> {
        System.out.println(i);
        return i;
    });

    final Sink<Integer, CompletionStage<Integer>> sum = Sink.reduce((a, b) -> a + b);
```

And then our run method will just be like this:

```java
        final RunnableGraph<CompletionStage<Integer>> stream =
                src.via(filter).via(square).via(print).toMat(sum, Keep.right());
        stream.run(mat);
```

Hold on, we just summed up the result of the stream. But where did it go? Of course, that sum would only be available once the stream is done running. That's why we get a `CompletionStage<Integer>`. We can wait for that. But remember, waiting **is a crime** in this architecture. So we need to be more creative. We can do many things with that `CompletionStage` but for now we want to send it back to the test that asked for it. Lets use one of the actor features we have not covered so far, the "pipe". Lets change our `run` method as follows:

```java
    import static akka.pattern.PatternsCS.pipe;

    private void run() {
        final RunnableGraph<CompletionStage<Integer>> stream =
                src.via(filter).via(square).via(print).toMat(sum, Keep.right());
        final CompletionStage<Integer> resultFuture = stream.run(mat);
        pipe(resultFuture, getContext().dispatcher()).to(sender());
    }
```

Now lets change our test case to expect that result back. Go to our test and change it as follows:

```java
    @Test
    public void pingStreamActor() {
        new TestKit(system) {{
            streamActor.tell("ping", getRef());
            int sum = expectMsgClass(Integer.class);
            assertEquals(171700, sum);
        }};
    }
```

We can also now take out the `sleep` at the end. As this test won't quit until it receives the response, we no longer have to compensate for test quitting before the test logic is done.

Now run the test and have some fun.

### Graph

So far, we have been dealing only with relatively linear streams. But most applications, stream segments or components cannot be linear. The stream itself does not at all need to be linear. And while it is possible to compose non-linear streams using the current stream syntax, it can become hard to deal with. That's where the stream graph syntax comes in very handy.

The graph is defined by a GraphDSL, which introduces a certain boilerplate as follows:

```java
        final RunnableGraph<CompletionStage<Integer>> graph =
                RunnableGraph.fromGraph(GraphDSL.create(sum, (builder, out) -> {
                            final UniformFanOutShape<Integer, Integer> bcast = builder.add(Broadcast.create(2));
                            final UniformFanInShape<Integer, Integer> merge = builder.add(Merge.create(2));
                            final FlowShape<Integer, Integer> filterS = builder.add(filter);
                            final FlowShape<Integer, Integer> squareS = builder.add(square);
                            final FlowShape<Integer, Integer> printS = builder.add(print);

                            final Outlet<Integer> source = builder.add(src).out();
                            builder.from(source).viaFanOut(bcast).via(filterS).viaFanIn(merge).via(printS).to(out);
                                              builder.from(bcast).via(squareS).toFanIn(merge);

                            return ClosedShape.getInstance();
                        }));
```

The `GraphDSL` block looks a little big. So lets break it down into the components:

1. The declaration and the factory. The case above shows the type as `RunnableGraph`, which means this graph is a full template and can be run immediately. However, a `GraphDSL` does not always need to create a `RunnableGraph`. It can as well create a `Flow`, `Source`, or `Sink`. These can again be composed with other components using `GraphDSL` or simple stream operations. To construct a `RunnableGraph` from a `GraphDSL`, you'd construct it with `RunnableGraph.fromGraph(...)`. Similarly, to construct a `Flow`, you'd start with `Flow.fromGraph(...)`. Similar methods exist for `Source.fromGraph(...)` and `Sink.fromGraph(...)`.

2. The graph itself is created by `GraphDSL.create(...)`. The `GraphDSL.create(...)` is a highly overloaded method. It can take no arguments to a very large number of arguments. The format shown in this example passes only the `sum` sink in. By doing so, we're saying we want this `RunnableGraph`/`Flow`/`Source`/`Sink` to to use the materialized value of `sum` as the materialized value of this graph. If no parameter is passed, the materialized value is simply a `CompletionStage<Done>`.

   Life gets more interesting when we pass multiple stream components into `GraphDSL.create(...)` In that case we have multiple materialized value for the graph. This is of course not valid. So we need to pass another lambda in called `combineMat`. This lambda will take *n* arguments where *n* is the number of stream stages passed in. The output is then the materialized value based on combining the input materialized values. This allows and enforces developers to define how to produce the final materialized value from the multiple values in question.

   The last argument of the `GraphDSL.create` is the builder lambda. This is a lambda that takes a `builder` as the first argument, and an imported version of the stream stages passed in as the first set of arguments to `GraphDSL.create`. For instance, if you have 3 stages passed in, you'll have the `builder` plus 3 arguments representing those imported stages. These will be used to compose the stream graph.
   
3. `builder.add(...)` calls. Every stream stage not passed in (and hence no materialized value captured) can be made available to the composition by passing it to the `builder.add(...)` calls. Note that the type of is appended with "Shape". For instance, a `Flow` stage is imported to a `FlowShape`. Add as many components as needed to compose the stream graph. Do not add the ones passed into `GraphDSL.create(...)` directly. 
4. The graph composition. We go through all the trouble just for this alone. This is where you describe how the stages/components connect to each others.

5. The return value. These need to match the type of graph you're composing. Depending on the type of composition, you want to return the relevant shape for that composition. Here is the return value for common stage types:

   | Type            | Return value                          |
   | --------------- | ------------------------------------- |
   | `RunnableGraph` | `ClosedShape.getInstance()`           |
   | `Source`        | `SourceShape.of(outputPort)`          |
   | `Sink`          | `SinkShape.of(inputPort)`             |
   | `Flow`          | `FlowShape.of(inputPort, outputPort)` |


Enough said. Lets get some code going. We'll create a new Java class that looks very much like our `PaymentStreamActor`. Lets call it `PaymentGraphActor`:

```java
import akka.Done;
import akka.NotUsed;
import akka.actor.AbstractActor;
import akka.stream.*;
import akka.stream.javadsl.*;

import java.util.concurrent.CompletionStage;

import static akka.pattern.PatternsCS.pipe;

public class PaymentGraphActor extends AbstractActor {

    final Materializer mat = ActorMaterializer.create(getContext());

    final Source<Integer, NotUsed> src = Source.range(1, 100);

    final Flow<Integer, Integer, NotUsed> filter = Flow.of(Integer.class).filter(i -> i % 2 == 0);

    final Flow<Integer, Integer, NotUsed> square = Flow.of(Integer.class).map(i -> i * i);

    final Sink<Integer, CompletionStage<Done>> sink = Sink.foreach(System.out::println);

    final Flow<Integer, Integer, NotUsed> print = Flow.of(Integer.class).map(i -> {
        System.out.println(i);
        return i;
    });

    final Sink<Integer, CompletionStage<Integer>> sum = Sink.reduce((a, b) -> a + b);

    @Override
    public Receive createReceive() {
        return receiveBuilder()
                .matchAny(o -> run())
                .build();
    }

    private void run() {

        final RunnableGraph<CompletionStage<Integer>> graph =
                RunnableGraph.fromGraph(GraphDSL.create(sum, (builder, out) -> {
                    final UniformFanOutShape<Integer, Integer> bcast = builder.add(Broadcast.create(2));
                    final UniformFanInShape<Integer, Integer> merge = builder.add(Merge.create(2));
                    final FlowShape<Integer, Integer> filterS = builder.add(filter);
                    final FlowShape<Integer, Integer> squareS = builder.add(square);
                    final FlowShape<Integer, Integer> printS = builder.add(print);

                    final Outlet<Integer> source = builder.add(src).out();
                    builder.from(source).viaFanOut(bcast).via(filterS).viaFanIn(merge).via(printS).to(out);
                    builder.from(bcast).via(squareS).toFanIn(merge);

                    return ClosedShape.getInstance();
                }));
        final CompletionStage<Integer> resultFuture = graph.run(mat);
        pipe(resultFuture, getContext().dispatcher()).to(sender());
    }
}
```

And for that, we also create the test `PaymentGraphTest` Java class:

```java
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;
import akka.testkit.javadsl.TestKit;
import org.junit.AfterClass;
import org.junit.BeforeClass;
import org.junit.Test;

import static org.junit.Assert.assertEquals;

public class PaymentGraphTest {

    private static ActorSystem system;
    private static ActorRef streamActor;

    @BeforeClass
    public static void setup() {
        system = ActorSystem.create();
        streamActor = system.actorOf(Props.create(PaymentGraphActor.class), "PaymentGraph");
    }

    @AfterClass
    public static void teardown() {
        TestKit.shutdownActorSystem(system);
        system = null;
    }

    @Test
    public void pingStreamActor() {
        new TestKit(system) {{
            streamActor.tell("ping", getRef());
            int sum = expectMsgClass(Integer.class);
            assertEquals(340900, sum);
        }};
    }
}
```

### `BidiFlow` - Bidirectional Flows

In many cases, it is useful to model a stream or stream component as bidirectional flows. More commonly, protocol stacks can be modeled as bidirectional flows. Even the squbs pipeline in 0.9 is modeled as a set of `BidiFlow`s stacked on top of each others.

###### Anatomy of a `BidiFlow`

```java
        +------+
  In1 ~>|      |~> Out1
        | bidi |
 Out2 <~|      |<~ In2
        +------+
```

###### Stacking of `BidiFlow`s

```java
       +-------+  +-------+  +-------+
  In ~>|       |~>|       |~>|       |~> toFlow
       | bidi1 |  | bidi2 |  | bidi3 |
 Out <~|       |<~|       |<~|       |<~ fromFlow
       +-------+  +-------+  +-------+

       bidi1.atop(bidi2).atop(bidi3);
```

But, the `BidiFlow` is not only useful for "literally" bidirectional flows. In some cases we need to gate off a part of the flow with a single entry and exit point of control at those places. `BidiFlow`s are extremely useful in such cases. The following squbs flow components are `BidiFlow`s:

* TimeoutBidiFlow
* CircuitBreakerBidi

As we can see, these stages fend off a section of the stream to test whether a message passed through that stream part in timely or not, in error or not. It also has the capability to short circuit the flow with alternatives paths (like timeout messages) at its entry/exit point as can be seen in the following sample:

```java
       +---------+  +------------+
  In ~>|         |~>|            |
       | timeout |  | processing |
 Out <~|         |<~|            |
       +---------+  +------------+
       
       timeout.join(processing);
```

Here, timeout is a `BidiFlow` where processing is just a regular flow stage (or combination of flow stages). The arrow is just pointed backwards to make it connect. It only has one input and one output port.

### Stream Stages

So far we have shown only a few most common stages, such as `Flow.map`, `Flow.filter`, `Broadcast`, `Merge`, etc. Akka Streams comes with a multitude of stages. Lets point your browser to [Overview of built-in stages and their semantics](https://doc.akka.io/docs/akka/current/stream/operators/index.html?language=java) and look at some of the many stages provided by Akka.

If the Akka-provided stages are not adequate, there are more stages provided as part of [Alpakka](https://developer.lightbend.com/docs/alpakka/current/). squbs (listed in the Alpakka catalog) also provides several interesting stream stages:

* PersistentBuffer (with and without commit stage)
* BroadcastBuffer (with and without commit stage)
* Timeout
* CircuitBreaker

#### Custom Stages

If stages we can find still do not fit requirements, we can build our own stage. This is documented at [Custom stream processing](https://doc.akka.io/docs/akka/current/stream/stream-customize.html?language=java). Since testing and qualification of custom stream stages are pretty elaborate, plese engage the squbs team. Also, if you think your requirements are more generic than just your use case, squbs is also welcomes your contributions!

### Stream Cookbook

For ideas how to satisfy your requirements with Streams, the [Streams Cookbook](https://doc.akka.io/docs/akka/current/stream/stream-cookbook.html?language=java) is an invaluable resource to get your ideas.
