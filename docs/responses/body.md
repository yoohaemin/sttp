# Response body specification

By default, the received response body will be read as a `Either[String, String]`, using the encoding specified in the `Content-Type` response header (and if none is specified, using `UTF-8`). This is of course configurable: response bodies can be ignored, deserialized into custom types, received as a stream or saved to a file.

The default `response.body` will be a:

* `Left(errorMessage)` if the request is successful, but response code is not expected (non 2xx).
* `Right(body)` if the request is successful and the response code is expected (2xx).

How the response body will be read is part of the request definition, as already when sending the request, the backend needs to know what to do with the response. The type to which the response body should be deserialized is the second type parameter of `RequestT`, and stored in the request definition as the `request.response: ResponseAs[T, S]` property.

## Basic response specifications

To conveniently specify how to deserialize the response body, a number of `asXxx` methods are available. They can be used to provide a value for the request definition's `response` modifier:

```scala
basicRequest.response(asByteArray)
```

When the above request is completed and sent, it will result in a `Response[Either[String, Array[Byte]]]`. Other possible response specifications are:

```
def ignore: ResponseAs[Unit, Nothing]
def asString: ResponseAs[Either[String, String], Nothing]
def asStringAlways: ResponseAs[String, Nothing]
def asString(encoding: String): ResponseAs[Either[String, String], Nothing]
def asStringAlways(encoding: String): ResponseAs[String, Nothing]
def asByteArray: ResponseAs[Either[String, Array[Byte]], Nothing]
def asByteArrayAlways: ResponseAs[Array[Byte], Nothing]
def asParams: ResponseAs[Either[String, Seq[(String, String)]], Nothing]
def asParams(encoding: String): ResponseAs[Either[String, Seq[(String, String)]], Nothing]
def asFile(file: File): ResponseAs[Either[String, File], Nothing]
def asFileAlways(file: File): ResponseAs[File, Nothing]
def asPath(path: Path): ResponseAs[Either[String, Path], Nothing]
def asPathAlways(path: Path): ResponseAs[Path], Nothing]

def asEither[L, R, S](onError: ResponseAs[L, S], 
                      onSuccess: ResponseAs[R, S]): ResponseAs[Either[L, R], S]
def fromMetadata[T, S](f: ResponseMetadata => ResponseAs[T, S]): ResponseAs[T, S]
```

Hence, to discard the response body, simply specify:

```
basicRequest.response(ignore)
```   

And to save the response to a file:

```
basicRequest.response(asFile(someFile))
```

```note:: As the handling of response is specified upfront, there's no need to "consume" the response body. It can be safely discarded if not needed.
```

## Custom body deserializers

It's possible to define custom body deserializers by taking any of the built-in response specifications and mapping over them. Each `ResponseAs` instance has `map` and `mapWithMetadata` methods, which can be used to transform it to a specification for another type (optionally using response metadata, such as headers or the status code). Each such value is immutable and can be used multiple times.

As an example, to read the response body as an int, the following response specification can be defined (warning: this ignores the possibility of exceptions!):

```scala
val asInt: ResponseAs[Either[String, Int], Nothing] = asString.map(_.toInt)

basicRequest
  .response(asInt)
  ...
```

To integrate with a third-party JSON library, and always parse the response as a json (regardless of the status code):

```scala
def parseJson(json: String): Either[JsonError, JsonAST] = ...
val asJson: ResponseAs[Either[JsonError, JsonAST], Nothing] = asStringAlways.map(parseJson)

basicRequest
  .response(asJson)
  ...
```           

For some mapped response specifications available out-of-the-box, see [json support](../json.html).

Using the `fromMetadata` combinator, it's possible to dynamically specify how the response should be deserialized, basing on the response status code and response headers. The default `asString`, `asByteArray` response descriptions use this method to return a `Left` in case of non-2xx responses, and a `Right` otherwise.

It's also possible to map over an existing response specification, by using the `request.mapResponse(...)` and `request.mapResponseRight(...)` methods (which is available, if the response body should be an either). That's equivalent to calling `request.response(request.response.map(...))`, that is setting a new response specification, to a modified old response specification; but with shorter syntax.

## Streaming

If the backend used supports streaming (see [backends summary](../backends/summary.html)), it's possible to receive responses as a stream. This can be specified using the following method:

```scala
def asStream[S]: ResponseAs[Either[String, S], S] = ResponseAsStream[S, S]()
def asStreamAlways[S]: ResponseAs[S, S] = ResponseAsStream[S, S]()
```

For example, when using the [Akk backend](../backends/akka.html):

```scala
import sttp.client._
import sttp.client.akkahttp._

import akka.stream.scaladsl.Source
import akka.util.ByteString

implicit val sttpBackend = AkkaHttpBackend() 

val response: Future[Response[Source[Either[String, ByteString], Any]]] =
  basicRequest
    .post(uri"...")
    .response(asStream[Source[ByteString, Any]])
    .send()    
```

```note:: Unlike with non-streaming response handlers, each streaming response should be entirely consumed by client code.
```