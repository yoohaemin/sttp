# fs2 backend

The [fs2](https://github.com/functional-streams-for-scala/fs2) backend is **asynchronous**. It can be created for any type implementing the `cats.effect.Async` typeclass, such as `cats.effect.IO`. Sending a request is a non-blocking, lazily-evaluated operation and results in a wrapped response. There's a transitive dependency on `cats-effect`. 

To use, add the following dependency to your project:

```scala
"com.softwaremill.sttp.client" %% "async-http-client-backend-fs2" % "2.0.0-RC9"
```
           
This backend depends on [async-http-client](https://github.com/AsyncHttpClient/async-http-client) and uses [Netty](http://netty.io) behind the scenes.

Next you'll need to add an implicit value:

```scala
import sttp.client.asynchttpclient.fs2.AsyncHttpClientFs2Backend

AsyncHttpClientFs2Backend().flatMap { implicit backend => ... }

// or, if you'd like to use custom configuration:
AsyncHttpClientFs2Backend.usingConfig(asyncHttpClientConfig).flatMap { implicit backend => ... }

// or, if you'd like to use adjust the configuration sttp creates:
AsyncHttpClientFs2Backend.usingConfigBuilder(adjustFunction, sttpOptions).flatMap { implicit backend => ... }

// or, if you'd like the backend to be wrapped in cats-effect Resource:
AsyncHttpClientFs2Backend.resource().use { implicit backend => ... }

// or, if you'd like to instantiate the AsyncHttpClient yourself:
implicit val sttpBackend = AsyncHttpClientFs2Backend.usingClient(asyncHttpClient)
```

## Streaming

The fs2 backend supports streaming for any instance of the `cats.effect.Effect` typeclass, such as `cats.effect.IO`. If `IO` is used then the type of supported streams is `fs2.Stream[IO, ByteBuffer]`.

Requests can be sent with a streaming body like this:

```scala
import sttp.client._
import sttp.client.asynchttpclient.fs2.AsyncHttpClientFs2Backend

import java.nio.ByteBuffer
import cats.effect.{ContextShift, IO}
import fs2.Stream

implicit val cs: ContextShift[IO] = IO.contextShift(ExecutionContext.Implicits.global)
val effect = AsyncHttpClientFs2Backend[IO]().flatMap { implicit backend =>
  val stream: Stream[IO, ByteBuffer] = ...

  basicRequest
    .streamBody(stream)
    .post(uri"...")
}
// run the effect
```

Responses can also be streamed:

```scala
import sttp.client._
import sttp.client.asynchttpclient.fs2.AsyncHttpClientFs2Backend

import java.nio.ByteBuffer
import cats.effect.{ContextShift, IO}
import fs2.Stream
import scala.concurrent.duration.Duration

implicit val cs: ContextShift[IO] = IO.contextShift(ExecutionContext.Implicits.global)
val effect = AsyncHttpClientFs2Backend[IO]().flatMap { implicit backend =>
  val response: IO[Response[Either[String, Stream[IO, ByteBuffer]]]] =
    basicRequest
      .post(uri"...")
      .response(asStream[Stream[IO, ByteBuffer]])
      .readTimeout(Duration.Inf)
      .send()

  response
}
// run the effect
```

## Websockets

The fs2 backend supports:

* high-level, "functional" websocket interface, through the `sttp.client.asynchttpclient.fs2.Fs2WebSocketHandler`
* low-level interface by wrapping a low-level Java interface, `sttp.client.asynchttpclient.WebSocketHandler`
* streaming - see below

See [websockets](../websockets.html) for details on how to use the high-level and low-level interfaces.

## Streaming websockets 

There are additionally high-level helpers collected in `sttp.client.asynchttpclient.fs2.Fs2Websockets` which provide means to run the whole websocket communication through an `fs2.Pipe`. Example for a simple echo client:

```scala
import cats.effect.IO
import cats.implicits._
import sttp.client._
import sttp.client.ws._
import sttp.model.ws.WebSocketFrame

basicRequest
  .get(uri"wss://echo.websocket.org")
  .openWebsocketF(Fs2WebSocketHandler())
  .flatMap { response =>
    Fs2WebSockets.handleSocketThroughTextPipe(response.result) { in =>
      val receive = in.evalMap(m => IO(println("Received")))
      val send = Stream("Message 1".asRight, "Message 2".asRight, WebSocketFrame.close.asLeft)
      send merge receive.drain
    }
  }
```