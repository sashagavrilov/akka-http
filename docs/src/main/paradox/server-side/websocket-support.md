# Server WebSocket Support

WebSocket is a protocol that provides a bi-directional channel between browser and webserver usually run over an
upgraded HTTP(S) connection. Data is exchanged in messages whereby a message can either be binary data or Unicode text.

Akka HTTP provides a stream-based implementation of the WebSocket protocol that hides the low-level details of the
underlying binary framing wire-protocol and provides a simple API to implement services using WebSocket.

## Model

The basic unit of data exchange in the WebSocket protocol is a message. A message can either be binary message,
i.e. a sequence of octets or a text message, i.e. a sequence of Unicode code points.

In the data model the two kinds of messages, binary and text messages, are represented by the two classes
@unidoc[BinaryMessage] and @unidoc[TextMessage] deriving from a common superclass
@scala[@scaladoc[Message](akka.http.scaladsl.model.ws.Message)]@java[@javadoc[Message](akka.http.javadsl.model.ws.Message)].
@scala[The subclasses @unidoc[BinaryMessage] and @unidoc[TextMessage] contain methods to access the data.]
@java[The superclass @javadoc[Message](akka.http.javadsl.model.ws.Message)]
contains `isText` and `isBinary` methods to distinguish a message and `asBinaryMessage` and `asTextMessage` methods to cast a message.]
Take the API of @unidoc[TextMessage] as an example (@unidoc[BinaryMessage] is very similar with `String` replaced by @unidoc[akka.util.ByteString]):

Scala
:  @@snip [Message.scala]($akka-http$/akka-http-core/src/main/scala/akka/http/scaladsl/model/ws/Message.scala) { #message-model }

Java
:  @@snip [Message.scala]($akka-http$/akka-http-core/src/main/scala/akka/http/javadsl/model/ws/Message.scala) { #message-model }

The data of a message is provided as a stream because WebSocket messages do not have a predefined size and could
(in theory) be infinitely long. However, only one message can be open per direction of the WebSocket connection,
so that many application level protocols will want to make use of the delineation into (small) messages to transport
single application-level data units like "one event" or "one chat message".

Many messages are small enough to be sent or received in one go. As an opportunity for optimization, the model provides
the notion of a "strict" message to represent cases where a whole message was received in one go.
@scala[Strict messages are represented with the `Strict` subclass for each kind of message which contains data as a strict, i.e. non-streamed, @unidoc[akka.util.ByteString] or `String`.]
@java[If `TextMessage.isStrict` returns true, the complete data is already available and can be accessed with `TextMessage.getStrictText` (analogously for @unidoc[BinaryMessage]).]

When receiving data from the network connection the WebSocket implementation tries to create a strict message whenever
possible, i.e. when the complete data was received in one chunk. However, the actual chunking of messages over a network
connection and through the various streaming abstraction layers is not deterministic from the perspective of the
application. Therefore, application code must be able to handle both streamed and strict messages and not expect
certain messages to be strict. (Particularly, note that tests against `localhost` will behave differently than tests
against remote peers where data is received over a physical network connection.)

For sending data, you can use @scala[`TextMessage.apply(text: String)`]@java[`TextMessage.create(String)`] to create a strict message if the
complete message has already been assembled. Otherwise, use @scala[`TextMessage.apply(textStream: Source[String, \_])`]@java[`TextMessage.create(Source<String, ?>)`]
to create a streaming message from an Akka Stream source.

## Server API

The entrypoint for the WebSocket API is the synthetic @unidoc[UpgradeToWebSocket] header which is added to a request
if Akka HTTP encounters a WebSocket upgrade request.

The WebSocket specification mandates that details of the WebSocket connection are negotiated by placing special-purpose
HTTP-headers into request and response of the HTTP upgrade. In Akka HTTP these HTTP-level details of the WebSocket
handshake are hidden from the application and don't need to be managed manually.

Instead, the synthetic @unidoc[UpgradeToWebSocket] represents a valid WebSocket upgrade request. An application can detect
a WebSocket upgrade request by looking for the @unidoc[UpgradeToWebSocket] header. It can choose to accept the upgrade and
start a WebSocket connection by responding to that request with an @unidoc[HttpResponse] generated by one of the
`UpgradeToWebSocket.handleMessagesWith` methods. In its most general form this method expects two arguments:
first, a handler @scala[@unidoc[Flow[Message, Message, Any]]]@java[@unidoc[Flow[Message, Message, ?]]] that will be used to handle WebSocket messages on this connection.
Second, the application can optionally choose one of the proposed application-level sub-protocols by inspecting the
values of @scala[`UpgradeToWebSocket.requestedProtocols`]@java[`UpgradeToWebSocket.getRequestedProtocols`] and pass the chosen protocol value to @scala[`handleMessages`]@java[`handleMessagesWith`].

### Handling Messages

A message handler is expected to be implemented as a @scala[@unidoc[Flow[Message, Message, Any]]]@java[@unidoc[Flow[Message, Message, ?]]]. For typical request-response
scenarios this fits very well and such a @unidoc[Flow] can be constructed from a simple function by using
@scala[`Flow[Message].map` or `Flow[Message].mapAsync`]@java[`Flow.<Message>create().map` or `Flow.<Message>create().mapAsync`].

There are other use-cases, e.g. in a server-push model, where a server message is sent spontaneously, or in a
true bi-directional scenario where input and output aren't logically connected. Providing the handler as a @unidoc[Flow] in
these cases may not fit. @scala[Another method named `UpgradeToWebSocket.handleMessagesWithSinkSource`]@java[An overload of `UpgradeToWebSocket.handleMessagesWith`] is provided, instead,
which allows to pass an output-generating @unidoc[Source[Message, \_]] and an input-receiving @unidoc[Sink[Message, \_]] independently.

Note that a handler is required to consume the data stream of each message to make place for new messages. Otherwise,
subsequent messages may be stuck and message traffic in this direction will stall.

### Example

Let's look at an @scala[@github[example](/docs/src/test/scala/docs/http/scaladsl/server/WebSocketExampleSpec.scala)]@java[@github[example](/docs/src/test/java/docs/http/javadsl/server/WebSocketCoreExample.java)].

WebSocket requests come in like any other requests. In the example, requests to `/greeter` are expected to be
WebSocket requests:

Scala
:  @@snip [WebSocketExampleSpec.scala]($test$/scala/docs/http/scaladsl/server/WebSocketExampleSpec.scala) { #websocket-request-handling }

Java
:  @@snip [WebSocketCoreExample.java]($test$/java/docs/http/javadsl/server/WebSocketCoreExample.java) { #websocket-handling }

@@@ div { .group-scala }
It uses pattern matching on the path and then inspects the request to query for the @unidoc[UpgradeToWebSocket] header. If
such a header is found, it is used to generate a response by passing a handler for WebSocket messages to the
`handleMessages` method. If no such header is found a `400 Bad Request` response is generated.
@@@

@@@ div { .group-java }
It uses a helper method `akka.http.javadsl.model.ws.WebSocket.handleWebSocketRequestWith` which can be used if
only WebSocket requests are expected. The method looks for the @unidoc[UpgradeToWebSocket] header and returns a response
that will install the passed WebSocket handler if the header is found. If the request is no WebSocket request it will
return a `400 Bad Request` error response.
@@@

In the example, the passed handler expects text messages where each message is expected to contain a (person's) name
and then responds with another text message that contains a greeting:

Scala
:  @@snip [WebSocketExampleSpec.scala]($test$/scala/docs/http/scaladsl/server/WebSocketExampleSpec.scala) { #websocket-handler }

Java
:  @@snip [WebSocketCoreExample.java]($test$/java/docs/http/javadsl/server/WebSocketCoreExample.java) { #websocket-handler }

@@@ note
Inactive WebSocket connections will be dropped according to the @ref[idle-timeout settings](../common/timeouts.md#idle-timeouts).
In case you need to keep inactive connections alive, you can either tweak your idle-timeout or inject
'keep-alive' messages regularly.
@@@

## Routing support

The routing DSL provides the @ref[handleWebSocketMessages](../routing-dsl/directives/websocket-directives/handleWebSocketMessages.md) directive to install a WebSocket handler if a request
is a WebSocket request. Otherwise, the directive rejects the request.

Let's look at how the above example can be rewritten using the high-level routing DSL.

Instead of writing the request handler manually, the routing behavior of the app is defined by a route that
uses the `handleWebSocketRequests` directive in place of the `WebSocket.handleWebSocketRequestWith`:

Scala
:  @@snip [WebSocketDirectivesExamplesSpec.scala]($test$/scala/docs/http/scaladsl/server/directives/WebSocketDirectivesExamplesSpec.scala) { #greeter-service }

Java
:  @@snip [WebSocketRoutingExample.java]($test$/java/docs/http/javadsl/server/WebSocketRoutingExample.java) { #websocket-route }

The handling code itself will be the same as with using the low-level API.

@@@ div { .group-scala }
The example also includes code demonstrating the testkit support for WebSocket services. It allows to create WebSocket
requests to run against a route using *WS* which can be used to provide a mock WebSocket probe that allows manual
testing of the WebSocket handler's behavior if the request was accepted.
@@@

See the @github[full routing example](/docs/src/test/java/docs/http/javadsl/server/WebSocketCoreExample.java).
