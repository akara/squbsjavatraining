## Day 4: Serving Http Requests

### Creating a Service with the High Level API

#### Simple Hello Service

Create a new Java class in your `*svc` project named `MockService`. We'll re-use it later. And lets start with something really simple. Just a `hello` service that says `hi` to you:

```java
import akka.http.javadsl.server.Route;
import org.squbs.unicomplex.AbstractRouteDefinition;

public class MockService extends AbstractRouteDefinition {

    @Override
    public Route route() {
        return route(
                path("hello", () ->
                        complete("hi")
                )
        );
    }
}
```

#### Service Registration

Next, register the service to `squbs`. Edit `*svc/src/main/resource/META-INF/squbs-meta.conf`.

```
cube-name = {my-package.cubename}

cube-version = "0.0.1-SNAPSHOT"
squbs-services = [
  {
    class-name = {my-package}.MockService
    listeners = [default-listener, paypal-listener]
    web-context = mock
  }
]
```

Make sure to replace all `{...}` with real values. The `cube-name` is likely generated for you already. No need to change. However, you need to ensure the `class-name` is accurate. To check in IntelliJ press the `Command` button on your Mac or the `Ctrl` button on your Windows system and mouse-over the class name. That name should become a link and IntelliJ should describe that class. If no popup occurs, the class name is wrong.

#### Run the service

1. From `sbt` window,
   * Type `reload` to ensure sbt is up-to-date with the dependencies
   * Run `extractConfig dev` to populate config
2. Configure `App` in IntelliJ - add AspectJ config
   * Click down button and choose “Edit Configurations"
   * Click the `+` sign
   * Choose `application`
   * Name: `App`
   * Main class: `org.squbs.unicomplex.Bootstrap`
   * VM options: `-javaagent:/Users/{my_login}/.ivy2/cache/org.aspectj/aspectjweaver/jars/aspectjweaver-1.8.13.jar`. Don't forget to replace `{my_login}` with your real user name. An equivalent path is needed for Windows.
   * Working directory: `…/{project}servsvc`
   * Use classpath of module: `{project}servsvc`
   * Press `OK`
3. Run the app by pressing the start button with the right arrow
4. Check the app and registered context
   * Point your browser to `http://localhost:8080/admin/v3console/ValidateInternals`
   * Choose the `Component Status` tab
   * Select the link `org.squbs.unicomplex…::Listeners`
   * See the app registered to Default Listener
5. Point your browser to `http://localhost:8080/mock/hello`

This way of setting up and running the service lends itself very well to debugging. There are other ways to run the service directly fromt he `sbt shell` as follows:

1. Assume the application is running from IntelliJ, stop it.
2. Open the `sbt shell` window in your IDE.
3. Type:
   
   `project {project}svc`
   `~re-start`
   
4. Wait until project is up, only a few seconds, then test it out the same way.  The `~` in front of `re-start` causes sbt to watch the project directory and re-start the server anytime a file changes.

5. After done testing, enter `re-stop` to stop the server.

#### Adding Meaning to MockService

So far we have been just serving a `hello` request. Now lets make the service serve some useful data. First, lets create an account class as follows:

```java
public class Account {
    public final long id;
    public final String name;
    public final String email;
    public final long lastBalance;
}
```

Then, with cursor in the `Account` class, use the tools at `Code` -> `Generate...` to generate constructor, `equals` and `hashcode`, and `toString` for the `Account` class. Select all fields when prompted.

Next lets make the `MockService` return an account. Add this following `path` directive below the `path(hello, ...)`

```java
    path(segment("account").slash(longSegment()), id ->
        complete(StatusCodes.OK, new Account(id, "Foo Bar", "foobar@foobar.com",
                 100), Jackson.marshaller())
    )
```

To ensure we do not have any missing symbols, add the following imports:

```java
import akka.http.javadsl.marshallers.jackson.Jackson;
import akka.http.javadsl.model.StatusCodes;

import static akka.http.javadsl.server.PathMatchers.*;
```

Restart your service and try this URL: `http://localhost:8080/mock/account/100`

So far we have learned how to use the `PathMatcher` and the `Marshaller` to send a marshalled object back.

The route DSL is very versatile and allows all kinds of matching and extractions. We just touched the surface of it using `path`, `PathMatcher`, and `complete`. For a full list of directives, please visit [Directives](https://doc.akka.io/docs/akka-http/current/routing-dsl/directives/index.html?language=java) in the Akka-HTTP documentation.

### Creating a Service with the Low Level API

So far we have used the high-level route API to serve requests through `MockService`. This high-level route API is very versatile and useful for complicated REST API matchings.

There is another low-level API that is based on Akka Streams `Flow` concept and component. It may be a little more cumbersome to use for complex REST requests. But in exchange, it delivers higher performance, lower latency, consumes less memory, and allows us to provide end-to-end back-pressure. It is extremely suitable for high-volume, highly resilient services.

The `Flow` API is defined as `Flow<HttpRequest, HttpResponse, NotUsed>` telling us to build a flow component that processes an `HttpRequest` into an `HttpResponse`. Lets try create a `ValidationService` with the low-level API. Again, we start simple. First create a Java class `ValidationService` that just provides a simple `"Hello"` flow:

```java
import akka.NotUsed;
import akka.http.javadsl.model.HttpRequest;
import akka.http.javadsl.model.HttpResponse;
import akka.http.javadsl.model.StatusCodes;
import akka.stream.javadsl.Flow;
import org.squbs.unicomplex.AbstractFlowDefinition;

public class ValidationService extends AbstractFlowDefinition {


    @Override
    public Flow<HttpRequest, HttpResponse, NotUsed> flow() {
        return Flow.of(HttpRequest.class)
                .map(req -> HttpResponse.create().withStatus(StatusCodes.OK).withEntity("Hello"));
    }
}
```

#### Register the Service

Similar to registering `MockService`, we need to register `ValidationService` in `*svc/src/main/resources/META-INF/` in order to have it being recognized and installed 

```yaml
cube-name = {my-package.cubename}

cube-version = "0.0.1-SNAPSHOT"
squbs-services = [
  {
    class-name = {my-package}.MockService
    listeners = [default-listener, paypal-listener]
    web-context = mock
  }
  {
    class-name = {my-package}.ValidationService
    listeners = [default-listener, paypal-listener]
    web-context = validate
  }
  # ^^^^^^^^^^^^^^^^^^^^^^^^^
  # Add this block as another service with another context, "validate"
]
```

Remember to check the classes referenced here are actually valid.

Restart the server and try the URL: `http://localhost:8080/validate`

### Calling Another Service

Similar to the `Flow`-based low-level service, we use streams to make HttpClient calls. `squbs` provides a beefed up HttpClient with all required enterprise functionality through the `ClientFlow` API. An example of the `ClientFlow` API can be seen in the followings:

```java
import akka.actor.ActorSystem;
import akka.http.javadsl.HostConnectionPool;
import akka.http.javadsl.model.HttpRequest;
import akka.http.javadsl.model.HttpResponse;
import akka.japi.Pair;
import akka.stream.ActorMaterializer;
import akka.stream.javadsl.Flow;
import akka.stream.javadsl.Sink;
import akka.stream.javadsl.Source;
import org.squbs.httpclient.ClientFlow;
import scala.util.Try;

import java.util.concurrent.CompletionStage;

public class ClientSample {

    final ActorSystem system = ActorSystem.create();
    final ActorMaterializer mat = ActorMaterializer.create(system);

    final Flow<Pair<HttpRequest, Integer>, Pair<Try<HttpResponse>, Integer>, HostConnectionPool>
            clientFlow = ClientFlow.create("sample", system, mat);

    public CompletionStage<Pair<Try<HttpResponse>, Integer>> clientCall(HttpRequest request) {

        return Source
                .single(Pair.create(request, 42))
                .via(clientFlow)
                .runWith(Sink.head(), mat);
    }
}
```

Lets take a closer look at each part of the API. First we need an `ActorSystem` to access the `ClientFlow`. Like all `Akka Streams`-based facilities, we also need a materializer. We can create new ones as seen in this example, or we can use an existing one. The actual `ClientFlow` instance is created by calling `ClientFlow.create(String name, ActorSystem system, Materializer mat)`. The name refers to a service name that gets resolved to a real service endpoint behind the scenes.

As we now have the `ClientFlow` we can go into its usage. Lets start with the HttpClient architecture. The HttpClient holds a pool of persistent connnections, for which the size can be specified in the configuration. Some http calls may be slower than the other. In order to maximize the stream throughput and response time of all `ClientFlow` calls, the `ClientFlow` does not guarantee the order of  messages. Some http calls may be faster and respond before preceding calls.

For this reason, the `ClientFlow` takes a context of arbitrary type (int 42 in this case) in addition to the `HttpRequest` itself. It allows users to identify which request a response corresponds to. While `ClientFlow` does not require this context to be unique, it is in the best interest of the developer to keep it unique.

In the example above, the context is just an `Integer` of value `42`. In the simplest case, we create a source of a single `Pair` of `HttpRequest` and `Integer` and pass it to the `ClientFlow`. Then we obtain the `Sink.head()` which is the first and only value coming out. That would be the materialized value for this stream, which gives us a `CompletionStage<Pair<Try<HttpResponse>, Integer>>` handing us the response asynchronously.

Well, this is just the simplest use case. When we build an end-to-end application, we may use the stream in a very different manner. Lets try it out.

## E2E flow + Testing

### Building an End-To-End Flow

In this section, we will modify our `ValidationService` to do something more useful and make a client REST call to `MockService`, asking for accounts given an account number range. Lets get started:

Design:

* We send an HttpRequest to do mass account fetches based on a begin id and end id as follows: `http://localhost:8080/validate/{startAcct}/{endAcct}`
* In the server, we extract the `startAcct` and `endAcct` and expand it to each account id in the stream.
* The stream then creates an `HttpRequest` from each account id and sends it along with the account id as a context to the `ClientFlow` which calls the `MockService`. The response is then aggregated into a JSON and sent back to the sender.

Lets get our hands dirty and build this service, in full:

```java
package com.paypal.squbs.jtraining.svc;

import akka.NotUsed;
import akka.http.javadsl.HostConnectionPool;
import akka.http.javadsl.model.*;
import akka.japi.Pair;
import akka.stream.*;
import akka.stream.javadsl.*;
import org.squbs.httpclient.ClientFlow;
import org.squbs.unicomplex.AbstractFlowDefinition;
import scala.util.Try;

import java.util.ArrayList;
import java.util.Collections;
import java.util.Iterator;
import java.util.Objects;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

public class ValidationService extends AbstractFlowDefinition {

    static final AtomicLong currentReqId = new AtomicLong(0L);

    static class MessageContext {

        final HttpRequest origRequest;
        final int rangeSize;
        final long accountId;
        final long reqId;

        MessageContext(HttpRequest origRequest, int rangeSize, long accountId, long reqId) {
            this.origRequest = origRequest;
            this.rangeSize = rangeSize;
            this.accountId = accountId;
            this.reqId = reqId;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            MessageContext that = (MessageContext) o;
            return rangeSize == that.rangeSize &&
                    accountId == that.accountId &&
                    reqId == that.reqId &&
                    Objects.equals(origRequest, that.origRequest);
        }

        @Override
        public int hashCode() {
            return Objects.hash(origRequest, rangeSize, accountId, reqId);
        }
    }

    final Materializer mat = ActorMaterializer.create(context());

    final Flow<HttpRequest, MessageContext, NotUsed> expandIdsFlow =
            Flow.of(HttpRequest.class).flatMapConcat(req -> {
                Iterator<String> segments = req.getUri().pathSegments().iterator();
                segments.next(); // This is the context "validate"
                int startId = Integer.parseInt(segments.next());
                int endId = Integer.parseInt(segments.next());
                int rangeSize = endId - startId + 1;
                long reqId = currentReqId.getAndIncrement();
                return Source
                        .range(startId, endId)
                        .map(id -> new MessageContext(req, rangeSize, id, reqId));
            });

    final Flow<MessageContext, Pair<HttpRequest, MessageContext>, NotUsed> buildRequestFlow =
            Flow.<MessageContext>create().map(ctx -> Pair.create(HttpRequest.GET("/mock/account/" + ctx.accountId), ctx));


    final Flow<Pair<HttpRequest, MessageContext>, Pair<Try<HttpResponse>, MessageContext>, HostConnectionPool> clientFlow =
            ClientFlow.create("mock", context().system(), mat);


    final Flow<Pair<Try<HttpResponse>, MessageContext>, Pair<String, MessageContext>, NotUsed> handleResponseFlow =
            Flow.<Pair<Try<HttpResponse>, MessageContext>>create()
                    .map(pair -> Pair.create(pair.first().get(), pair.second()))
                    .mapAsync(5, pair -> pair.first().entity().toStrict(5000, mat)
                            .thenApply(strict -> Pair.create(strict.getData().utf8String(), pair.second())));

    final Flow<Pair<Try<HttpResponse>, MessageContext>, Pair<String, MessageContext>, NotUsed> handleErrorStatusFlow =
            Flow.<Pair<Try<HttpResponse>, MessageContext>>create().map(pair -> {
                HttpResponse response = pair.first().get();
                StatusCode status = response.status();
                long acctId = pair.second().accountId;
                String s =  "{ \"account\" : " + acctId + ", \"status\" : \"" + status + "\"}";
                response.discardEntityBytes(mat);
                return Pair.create(s, pair.second());
            });

    final Flow<Pair<Try<HttpResponse>, MessageContext>, Pair<String, MessageContext>, NotUsed> handleExceptionFlow =
            Flow.<Pair<Try<HttpResponse>, MessageContext>>create().map(pair -> {
                Throwable t = pair.first().failed().get();
                long acctId = pair.second().accountId;
                String s = "{ \"account\" : " + acctId + ", \"status\" : \"" + t + "\"}";
                return Pair.create(s, pair.second());
            });


    final Flow<Pair<String, MessageContext>,  HttpResponse, NotUsed> buildResponseFlow =
            Flow.<Pair<String, MessageContext>>create()
                    .scan(Collections.<Pair<String, MessageContext>>emptyList(), (list, pair) -> {
                        if (!list.isEmpty() && list.get(0).second().reqId != pair.second().reqId) {
                            return Collections.singletonList(pair);
                        } else {
                            // Note: Collections used in `scan` MUST BE IMMUTABLE
                            // You can use other collections library that deals with immutability more effectively.
                            ArrayList<Pair<String, MessageContext>> newList = new ArrayList<>(list.size() + 1);
                            newList.addAll(list);
                            newList.add(pair);
                            return Collections.unmodifiableList(newList);
                        }
                    })
                    .filter(list -> list.size() > 0 && list.size() >= list.get(0).second().rangeSize)
                    .map(list -> {
                        String content = list.stream().map(Pair::first).collect(Collectors.joining(",", "[", "]"));
                        return HttpResponse.create()
                                .withStatus(StatusCodes.OK)
                                .withEntity(ContentTypes.APPLICATION_JSON, content);
                    });


    @Override
    public Flow<HttpRequest, HttpResponse, NotUsed> flow() {

        return Flow.fromGraph(GraphDSL.create(b -> {
            final UniformFanOutShape<Pair<Try<HttpResponse>, MessageContext>, Pair<Try<HttpResponse>, MessageContext>> partition =
                    b.add(Partition.create(3, pair -> {
                        if (pair.first().isFailure()) {
                            return 2;
                        } else if (StatusCodes.OK.equals(pair.first().get().status())) {
                            return 0;
                        } else {
                            return 1;
                        }
                    }));
            final UniformFanInShape<Pair<String, MessageContext>, Pair<String, MessageContext>> merge = b.add(Merge.create(3));
            final FlowShape<HttpRequest, MessageContext> expandIds = b.add(expandIdsFlow);
            final FlowShape<MessageContext, Pair<HttpRequest, MessageContext>> buildRequest = b.add(buildRequestFlow);
            final FlowShape<Pair<HttpRequest, MessageContext>, Pair<Try<HttpResponse>, MessageContext>> client = b.add(clientFlow);
            final FlowShape<Pair<Try<HttpResponse>, MessageContext>, Pair<String, MessageContext>> handleResponse = b.add(handleResponseFlow);
            final FlowShape<Pair<Try<HttpResponse>, MessageContext>, Pair<String, MessageContext>> handleErrorStatus = b.add(handleErrorStatusFlow);
            final FlowShape<Pair<Try<HttpResponse>, MessageContext>, Pair<String, MessageContext>> handleException = b.add(handleExceptionFlow);
            final FlowShape<Pair<String, MessageContext>,  HttpResponse> buildResponse = b.add(buildResponseFlow);

            b.from(expandIds).via(buildRequest).via(client).viaFanOut(partition);
            b.from(partition.out(0)).via(handleResponse).viaFanIn(merge).via(buildResponse);
            b.from(partition.out(1)).via(handleErrorStatus).viaFanIn(merge);
            b.from(partition.out(2)).via(handleException).viaFanIn(merge);

            return FlowShape.of(expandIds.in(), buildResponse.out());
        }));
    }
}
```

Remember, we referred to a service called `"mock"`. Before we can get this running, we also have to edit our `application.properties` file. This is in `*svc/src/main/resources/META-INF/configuration/Dev/env.conf`. Add the resource as follows:

```
topo-connections {
  mock_host = localhost
  mock_port = 8080
  mock_protocol = http
}
```

After changing the configurations, make sure to re-run `extractConfig Dev` from your `sbt shell` window.

### Testing Http(s) Services

Since we're using JUnit in this our session, we also need to make sure `JUnit` is in your dependencies. Edit the service project's `build.sbt` and add the necessary `JUnit` dependencies as follows:

```scala
  "junit" % "junit" % "4.12" % "test",
  "com.novocode" % "junit-interface" % "0.11" % "test->default"
``` 

squbs provides a powerful `CustomTestKit` API used for creating tests. Just extend the `CustomTestKit` API. You can pass configuration to this API through calling the superclass constructor. A picture is worth a thousand words, so lets get straight down to code.

* In your service project under `src/test/java` create the same package as your mock service. If the `java` directory does not exist, you may want to create that, too.
* Under this package, create a class called `MockServiceTest`.

The code for `MockServiceTest` is as follows:

```java
import akka.http.javadsl.Http;
import akka.http.javadsl.model.HttpRequest;
import akka.http.javadsl.model.HttpResponse;
import akka.stream.ActorMaterializer;
import akka.stream.Materializer;
import org.junit.Test;
import org.squbs.testkit.japi.CustomTestKit;

import java.util.concurrent.CompletionStage;
import java.util.concurrent.ExecutionException;

import static org.junit.Assert.assertEquals;

public class MockServiceTest extends CustomTestKit {

    final Materializer mat;

    public MockServiceTest() {
        // Add more configuration parameters to the super() call as needed.
        super(true);
        mat = ActorMaterializer.create(system());
    }

    @Test
    public void callMockActor() throws ExecutionException, InterruptedException {
        CompletionStage<HttpResponse> responseF = Http.get(system()).singleRequest(
                HttpRequest.create("http://localhost:" + port() + "/mock/hello"));

        HttpResponse response = responseF.toCompletableFuture().get();

        CompletionStage<String> contentF = response.entity().toStrict(1000, mat)
                .thenApply(e -> e.getData().utf8String());
        String content = contentF.toCompletableFuture().get();
        assertEquals("hi", content);
    }
}
```

The test configuration is not the same as the Dev configuration. You can pass the configuration in through the CustomTestKit constructor. Or in this case, we'll add the test configuration to `/src/test/resources`. Lets create `/src/test/resources/application.conf` with the following content:

```
default-listener  {
  aliases = [dev-listener]
}

topo-connections {
  mock_host = localhost
  mock_port = 8080
  mock_protocol = http
}
```

Now the test should run without an issue. Just run the JUnit test as usual.
