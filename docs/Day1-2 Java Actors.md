# Akka Java8/squbs Training

## Day1: Your first Actors

### Create Status Enum and Message Classes

1. Create enum `PaymentStatus`

   ```java
   public enum PaymentStatus {
       ACCEPTED,
       APPROVED,
       REJECTED,
   }
   ```

2. Create class RecordId:
   
   ```java
   public class RecordId {

       /** The creation time of this identifier. */
       public final long creationTime;

       /** The identity of the creator */
       public final long creator;

       /** A running number created and assigned by the creator */
       public final long id;
   }
   ```
   
   Optionally, you may want to make the fields `private`, with some consequences... later.

2. In the IDE: `Code` -> `Generate...` -> `Constructor`

   Select fields `creator` and `id` as constructor parameter.
    
3. Add code to assign the `creationTime` from `System.currentTimeMillis()` to the constructor.

   ```java
       this.creationTime = System.currentTimeMillis();
   ```
   
4. `Code` -> `Generate...` -> `equals() and hashCode()` and select `JDK7+ hashCode generation`.

5. If you decide to make the fields private, you also need to generate the getters in the same manner. Just more boilerplate code. The samples here do not use getters.

6. Now, do the same generation for `PaymentRequest` and `PaymentResponse` with following fields:

   ```java
   public class PaymentRequest {

       /** Unique record identifier. */
       public final RecordId id;

       /** Payer account. */
       public final long payerAcct;

       /** Payee account. */
       public final long payeeAcct;

       /** Amount, including two decimal points. Divide by 100D to get the dollar amount */
       public final long amount;
   }
   ```
   
   ```java
   public class PaymentResponse {

       /** Record id for this response. */
       public final RecordId id;
    
       /** Record id for the corresponding request. */
       public final RecordId requestId;
    
       /** The status of the payment request. */
       public final PaymentStatus status;   
   }
   ```

### Create our Fundamental Actor

We'll expand on this actor a bit later. So lets start with something very basic:

```java
import akka.actor.AbstractActor;
import akka.event.Logging;
import akka.event.LoggingAdapter;
import akka.japi.pf.ReceiveBuilder;

public class PaymentActor extends AbstractActor {

    private final LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    private final long creatorId = 100;

    private long id = 1;

    @Override
    public Receive createReceive() {
        ReceiveBuilder builder = ReceiveBuilder.create();

        builder.match(PaymentRequest.class, request -> {
                    log.info("Received payment request of ${} accounts {} => {}",
                            request.amount / 100D, request.payerAcct, request.payeeAcct);
                }
        );

        builder.matchAny(o ->
                log.error("Message of type {} received is not understood", o.getClass().getName())
        );

        return builder.build();
    }
}
```

So far our actor is not doing a lot. It receives a message for a payment and just logs. Nothing very useful. But at least we have a running actor.

### Register Your Actor

So that squbs automatically starts the actor. Add your actor to your cube project's `src/main/resources/META-INF/squbs-meta.conf`.

```
cube-name = com.paypal.myorg.myservcube
cube-version = "0.0.1-SNAPSHOT"
squbs-actors = [
  {
    class-name = com.paypal.myorg.myserv.cube.PaymentActor
    name = paymentActor
  }
]
```

### Testing Your Actor

1. Edit your cube project's `build.sbt`.
2. Add dependencies for JUnit:

   ```scala
     "junit" % "junit" % "4.12" % "test",
     "com.novocode" % "junit-interface" % "0.11" % "test->default"
   ```

3. Add settings for JUnit:

   ```scala
   testOptions in Test += Tests.Argument(TestFrameworks.JUnit, "-v", "-a")
   ```
   
4. Save and refresh project. You'll be asked by the IDE.
5. Create folder `java` under src/test.
6. Under `src/test/java`, create package with the same name as the package containing your actor.
7. Create the test file `PaymentActorTest` with a simple call to the actor as follows:

   ```java
   import akka.actor.ActorRef;
   import akka.actor.ActorSystem;
   import akka.actor.Props;
   import akka.testkit.javadsl.TestKit;
   import org.junit.AfterClass;
   import org.junit.BeforeClass;
   import org.junit.Test;

   public class PaymentActorTest {

       private static final long creator = 200L;
       private static long id = 1;

       private static ActorSystem system;
       private static ActorRef paymentActor;

       @BeforeClass
       public static void setup() {
           system = ActorSystem.create();
           paymentActor = system.actorOf(Props.create(PaymentActor.class), "PaymentActor");
       }

       @AfterClass
       public static void teardown() {
           TestKit.shutdownActorSystem(system);
           system = null;
       }

       @Test
       public void pingPaymentActor() {
           new TestKit(system) {{
               paymentActor.tell(new PaymentRequest(new RecordId(creator, id++), 1001, 2001, 30000), getRef());
           }};

           // Temporarily have the sleep here so the test does not terminate prematurely.
           try {
               Thread.sleep(1000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
   }
   ```
   
8. You'll see a double-arrow `>>` icon to the left of the `PaymentActorTest` class declaration. Click on the `>>` and select `Run 'PaymentActorTest'. See your test results. Since our actor does no assertioins but only logs, we can observe it running by looking at the logs. We'll add assertions in the next step.
9. If you enabled the toolbar, you'll see `PaymentActorTest` in list of items you can run. Select the test and press the `>` icon to start the test at any future time. Since our actor does nothing but log, we can observe it running by looking at the logs.

### Make the Actor send PaymentResponse back

1. Edit `PaymentActor` and add the following lines:

   ```java
           builder.match(PaymentRequest.class, request -> {
                   log.info("Received payment request of ${} accounts {} => {}",
                           request.amount / 100D, request.payerAcct, request.payeeAcct);
                   PaymentResponse response =
                           new PaymentResponse(new RecordId(creatorId, id++), request.id, PaymentStatus.ACCEPTED);
                   sender().tell(response, self());
                   // ^^^ Add these two lines here ^^^
               }
           );

   ```

2. Since `PaymentActor` now responds with a message, lets update our test expecting a message back, too. First add assertion import:

   ```java
   import static org.junit.Assert.*;
   ```

   Then update the test to obtain response messages.
   
   ```java
    @Test
    public void pingPaymentActor() {
        new TestKit(system) {{
            paymentActor.tell(new PaymentRequest(new RecordId(creator, id++), 1001, 2001, 30000), getRef());
            PaymentResponse response = expectMsgClass(PaymentResponse.class);
            assertEquals(PaymentStatus.ACCEPTED, response.status);
            // ^^^ Add the 2 lines above. ^^^
        }};

        // Temporarily have the sleep here so the test does not terminate prematurely.
        // try {
        //     Thread.sleep(1000);
        // } catch (InterruptedException e) {
        //     e.printStackTrace();
        // }
        // ^^^ Remove the sleep, commented lines here ^^^
    }

   ```

### Qualifying Messages

Sometimes we want to treat different messages arriving a bit different. For instance, a cafeteria payment of $12 or less should just be auto-approved. So lets add another matcher.

**Note:** This matcher needs to be added above the current matcher as the match is done in sequence this it is more specific than the existing one.

```java
        builder.match(PaymentRequest.class, request -> request.amount < 1200, request -> {
                    log.info("Received small payment request of ${} accounts {} => {}",
                            request.amount / 100D, request.payerAcct, request.payeeAcct);
                    PaymentResponse response =
                            new PaymentResponse(new RecordId(creatorId, id++), request.id, PaymentStatus.APPROVED);
                    sender().tell(response, self());
                }
        );

        builder.match(PaymentRequest.class, request -> {
						...
```

This matcher has an added qualifier testing that the amount is less than 1200, or $12.00 and it will immediately respond with a `APPROVED` status.

Lets also add a test to see this in action. In the test, add another test method.

```java
    @Test
    public void lowRiskPayment() {
        new TestKit(system) {{
            paymentActor.tell(new PaymentRequest(new RecordId(creator, id++), 1001, 2001, 1000), getRef());
            PaymentResponse response = expectMsgClass(PaymentResponse.class);
            assertEquals(PaymentStatus.APPROVED, response.status);
        }};
    }
```

Now run the test again. It should just pass. Also note the log messages.

### Creating a Child Actor

Remember, based on the Actor Model of Computation, an actor, upon receiving a message can do one or more of these three:

* Send messages
* Create other actors
* Change state/behavior to be used by next message

Now, lets explore the second property: Creating another child actor. Note that this actor becomes the parent of this child actor.

Now we create another actor, the `RiskAssessmentActor`. In this case we'll let it approve every request, just for simplicity. Here is the code:

```java
import akka.actor.AbstractActor;
import akka.event.Logging;
import akka.event.LoggingAdapter;
import akka.japi.pf.ReceiveBuilder;

public class RiskAssessmentActor extends AbstractActor {

    private final LoggingAdapter log = Logging.getLogger(getContext().system(), this);

    private final long creatorId = 300;

    private long id = 1;

    @Override
    public Receive createReceive() {
        ReceiveBuilder builder = ReceiveBuilder.create();

        builder.match(PaymentRequest.class, request -> {
                    log.info("Received payment assessment request of ${} accounts {} => {}",
                            request.amount / 100D, request.payerAcct, request.payeeAcct);
                    PaymentResponse response =
                            new PaymentResponse(new RecordId(creatorId, id++), request.id, PaymentStatus.APPROVED);
                    sender().tell(response, self());
                }
        );

        builder.matchAny(o ->
                log.error("Message of type {} received is not understood", o.getClass().getName())
        );

        return builder.build();
    }
}
```

By itself, there is nothing interesting about `RiskAssessmentActor`. It is similar to `PaymentActor` in many ways. But now we'll let `PaymentActor` forward the approval request to `RiskAssessmentActor` and then return it to the client, which is the test code.

Lets enhance the `PaymentActor` to do so. We'll modify the current high risk payment code to do the forward as follows:

```java
        builder.match(PaymentRequest.class, request -> {
                    log.info("Received payment request of ${} accounts {} => {}",
                            request.amount / 100D, request.payerAcct, request.payeeAcct);
                    PaymentResponse response =
                            new PaymentResponse(new RecordId(creatorId, id++), request.id, PaymentStatus.ACCEPTED);
                    sender().tell(response, self());

                    ActorRef riskActor = getContext().actorOf(Props.create(RiskAssessmentActor.class));
                    riskActor.forward(request, getContext());
                    // ^^^Add the two lines above^^^
                }
        );
```

Notice now the client should receive two messages. One with `ACCEPTED` status and one with `APPROVED` status. So lets modify our first test.

```java
    @Test
    public void pingPaymentActor() {
        new JavaTestKit(system) {{
            paymentActor.tell(new PaymentRequest(new RecordId(creator, id++), 1001, 2001, 30000), getRef());
            PaymentResponse response = expectMsgClass(PaymentResponse.class);
            assertEquals(PaymentStatus.ACCEPTED, response.status);
            PaymentResponse response2 = expectMsgClass(PaymentResponse.class);
            assertEquals(PaymentStatus.APPROVED, response2.status);
            // ^^^Add the two lines above^^^
        }};
     }

```