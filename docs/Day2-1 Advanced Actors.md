## Day 2: Advanced Actors

### Using `ask`

So far we have learned about `tell`, the fire-and-forget sender, and `forward`. There is another way to send an actor a message and expect a response back, even in production and not test code, by using `ask`. Lets add another test using just this to `PaymentActorTest`:

First we need to deal with an import:

```java
import static akka.pattern.PatternsCS.*;
```

Then the test itself:

```java
    @Test
    public void askLowRisk() throws Exception {
        new TestKit(system) {{
            CompletionStage<Object> cs = ask(paymentActor, new PaymentRequest(
                            new RecordId(creator, id++), 1001, 2001, 1000),
                    1000);
            
            CompletionStage<PaymentResponse> responseCS = cs.thenApply(o -> (PaymentResponse) o);

            // WARNING: CompletableFuture.get() blocks!
            // Use only for test code!!!
            PaymentResponse response = responseCS.toCompletableFuture().get();

            assertEquals(PaymentStatus.APPROVED, response.status);
        }};
    }
```

The `CompletionStage` may succeed or fail. To signal a failure to an `ask`, the actor must respond back with a `akka.actor.Status.Failure`. Lets try that with our `matchAny` case. Modify `PaymentActor`'s `matchAny` to send back a `Failure` when that case is matched:

Once again, the import:

```java
import akka.actor.Status;
```

Then in the `PaymentActor` modify the `matchAny` to be as follows:

```java
        rcvBuilder.matchAny(o -> {
            log.error("Message of type {} received is not understood", o.getClass().getName());
            sender().tell(new Status.Failure(new IllegalArgumentException("Message type not understood")), self());
            // ^^^ This line added ^^^
        });
```

Lets try this out in our test. Lets add a test that checks for such failure to `PaymentActorTest`:

```java
    @Test
    public void invalidMessage() throws Exception {
        new TestKit(system) {{
           CompletionStage<Object> cs = ask(paymentActor, "Pay me $1,000,000 or else ...", 1000);
           CompletionStage<Object> recovered = cs.handle((o, e) -> Optional.ofNullable(o).orElse(e));
           Object response = recovered.toCompletableFuture().get();
           assertEquals(IllegalArgumentException.class, response.getClass());
        }};
    }
```

### Changing Actor Behavior

Now we come to the last property of actors. It can change behavior based on a message. Lets try having an `offline` behavior for our actors. If we send an `Offline` message to our `PaymentActor`, it will only send an error back. Once we send an `Online` message to the actor, it will start resume processing. Here we change the state of the actor to off-line and have it behave differently in offline mode. For that, we need our `Offline` and our `Online` messages first. Lets also make them singletons to save on any GC cost. They are just signals and don't contain state.

```java
public class Offline {

    private static Offline instance = new Offline();

    public static Offline getInstance() {
        return instance;
    }
}
```

```java
public class Online {

    private static Online instance = new Online();

    public static Online getInstance() {
        return instance;
    }
}
```

Next, lets add a Receive matcher to our `PaymentActor` to build the offline behavior. This is a final variable to our receive and should never change:

```java
    private final Receive offlineBehavior = receiveBuilder()
            .match(Online.class, o -> {
                getContext().unbecome();
                sender().tell(o, self());
            })
            .match(PaymentRequest.class, p -> {
                log.error("Received PaymentRequest but still Offline");
                sender().tell(Offline.getInstance(), self());
            })
            .matchAny(o -> {
                log.error("Message of type {} received is not understood", o.getClass().getName());
                sender().tell(new Status.Failure(new IllegalArgumentException("Message type not understood")), self());
            }).build();
```

Add the receive match for `Offline` to take this actor to its `offlineBehavior`. This match can go anywhere before the `rcvBuilder.matchAny`:

```java
        builder.matchEquals(Offline.getInstance(), o -> {
            getContext().become(offlineBehavior, false);
            sender().tell(o, self());
        });
```

The `getContext().become(...)` call moves the actor to `offlineBehavior`. And we can see in the `offlinePF()` the `getContext().unbecome()` will set the behavior back to original. If the `discardOld` parameter to `getContext().become(...)` is set to `true`, we can only keep shifting to new behavior. This is to prevent memory leaks caused by behavior change.

Lets build a test for this. This test is a bit longer, though. It checks the behavior before going off-line, while off-line, and after coming back on-line. Here is the test code:

```java
    @Test
    public void testOffline() throws Exception {
        new TestKit(system) {{
            // Try a small payment request, all should be good.
            CompletionStage<Object> cs = ask(paymentActor, new PaymentRequest(
                            new RecordId(creator, id++), 1001, 2001, 1000),
                    1000);
            CompletionStage<PaymentResponse> responseCS = cs.thenApply(o -> (PaymentResponse) o);

            PaymentResponse response = responseCS.toCompletableFuture().get();

            assertEquals(PaymentStatus.APPROVED, response.status);

            // Now take it offline.
            CompletionStage<Offline> cs2 = ask(paymentActor, Offline.getInstance(),
                    1000).thenApply(o -> (Offline) o);
            Offline offLineConfirm = cs2.toCompletableFuture().get();
            assertEquals(Offline.getInstance(), offLineConfirm);

            // Lets try to send it 5 payment requests. Each should return offline.
            for (int i = 0; i < 5; i++) {
                CompletionStage<Offline> cs3 = ask(paymentActor, new PaymentRequest(
                                new RecordId(creator, id++), 1001, 2001, 1000),
                        1000).thenApply(o -> (Offline) o);
                Offline offLineResponse = cs3.toCompletableFuture().get();
                assertEquals(Offline.getInstance(), offLineResponse);
            }

            // Next we turn the PaymentActor back on-line
            CompletionStage<Online> cs4 = ask(paymentActor, Online.getInstance(),
                    1000).thenApply(o -> (Online) o);
            Online onLineConfirm = cs4.toCompletableFuture().get();
            assertEquals(Online.getInstance(), onLineConfirm);

            // Now we should be back in business
            CompletionStage<PaymentResponse> cs5 = ask(paymentActor, new PaymentRequest(
                            new RecordId(creator, id++), 1001, 2001, 800),
                    1000).thenApply(o -> (PaymentResponse) o);

            PaymentResponse response5 = cs5.toCompletableFuture().get();

            assertEquals(PaymentStatus.APPROVED, response5.status);
        }};
    }
```

### Stashing Messages

In some cases we don't just want to say we're offline. We want to stash the received messages till we're back online and deal with them at that time. This is often used in actors that act as caches. There is a short time while loading/reloading caches we just want to hold onto requests until our data is fully loaded.

For this exercise, we're going to replace some logic we have built. In order to deal with this scenario, we want to make a copy of our `PaymentActor` into a new class called `StashingPaymentActor`. You'll need to copy and modify all references to `PaymentActor` to `StashingPaymentActor`.

We will then change the superclass to `AbstractActorWithStash`:

```java
public class StashingPaymentActor extends AbstractActorWithStash {
```

Next lets go to the `offlineBehavior` and make it stash the messages instead of just sending an off-line response. Similarly, we need to unstash everything at the time we go back on-line.:

```java
    private final Receive offlineBehavior = receiveBuilder()
            .match(Online.class, o -> {
                unstashAll();
                // ^^^ Here's your unstash change ^^^
                getContext().unbecome();
                sender().tell(o, self());
            })
            .match(PaymentRequest.class, p -> {
                log.error("Received PaymentRequest but still Offline");
                stash();
                // ^^^ Here's your stash change ^^^
            })
            .matchAny(o -> {
                log.error("Message of type {} received is not understood", o.getClass().getName());
                sender().tell(new Status.Failure(new IllegalArgumentException("Message type not understood")), self());
            }).build();
```

Then we want to handle all stashed messages when we get back online.

Now it should all work. Similarly, we want to make a copy of `PaymentActorTest` into `StashingPaymentActorTest` to make the test modifications.

We'll now modify the `testOffline()` method to test the stashing behavior. Hers is our new `testOffline()` method in `StashingPaymentActorTest`:

```java
    @Test
    public void testOffline() throws Exception {
        new TestKit(system) {{
            // Try a small payment request, all should be good.
            CompletionStage<Object> cs = ask(paymentActor, new PaymentRequest(
                            new RecordId(creator, id++), 1001, 2001, 1000),
                    1000);
            CompletionStage<PaymentResponse> responseCS = cs.thenApply(o -> (PaymentResponse) o);

            PaymentResponse response = responseCS.toCompletableFuture().get();

            assertEquals(PaymentStatus.APPROVED, response.status);

            // Now take it offline.
            CompletionStage<Offline> cs2 = ask(paymentActor, Offline.getInstance(),
                    1000).thenApply(o -> (Offline) o);
            Offline offLineConfirm = cs2.toCompletableFuture().get();
            assertEquals(Offline.getInstance(), offLineConfirm);

            // Lets try to send it 5 payment requests. Each should return offline.
            for (int i = 0; i < 5; i++) {
                paymentActor.tell(new PaymentRequest(new RecordId(creator, id++), 1001,
                        2001, 1000), getRef());
                expectNoMsg(duration("1 second"));
            }

            // Next we turn the stashingPaymentActor back on-line
            CompletionStage<Online> cs4 = ask(paymentActor, Online.getInstance(),
                    1000).thenApply(o -> (Online) o);
            Online onLineConfirm = cs4.toCompletableFuture().get();
            assertEquals(Online.getInstance(), onLineConfirm);

            // Receive the stashed messages
            List<Object> responses = receiveN(5, duration("5 seconds"));
            for (Object resp: responses) {
                assertEquals(PaymentResponse.class, resp.getClass());
                assertEquals(PaymentStatus.APPROVED, ((PaymentResponse) resp).status);
            }

            // Now we should be back in business
            CompletionStage<PaymentResponse> cs5 = ask(paymentActor, new PaymentRequest(
                            new RecordId(creator, id++), 1001, 2001, 800),
                    1000).thenApply(o -> (PaymentResponse) o);

            PaymentResponse response5 = cs5.toCompletableFuture().get();

            assertEquals(PaymentStatus.APPROVED, response5.status);
        }};
    }
```

### Actor Lifecycle

An actor keeps on living, until they are sent a `PoisonPill`, call `getContext().stop(self())` internally, or `getContext().stop(actorRef)` externally.

It is important to terminate actors when no longer in use. If we keep creating actors and not terminate them, we have a memory leak.

![Image showing actor lifecycle](https://doc.akka.io/docs/akka/current/images/actor_lifecycle.png)

### What we have not covered

* Piping results from async operations
* FSM
* Supervisor Policy

Please do read up on these in your own time.

Now we are done with Actors. Yeah!
