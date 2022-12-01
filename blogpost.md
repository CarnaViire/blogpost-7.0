With the recent [release of .NET 7](https://devblogs.microsoft.com/dotnet/announcing-dotnet-7/), we'd like to introduce some interesting changes and additions done in the networking space. This blog post talks about .NET 7 changes in [HTTP space](#http), [new QUIC APIs](#quic), [networking security](#security) and [WebSockets](#websockets).

# HTTP

## Improved handling of connection attempt failures

In versions prior to .NET 6, in case there is no connection immediately available in the connection pool, a new HTTP request will always spin up a new connection attempt and wait for it. The downside to this is, if it takes a while to establish that connection and another connection becomes available in the meantime, that request would continue waiting for the connection it spawned, hurting latency. In .NET 6.0 [we changed this](https://devblogs.microsoft.com/dotnet/dotnet-6-networking-improvements/#other-http-changes) to process requests on whichever connection that becomes available first, whether that’s a newly established one or one that became ready to handle the request in the meantime.

Unfortunately, the .NET 6.0 implementation turned out to be problematic for some users: a failing connection attempt also fails the request on the top of the request queue, which  may lead to unexpected request failures in [certain scenarios](https://github.com/dotnet/runtime/issues/60654#issue-1030717005). Moreover, if there is a pending connection in the pool that will never become usable eg. because of a misbehaving server or a network issue, new incoming requests associated with it will also stall and may time out massively.

In .NET 7.0 we implemented the following changes to address these issues:

- A failing connection attempt can only fail it's initiating request, and never an unrelated one. If the original request has been handled by the time a connection fails, the connection failure is ignored ([dotnet/runtime#62935](https://github.com/dotnet/runtime/pull/62935)).
- If a request initiates a new connection, but then becomes handled by another connection from the pool, the new pending connection attempt will time out silently after a short period, regardless of `ConnectTimeout`. With this change, stalled connections will not stall unrelated requests ([dotnet/runtime#71785](https://github.com/dotnet/runtime/pull/71785)). Note that the failures of such "disowned" pending connection attempts will happen in the background and never surface to the user, the only way to observe them is to enable telemetry.

## HttpHeaders Read Thread Safety

The `HttpHeaders` collections were never thread-safe. Accessing a header may force lazy parsing of its value, resulting in modifications to the underlying data structures.

Before .NET 6, reading from the collection concurrently happened to be thread-safe in *most* cases.

Starting with .NET 6, less locking was performed around header parsing as it was no longer needed internally.
Due to this change, multiple examples of users accessing the headers concurrently surfaced, for example, in [gRPC](https://github.com/dotnet/runtime/issues/55898), [NewRelic](https://github.com/newrelic/newrelic-dotnet-agent/issues/803), or even [HttpClient itself](https://github.com/dotnet/runtime/issues/65379).
Violating thread safety in .NET 6 may result in the header values being duplicated/malformed or various exceptions being thrown during enumeration/header accesses.

.NET 7 makes the header behavior more intuitive. The `HttpHeaders` collection now matches the thread-safety guarantees of a `Dictionary`:
> The collection can support multiple readers concurrently, as long as it is not modified. In the rare case where an enumeration contends with write accesses, the collection must be locked during the entire enumeration. To allow the collection to be accessed by multiple threads for reading and writing, you must implement your own synchronization.

This was achieved by the following changes:
- A "validating read" of an invalid value does not result in the removal of the invalid value: [dotnet/runtime#67833](https://github.com/dotnet/runtime/pull/67833) (thanks [@heathbm](https://github.com/heathbm)).
- Concurrent reads are thread-safe: [dotnet/runtime#68115](https://github.com/dotnet/runtime/pull/68115).

## Detect HTTP/2 and HTTP/3 Protocol Errors

The HTTP/2 and HTTP/3 protocols define protocol-level error codes in [RFC 7540 section 7](https://www.rfc-editor.org/rfc/rfc7540#section-7) and [RFC 9114 section 8.1](https://www.rfc-editor.org/rfc/rfc9114.html#section-8.1), for example `REFUSED_STREAM (0x7)` in HTTP/2 or `H3_EXCESSIVE_LOAD (0x0107)` in HTTP/3. Unlike HTTP-status codes, this is low-level error information that is unimportant for most `HttpClient` users, but it helps in advanced HTTP/2 or HTTP/3 scenarios, notably grpc-dotnet, where distinguishing protocol errors is vital to implement [client retries](https://learn.microsoft.com/en-us/aspnet/core/grpc/retries?view=aspnetcore-7.0).

We defined a new exception [`HttpProtocolException`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpprotocolexception) to hold the protocol-level error code in it's `ErrorCode` property.

When calling `HttpClient` directly, `HttpRequestException` can be an inner exception of `HttpRequestException`:

```csharp
try
{
    using var response = await httpClient.GetStringAsync(url);
}
catch (HttpRequestException ex) when (ex.InnerException is HttpProtocolException pex)
{
    Console.WriteLine("HTTP error code: " + pex.ErrorCode)
}
```

When working with `HttpContent`'s response stream, it is being thrown directly:

```csharp
using var response = await httpClient.GetAsync(url, HttpCompletionOption.ResponseHeadersRead);
using var responseStream = await response.Content.ReadAsStreamAsync();
try
{
    await responseStream.ReadAsync(buffer);
}
catch (HttpProtocolException pex)
{
    Console.WriteLine("HTTP error code: " + pex.ErrorCode)
}
```

## HTTP/3

HTTP/3 support in `HttpClient` was already feature complete in the previous .NET release, so we mostly concentrated our efforts in this space on the underlying `System.Net.Quic`. Despite that, we did introduce few fixes and changes in .NET 7.

The most important change is that HTTP/3 is now enabled by default ([PR #73153](https://github.com/dotnet/runtime/pull/73153)). It doesn't mean that all HTTP requests will prefer HTTP/3 from now on, but in certain cases they might upgrade to it. For it to happen, the request must opt into version upgrade via [`HttpRequestMessage.VersionPolicy`](https://learn.microsoft.com/cs-cz/dotnet/api/system.net.http.httprequestmessage.versionpolicy?view=net-7.0) set to `RequestVersionOrHigher`. Then, if the server announces HTTP/3 authority in `Alt-Svc` header, `HttpClient` will use it for further requests, see [RFC 9114 - 3.1.1. HTTP Alternative Services](https://www.rfc-editor.org/rfc/rfc9114.html#name-http-alternative-services).

To name a few other interesting changes:
- HTTP telemetry was extended to cover HTTP/3 - [Issue #40896](https://github.com/dotnet/runtime/issues/40896).
- Exception details were improved in case QUIC connection cannot be established - [Issue #70949](https://github.com/dotnet/runtime/issues/70949)
- Proper usage of Host header for Server Name Identification (SNI) was fixed - [Issue #57169](https://github.com/dotnet/runtime/issues/57169)



# QUIC

QUIC is a new, transport layer protocol. It has been recently standardized in [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html). It uses UDP as an underlying protocol and it's inherently secure as it mandates TLS 1.3 usage, see [RFC 9001](https://www.rfc-editor.org/rfc/rfc9001.html). Another interesting difference from well-known transport protocols such as TCP and UDP is that it has stream multiplexing built-in on the transport layer. This allows having multiple, concurrent, independent data streams that do not affect each other.

QUIC itself doesn't define any semantics for the exchanged data as it's a transport protocol. It's rather used in application layer protocols, for example in [HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html) or in [SMB over QUIC](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-over-quic). It can also be used for any custom defined protocol.

The protocol offers many advantages over TCP with TLS. For instance, faster connection establishment as it doesn't require as many round trips as TCP with TLS on top. Or avoidance of head-of-line blocking problem where one lost packet doesn't block data of all the other streams. On the other hand, there are disadvantages that come with using QUIC. As it is a new protocol, its adoption is still growing and is limited. Apart from that, QUIC traffic might be even blocked by some networking components.

## QUIC in .NET

We introduced QUIC implementation in .NET 5 in `System.Net.Quic` library. However, up until now the library was strictly internal and served only for own implementation of HTTP/3. With the release of .NET 7, we're making the library public and we're exposing its APIs. Since we had only `HttpClient` and Kestrel as consumers of the APIs for this release, we decided to keep them as [preview feature](https://github.com/dotnet/designs/blob/main/accepted/2021/preview-features/preview-features.md). It gives us ability to tweak the API in the next release before we settle on the final shape.

From the implementation perspective, `System.Net.Quic` depends on [MsQuic](https://github.com/microsoft/msquic), native implementation of QUIC protocol. As a result, `System.Net.Quic` platform support and dependencies are inherited from MsQuic and documented in [HTTP/3 Platform dependencies](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-http3#platform-dependencies). In short, MsQuic library is shipped as part of .NET for Windows. But for Linux, `libmsquic` must be manually installed via an appropriate package manager. For the other platforms, it is still possible to build MsQuic manually, whether against SChannel or OpenSSL, and use it with `System.Net.Quic`. However, these scenarios are not part of our testing matrix and unforeseen problems might occur.

## API Overview

[`System.Net.Quic`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic?view=net-7.0) brings three major classes that enable usage of QUIC protocol:
- `QuicListener`: server side class for accepting incoming connections.
- `QuicConnection`: QUIC connection, corresponding to [RFC 9000 -  5. Connections](https://www.rfc-editor.org/rfc/rfc9000.html#name-connections).
- `QuicStream`: QUIC stream, corresponding to [RFC 9000 -   2. Streams ](https://www.rfc-editor.org/rfc/rfc9000.html#name-streams).

But before any usage of these classes, user code should check whether QUIC is currently supported, as `libmsquic` might be missing, or TLS 1.3 might not be supported. For that, both `QuicListener` and `QuicConnection` expose a static property `IsSupported`:
```C#
if (QuicListener.IsSupported)
{
    // Use QuicListener
}
else
{
    // Fallback/Error
}

if (QuicConnection.IsSupported)
{
    // Use QuicConnection
}
else
{
    // Fallback/Error
}
```
Note that, at the moment both these properties are in sync and will report the same value, but that might change in the future. So we recommend to check [`QuicListener.IsSupported`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.issupported?view=net-7.0) for server-scenarios and [`QuicConnection.IsSupported`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.issupported?view=net-7.0) for the client ones.

### QuicListener

[`QuicListener`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener?view=net-7.0) represents a server side class that accepts incoming connections from the clients. The listener is constructed and started with a static method [`QuicListener.ListenAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.listenasync?view=net-7.0). The method accepts an instance of [`QuicListenerOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclisteneroptions?view=net-7.0) class with all the settings necessary to start the listener and accept incoming connections. After that, listener is ready to hand out connections via [`AcceptConnectionAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.acceptconnectionasync?view=net-7.0). Connections returned by this method are always fully connected, meaning that the TLS handshake is finished and the connection is ready to be used. Finally, to stop listening and release all resources, [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.disposeasync?view=net-7.0) must be called.

The sample usage of `QuicListener`:
```C#
using System.Net.Quic;

// First, check if QUIC is supported.
if (!QuicListener.IsSupported)
{
    Console.WriteLine("QUIC is not supported, check for presence of libmsquic and support of TLS 1.3.");
    return;
}

// We want the same configuration for each incoming connection, so we prepare the connection options upfront and reuse them.
// This represents the minimal configuration necessary.
var serverConnectionOptions = new QuicServerConnectionOptions()
{
    // Used to abort stream if it's not properly closed by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultStreamErrorCode = 0x0A, // Protocol-dependent error code.

    // Used to close the connection if it's not done by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultCloseErrorCode = 0x0B, // Protocol-dependent error code.

    // Same options as for server side SslStream.
    ServerAuthenticationOptions = new SslServerAuthenticationOptions
    {
        // List of supported application protocols, must be the same or subset of QuicListenerOptions.ApplicationProtocols.
        ApplicationProtocols = new List<SslApplicationProtocol>() { "protocol-name" },
        // Server certificate, it can also be provided via ServerCertificateContext or ServerCertificateSelectionCallback.
        ServerCertificate = serverCertificate
    }
};

// Initialize, configure the listener and start listening.
var listener = await QuicListener.ListenAsync(new QuicListenerOptions()
{
    // Listening endpoint, port 0 means any port.
    ListenEndPoint = new IPEndPoint(IPAddress.Loopback, 0),
    // List of all supported application protocols by this listener.
    ApplicationProtocols = new List<SslApplicationProtocol>() { "protocol-name" },
    // Callback to provide options for the incoming connections, it gets called once per each connection.
    ConnectionOptionsCallback = (_, _, _) => ValueTask.FromResult(serverConnectionOptions)
});

// Accept and process the connections.
while (isRunning)
{
    // Accept will propagate any exceptions that occurred during the connection establishment,
    // including exceptions thrown from ConnectionOptionsCallback, caused by invalid QuicServerConnectionOptions or TLS handshake failures.
    var connection = await listener.AcceptConnectionAsync();

    // Process the connection...
}

// When finished, dispose the listener.
await listener.DisposeAsync();
```

More details about how this class was designed can be found in the [`QuicListener` API Proposal](https://github.com/dotnet/runtime/issues/67560) issue.

### QuicConnection

[`QuicConnection`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection?view=net-7.0) is a class used for both server and client side QUIC connections. Server side connections are created internally by the listener and handed out via [`QuicListener.AcceptConnectionAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.acceptconnectionasync?view=net-7.0). Client side connections must be opened and connected to the server. As with the listener, there's a static method [`QuicConnection.ConnectAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.connectasync?view=net-7.0) that instantiates and connects the connection. It accepts an instance of [`QuicClientConnectionOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicclientconnectionoptions?view=net-7.0), an analogous class to [`QuicServerConnectionOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicserverconnectionoptions?view=net-7.0). After that, the work with the connection doesn't differ between client and server. It can open outgoing streams and accept incoming ones. It also provides properties with information about the connection, like [`LocalEndPoint`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.localendpoint?view=net-7.0), [`RemoteEndPoint`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.remoteendpoint?view=net-7.0), or [`RemoteCertificate`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.remotecertificate?view=net-7.0).

When the work with the connection is done, it needs to be closed and disposed. QUIC protocol mandates using an application layer code for immediate closure, see [RFC 9000 - 10.2. Immediate Close](https://www.rfc-editor.org/rfc/rfc9000.html#name-immediate-close). For that, [`CloseAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.closeasync?view=net-7.0) with application layer code can be called or if not, [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.disposeasync?view=net-7.0) will use the code provided in [`QuicConnectionOptions.DefaultCloseErrorCode`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnectionoptions.defaultcloseerrorcode?view=net-7.0#system-net-quic-quicconnectionoptions-defaultcloseerrorcode). Either way, [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.disposeasync?view=net-7.0) must be called at the end of the work with the connection to fully release all the associated resources.

The sample usage of `QuicConnection`:
```C#
using System.Net.Quic;

// First, check if QUIC is supported.
if (!QuicConnection.IsSupported)
{
    Console.WriteLine("QUIC is not supported, check for presence of libmsquic and support of TLS 1.3.");
    return;
}

// This represents the minimal configuration necessary to open a connection.
var clientConnectionOptions = new QuicClientConnectionOptions()
{
    // End point of the server to connect to.
    RemoteEndPoint = listener.LocalEndPoint,

    // Used to abort stream if it's not properly closed by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultStreamErrorCode = 0x0A, // Protocol-dependent error code.

    // Used to close the connection if it's not done by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultCloseErrorCode = 0x0B, // Protocol-dependent error code.

    // Optionally set limits for inbound streams.
    MaxInboundUnidirectionalStreams = 10,
    MaxInboundBidirectionalStreams = 100,

    // Same options as for client side SslStream.
    ClientAuthenticationOptions = new SslClientAuthenticationOptions()
    {
        // List of supported application protocols.
        ApplicationProtocols = new List<SslApplicationProtocol>() { "protocol-name" }
    }
};

// Initialize, configure and connect to the server.
var connection = await QuicConnection.ConnectAsync(clientConnectionOptions);

Console.WriteLine($"Connected {connection.LocalEndPoint} --> {connection.RemoteEndPoint}");

// Open a bidirectional (can both read and write) outbound stream.
var outgoingStream = await connection.OpenOutboundStreamAsync(QuicStreamType.Bidirectional);

// Work with the outgoing stream ...

// To accept any stream on a client connection, at least one of MaxInboundBidirectionalStreams or MaxInboundUnidirectionalStreams of QuicConnectionOptions must be set.
while (isRunning)
{
    // Accept an inbound stream.
    var incomingStream = await connection.AcceptInboundStreamAsync();

    // Work with the incoming stream ...
}

// Close the connection with the custom code.
await connection.CloseAsync(0x0C);

// Dispose the connection.
await connection.DisposeAsync();
```

More details about how this class was designed can be found in the [`QuicConnection` API Proposal](https://github.com/dotnet/runtime/issues/68902) issue.

### QuicStream

[`QuicStream`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream?view=net-7.0) is the actual type that is used to send and receive data in QUIC protocol. It derives from ordinary [`Stream`](https://learn.microsoft.com/en-us/dotnet/api/system.io.stream?view=net-7.0) and can be used as such, but it also offers several features that are specific to QUIC protocol. Firstly, a QUIC stream can either be unidirectional or bidirectional, see [RFC 9000 - 2.1. Stream Types and Identifiers](https://www.rfc-editor.org/rfc/rfc9000.html#name-stream-types-and-identifier). A bidirectional stream is able to send and receive data on both sides, whereas unidirectional stream can only write from the initiating side and read on the accepting one. Each peer can limit how many concurrent stream of each type is willing to accept, see [`QuicConnectionOptions.MaxInboundBidirectionalStreams`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnectionoptions.maxinboundbidirectionalstreams?view=net-7.0) and [`QuicConnectionOptions.MaxInboundUnidirectionalStreams`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnectionoptions.maxinboundunidirectionalstreams?view=net-7.0).

Another particularity of QUIC stream is ability to explicitly close the writing side in the middle of work with the stream, see [`CompleteWrites`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream.completewrites?view=net-7.0) or [`WriteAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream.writeasync?view=net-7.0#system-net-quic-quicstream-writeasync(system-readonlymemory((system-byte))-system-boolean-system-threading-cancellationtoken)) overload with `completeWrites` argument. Closing of the writing side lets the peer know that no more data will arrive, yet the peer still can continue sending (in case of a bidirectional stream). This is useful in scenarios like HTTP request/response exchange when the client sends the request and closes the writing side to let the server know that this is the end of the request content. Server is still able to send the response after that, but knows that no more data will arrive from the client. And for erroneous cases, either writing or reading side of the stream can be aborted, see [`Abort`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream.abort?view=net-7.0). The behavior of the individual methods for each stream type is summarized in the following table (note that both client and server can open and accept streams):

|            | peer opening stream  | peer accepting stream  |
| -          | -       | -       |
| `CanRead`  | _bidirectional_: `true`<br/> _unidirectional_: `false` | `true`  |
| `CanWrite` | `true`  | _bidirectional_: `true`<br/> _unidirectional_: `false` |
| `ReadAsync` | _bidirectional_: reads data<br/> _unidirectional_: `InvalidOperationException` | reads data |
| `WriteAsync` | sends data => peer read returns the data | _bidirectional_: sends data => peer read returns the data<br/> _unidirectional_: `InvalidOperationException`  |
| `CompleteWrites` | closes writing side => peer read returns 0 | _bidirectional_: closes writing side => peer read returns 0<br/> _unidirectional_: no-op |
| `Abort(QuicAbortDirection.Read)` | _bidirectional_: [STOP_SENDING](https://www.rfc-editor.org/rfc/rfc9000.html#name-stop_sending-frames) => peer write throws `QuicException(QuicError.OperationAborted)`<br/> _unidirectional_: no-op | [STOP_SENDING](https://www.rfc-editor.org/rfc/rfc9000.html#name-stop_sending-frames) => peer write throws `QuicException(QuicError.OperationAborted)`|
| `Abort(QuicAbortDirection.Write)` | [RESET_STREAM](https://www.rfc-editor.org/rfc/rfc9000.html#name-reset_stream-frames) => peer read throws `QuicException(QuicError.OperationAborted)` | _bidirectional_: [RESET_STREAM](https://www.rfc-editor.org/rfc/rfc9000.html#name-reset_stream-frames) => peer read throws `QuicException(QuicError.OperationAborted)`<br/> _unidirectional_: no-op |

On top of these methods, `QuicStream` offers two specialized properties to get notified whenever either reading or writing side of the stream has been closed: [`ReadsClosed`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream.readsclosed?view=net-7.0) and [`WritesClosed`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream.writesclosed?view=net-7.0). Both return a `Task` that completes with its corresponding side getting closed, whether it be success or abort, in which case the `Task` will contain appropriate exception. These properties are useful when the user code needs to know about stream side getting closed without issuing call to `ReadAsync` or `WriteAsync`.

Finally, when the work with the stream is done, it needs to be disposed with [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicstream.disposeasync?view=net-7.0). The dispose will make sure that both reading and/or writing side - depending on the stream type - is closed. If stream hasn't been properly read till the end, dispose will issue an equivalent of `Abort(QuicAbortDirection.Read)`. However, if stream writing side hasn't been closed, it will be gracefully closed as it would be with `CompleteWrites`. The reason for this difference is to make sure that scenarios working with an ordinary `Stream` behave as expected and lead to a successful path. Consider the following example:
```C#
// Work done with all different types of streams.
async Task WorkWithStream(Stream stream)
{
    // This will dispose the stream at the end of the scope.
    await using (stream)
    {
        // Simple echo, read data and send them back.
        byte[] buffer = new byte[1024];
        int count = 0;
        // The loop stops when read returns 0 bytes as is common for all streams.
        while ((count = await stream.ReadAsync(buffer)) > 0)
        {
            await stream.WriteAsync(buffer.AsMemory(0, count));
        }
    }
}

// Open a QuicStream and pass to the common method.
var quicStream = await connection.OpenOutboundStreamAsync(QuicStreamType.Bidirectional);
await WorkWithStream(quicStream);

```

The sample usage of `QuicStream` in client scenario:
```C#
// Consider connection from the connection example, open a bidirectional stream.
await using var stream = await connection.OpenStreamAsync(QuicStreamType.Bidirectional, cancellationToken);

// Send some data.
await stream.WriteAsync(data, cancellationToken);
await stream.WriteAsync(data, cancellationToken);

// End the writing-side together with the last data.
await stream.WriteAsync(data, endStream: true, cancellationToken);
// Or separately.
stream.CompleteWrites();

// Read data until the end of stream.
while (await stream.ReadAsync(buffer, cancellationToken) > 0)
{
    // Handle buffer data...
}

// DisposeAsync called by await using at the top.
```

And the sample usage of `QuicStream` in server scenario:
```C#
// Consider connection from the connection example, accept a stream.
await using var stream = await connection.AcceptStreamAsync(cancellationToken);

if (stream.Type != QuicStreamType.Bidirectional)
{
    Console.WriteLine($"Expected bidirectional stream, got {stream.Type}");
    return;
}

// Read the data.
while (stream.ReadAsync(buffer, cancellationToken) > 0)
{
    // Handle buffer data...

    // Client completed the writes, the loop might be exited now without another ReadAsync.
    if (stream.ReadsCompleted.IsCompleted)
    {
        break;
    }
}

// Listen for Abort(QuicAbortDirection.Read) from the client.
var writesClosedTask = WritesClosedAsync(stream);
async ValueTask WritesClosedAsync(QuicStream stream)
{
    try
    {
        await stream.WritesClosed;
    }
    catch (Exception ex)
    {
        // Handle peer aborting our writing side ...
    }
}

// DisposeAsync called by await using at the top.
```

More details about how this class was designed can be found in the [`QuicStream` API Proposal](https://github.com/dotnet/runtime/issues/69675) issue.

## Future

As `System.Net.Quic` is newly made public and we have only limited usages, we'll appreciate any bug reports or insights on the API shape. Thanks to APIs being in preview mode, we still have a chance to tweak them for .NET 8 based on the feedback we get. Issues can be filed in [dotnet/runtime](https://github.com/dotnet/runtime) repository.



# Security

## Negotiate API
[Windows Authentication](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-overview) is an encompassing term for multiple technologies used in enterprises to authenticate users and applications against a central authority, usually the domain controller. It enables scenarios like single sign-on to email services or intranet applications. The underlying technologies used for the authentication are Kerberos, NTLM, and the encompassing Negotiate protocol, where the most suitable technology is chosen for a specific authentication scenario.

Prior to .NET 7, Windows Authentication was exposed in the high-level APIs, like `HttpClient` (`Negotiate` and `NTLM` authentication schemes), `SmtpClient` (`GSSAPI` and `NTLM` authentication schemes), `NegotiateStream`, [ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/windowsauth?view=aspnetcore-7.0), and the SQL Server client libraries. While that covers most scenarios for end users, it is limiting for library authors. Third-party libraries like the [Npgsql PostgreSQL client](https://www.npgsql.org/), [MailKit](https://github.com/jstedfast/MailKit), [Apache Kudu client](https://github.com/xqrzd/kudu-client-net) and others needed to resort to various tricks to implement the same authentication schemes for low-level protocols that were not built on HTTP or other available high-level building blocks.

.NET 7 introduces new API providing low-level building blocks to perform the authentication exchange for the above mentioned protocols, see [Issue #69920](https://github.com/dotnet/runtime/issues/69920). As with all the other APIs in .NET, it is built with cross-platform interoperability in mind. On Linux, macOS, iOS, and other similar platforms, it uses the GSSAPI system library. On Windows, it relies on the [SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/sspi-architectural-overview) library. For platforms where system implementation is not available, such as Android and tvOS, a limited client-only implementation is present.

### How to use the API

In order to understand how the authentication API works, let's start with an example of how the authentication session looks in a high-level protocol like SMTP. The example is taken from [Microsoft protocol documentation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-ssean/0fa15a99-af88-428a-a51d-742b8450d877) that explains it in greater detail.

```
S: 220 server.contoso.com Authenticated Receive Connector
C: EHLO client.contoso.com
S: 250-server-contoso.com Hello [203.0.113.1]
S: 250-AUTH GSSAPI NTLM
S: 250 OK
C: AUTH GSSAPI <token1>
S: 334 <token2>
C: <token3>
S: 235 2.7.0 Authentication successful
```

The authentication starts with the client producing a challenge token. Then the server produces a response. The client processes the response, and a new challenge is sent to the server. This challenge/response exchange can happen multiple times. It finishes when either party rejects the authentication or when both parties accept the authentication. The format of the tokens is defined by the Windows Authentication protocols, while the encapsulation is part of the high-level protocol specification. In this example, the SMTP protocol prepends the `334` code to tell the client that the server produced an authentication response, and the `235` code indicates successful authentication.

The bulk of the new API centers around a new [`NegotiateAuthentication`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication?view=net-7.0) class. It is used to instantiate the context for client-side or server-side authentication. There are various options to specify requirements for establishing the authenticated session, like requiring encryption or determining that a specific protocol (Negotiate, Kerberos, or NTLM) is to be used. Once the parameters are specified, the authentication proceeds by exchanging the authentication challenges/responses between the client and the server. The [`GetOutgoingBlob`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication.getoutgoingblob?view=net-7.0) method is used for that. It can work either on byte spans or base64-encoded strings.

The following code will perform both, client and server, part of the authentication for the current user on the same machine:

```C#
using System.Net;
using System.Net.Security;

var serverAuthentication = new NegotiateAuthentication(new NegotiateAuthenticationServerOptions { });
var clientAuthentication = new NegotiateAuthentication(
    new NegotiateAuthenticationClientOptions
    {
        Package = "Negotiate",
        Credential = CredentialCache.DefaultNetworkCredentials,
        TargetName = "HTTP/localhost",
        RequiredProtectionLevel = ProtectionLevel.Sign
    });

string? serverBlob = null;
while (!clientAuthentication.IsAuthenticated)
{
    // Client produces the authentication challenge, or response to server's challenge
    string? clientBlob = clientAuthentication.GetOutgoingBlob(serverBlob, out var clientStatusCode);
    if (clientStatusCode == NegotiateAuthenticationStatusCode.ContinueNeeded)
    {
        // Send the client blob to the server; this would normally happen over a network
        Console.WriteLine($"C: {clientBlob}");
        serverBlob = serverAuthentication.GetOutgoingBlob(clientBlob, out var serverStatusCode);
        if (serverStatusCode != NegotiateAuthenticationStatusCode.Completed &&
            serverStatusCode != NegotiateAuthenticationStatusCode.ContinueNeeded)
        {
            Console.WriteLine($"Server authentication failed with status code {serverStatusCode}");
            break;
        }
        Console.WriteLine($"S: {serverBlob}");
    }
    else
    {
        Console.WriteLine(
            clientStatusCode == NegotiateAuthenticationStatusCode.Completed ?
            "Successfully authenticated" :
            $"Authentication failed with status code {clientStatusCode}");
        break;
    }
}
```

Once the authenticated session is established, the `NegotiateAuthentication` instance can be used to sign/encrypt the outgoing messages and verify/decrypt the incoming messages. This is done through the [Wrap](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication.wrap?view=net-7.0) and [Unwrap](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication.unwrap?view=net-7.0) methods.

This change was done as well as this part of the article was written by a community contributor [@filipnavara](https://github.com/filipnavara).


## Options for certificate validation

When a client receives server's certificate, or vice-versa if client certificate is requested, the certificate is validated via [`X509Chain`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509chain?view=net-7.0). The validation happens always, even if [`RemoteCertificateValidationCallback`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.remotecertificatevalidationcallback?view=net-7.0) is provided, and additional certificates might get downloaded during the validation. Several issues were raised as there was no way to control this behavior. Among them were asks to completely prevent certificate download, put a timeout on it, or to provide custom store to get the certificates from. To mitigate this whole group of issues, we decided to introduce a new property `CertificateChainPolicy` on both [`SslClientAuthenticationOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.sslclientauthenticationoptions?view=net-7.0) and [`SslServerAuthenticationOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.sslserverauthenticationoptions?view=net-7.0). The goal of this property is to override the default behavior of [`SslStream`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.sslstream?view=net-7.0) when building the chain during [`AuthenticateAsClientAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.sslstream.authenticateasclientasync?view=net-7.0) / [`AuthenticateAsServerAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.sslstream.authenticateasserverasync?view=net-7.0) operation. In normal circumstances, [`X509ChainPolicy`](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509chainpolicy?view=net-7.0) is constructed automatically in the background. But if this new property is specified, it will take precedence and be used instead, giving the user full control over the certificate validation process.

The usage of the chain policy might look like:
```C#
// Client side:
var policy = new X509ChainPolicy();
policy.TrustMode = X509ChainTrustMode.CustomRootTrust;
policy.ExtraStore.Add(s_ourPrivateRoot);
policy.UrlRetrievalTimeout = TimeSpan.FromSeconds(3);

var options  = new SslClientAuthenticationOptions();
options.TargetHost = "myServer";
options.CertificateChainPolicy = policy;

var sslStream = new SslStream(transportStream);
sslStream.AuthenticateAsClientAsync(options, cancellationToken);

// Server side:
var policy = new X509ChainPolicy();
policy.DisableCertificateDownloads = true;

var options  = new SslServerAuthenticationOptions();
options.CertificateChainPolicy = policy;

var sslStream = new SslStream(transportStream);
sslStream.AuthenticateAsServerAsync(options, cancellationToken);
```

More info can be found the API proposal [Issue #71191](https://github.com/dotnet/runtime/issues/71191).

## Performance

Most of the networking perfomance improvements in .NET 7 are covered by Stephen's article [Performance Improvements in .NET 7 - Networking](https://devblogs.microsoft.com/dotnet/performance_improvements_in_net_7/#networking), but some of the security ones are worth mentioning again.

### TLS Resume

Establishing new TLS connection is fairly expensive operation as it requires multiple steps and several round trips. In scenarios where connection to the same server is re-created very often, time consumed by the handshakes will add up. TLS offers feature to mitigate this called Session Resumption, see [RFC 5246 - 7.3.  Handshake Protocol Overview](https://www.rfc-editor.org/rfc/rfc5246.html#section-7.3) and [RFC 8446 - 2.2.  Resumption and Pre-Shared Key](https://www.rfc-editor.org/rfc/rfc8446#section-2.2). In short, during the handshake, client can send an identification of previously established TLS session and if server agrees, the security context gets re-established based on the cached data from the previous connection. Even though the mechanics differ for different TLS versions, the end goal is the same, save a round-trip and some CPU time when re-establishing connection to a previously connected server.
This feature is automatically provided by SChannel on Windows, but with OpenSSL on Linux it required several changes to enable this:
- Server side (stateless): [PR #57079](https://github.com/dotnet/runtime/pull/57079) and [PR #63030](https://github.com/dotnet/runtime/pull/63030)
- Client side: [PR #64369](https://github.com/dotnet/runtime/pull/64369)
- Cache size control: [PR #69065](https://github.com/dotnet/runtime/pull/69065)

If caching the TLS context is not desired, it can be turned off process-wide with either environment variable "DOTNET_SYSTEM_NET_SECURITY_DISABLETLSRESUME" or via [`AppContext.SetSwitch`](https://learn.microsoft.com/en-us/dotnet/api/system.appcontext.setswitch?view=net-7.0) "System.Net.Security.TlsCacheSize".

### OCSP Stapling

Online Certificate Status Protocol (OCSP) Stapling is a mechanism for server to provide signed and timestamped proof (OCSP response) that the sent certificate has not been revoked, see [RFC 6961](https://www.rfc-editor.org/rfc/rfc6961). As a result, client doesn't need to contact the OCSP server itself, reducing the number of requests necessary to establish the connection as well as the load exerted on the OCSP server. And as the OCSP response needs to be signed by the certificate authority (CA), it cannot be forged by the server providing the certificate. We didn't take advantage of this TLS feature until this release, for more details see [Issue #33377](https://github.com/dotnet/runtime/issues/33377).


## Consistency across platforms

We are aware, that some of the functionality provided by .NET is available only on certain platforms. But each release we try to narrow the gap a bit more. In .NET 7, we made several changes in the networking security space to improve the disparity:
- Support for post-handshake authentication on Linux for TLS 1.3: [PR #64268](https://github.com/dotnet/runtime/pull/64268)
- Remote certificate is now set on Windows in [`SslClientAuthenticationOptions.LocalCertificateSelectionCallback`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.sslclientauthenticationoptions.localcertificateselectioncallback?view=net-7.0): [PR #65134](https://github.com/dotnet/runtime/pull/65134)
- Support for sending names of trusted CA in TLS handshake on OSX and Linux: [PR #65195](https://github.com/dotnet/runtime/pull/65195)



# WebSockets

## WebSocket Handshake Response Details

Prior to .NET 7, server's response part of WebSocket's opening handshake (HTTP response to Upgrade request) was hidden inside `ClientWebSocket` implementation, and all handshake errors would surface as `WebSocketException` without much details beside the exception message. However, the information about HTTP response headers and status code might be important in both failure and success scenarios.

In case of failure, HTTP status code can help to distinguish between retriable and non-retriable errors (e.g. server doesn't support WebSockets at all, or it was just a transient network error). Headers might also contain additional information on how to handle the situation. The headers are useful even in case of a successful WebSocket handshake, e.g. they can contain token tied to a session, information related to subprotocol version, or that the server can go down soon.

.NET 7 adds a setting [ClientWebSocketOptions.CollectHttpResponseDetails](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.collecthttpresponsedetails?view=net-7.0) that enables collecting upgrade response details in `ClientWebSocket` instance during `ClientWebSocket.ConnectAsync` call. You can later access the data using [ClientWebSocket.HttpStatusCode](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.httpstatuscode?view=net-7.0) and [ClientWebSocket.HttpResponseHeaders](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.httpresponseheaders?view=net-7.0) properties, even in case of `ClientWebSocket.ConnectAsync` throwing an exception. Note that in the exceptional case, the information might be unavailable, i.e. if the server never responded to the request.

Also note that in case of a successful connect and after consuming `HttpResponseHeaders` data, you can reduce `ClientWebSocket`'s memory footprint by setting `ClientWebSocket.HttpResponseHeaders` property to `null`.

```c#
var ws = new ClientWebSocket();
ws.Options.CollectHttpResponseDetails = true;
try
{
    await ws.ConnectAsync(uri, cancellationToken);
    // success scenario
    ProcessSuccess(ws.HttpResponseHeaders);
    ws.HttpResponseHeaders = null; // clean up (if needed)
}
catch (WebSocketException)
{
    // failure scenario
    if (ws.HttpStatusCode != 0)
    {
        ProcessFailure(ws.HttpStatusCode, ws.HttpResponseHeaders);
    }
}
```

## Providing external HTTP client

In the default case, `ClientWebSocket` uses a cached static `HttpMessageInvoker` instance to execute HTTP Upgrade requests. However, there are `ClientWebSocket` [options](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.options?view=net-7.0#system-net-websockets-clientwebsocket-options) that prevent caching the invoker, such as `ClientWebSocketOptions.Proxy`, `ClientWebSocketOptions.ClientCertificates` or `ClientWebSocketOptions.Cookies`. `HttpMessageInvoker` instance with these parameters is not safe to be reused and needs be created each time `ClientWebSocket.ConnectAsync` is called. This results in many unnecessary allocations and makes reuse of `HttpMessageInvoker` connection pool impossible.

.NET 7 allows you to pass an existing `HttpClient` or `HttpMessageInvoker` instance to `ClientWebSocket.ConnectAsync` call, using the [ConnectAsync(Uri, HttpMessageInvoker, CancellationToken)](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.connectasync?view=net-7.0#system-net-websockets-clientwebsocket-connectasync(system-uri-system-net-http-httpmessageinvoker-system-threading-cancellationtoken)) overload. In that case, HTTP Upgrade request would be executed using the provided instance.

```c#
var httpClient = new HttpClient();

var ws = new ClientWebSocket();
await ws.ConnectAsync(uri, httpClient, cancellationToken);
```

Note that in case a custom HTTP invoker is passed, any of the following `ClientWebSocketOptions` *must not* be set, and should be set up on the HTTP invoker instead:
- [ClientCertificates](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.clientcertificates?view=net-7.0)
- [Cookies](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.cookies?view=net-7.0)
- [Credentials](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.credentials?view=net-7.0)
- [Proxy](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.proxy?view=net-7.0)
- [RemoteCertificateValidationCallback](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.remotecertificatevalidationcallback?view=net-7.0)
- [UseDefaultCredentials](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.usedefaultcredentials?view=net-7.0)

This is how you can set up all of these options on the HTTP invoker instance:

```c#
var handler = new HttpClientHandler();
handler.CookieContainer = cookies;
handler.UseCookies = cookies != null;
handler.ServerCertificateCustomValidationCallback = remoteCertificateValidationCallback;
handler.Credentials = useDefaultCredentials ?
    CredentialCache.DefaultCredentials :
    credentials;
if (proxy == null)
{
    handler.UseProxy = false;
}
else
{
    handler.Proxy = proxy;
}
if (clientCertificates?.Count > 0)
{
    handler.ClientCertificates.AddRange(clientCertificates);
}
var invoker = new HttpMessageInvoker(handler);

var ws = new ClientWebSocket();
await ws.ConnectAsync(uri, invoker, cancellationToken);
```

## WebSockets over HTTP/2

.NET 7 also adds an ability to use WebSocket protocol over HTTP/2, as described in [RFC 8441](https://www.rfc-editor.org/rfc/rfc8441). With that, WebSocket connection is established over a single stream on HTTP/2 connection. This allows for a single TCP connection to be shared between several WebSocket connections and HTTP requests at the same time, resulting in more efficient use of the network.

To enable WebSockets over HTTP/2, you can set [`ClientWebSocketOptions.HttpVersion`](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.httpversion?view=net-7.0) option to `HttpVersion.Version20`. You can also enable upgrade/downgrade of HTTP version used by setting [`ClientWebSocketOptions.HttpVersionPolicy`](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.httpversionpolicy?view=net-7.0) property. These options will behave in the same way `HttpRequestMessage.Version` and `HttpRequestMessage.VersionPolicy` behave.

For example, the following code would probe for HTTP/2 WebSockets, and if a WebSocket connection cannot be established, it will fallback to HTTP/1.1:

```c#
var ws = new ClientWebSocket();
ws.Options.HttpVersion = HttpVersion.Version20;
ws.Options.HttpVersionPolicy = HttpVersionPolicy.RequestVersionOrLower;
await ws.ConnectAsync(uri, httpClient, cancellationToken);
```

The combination of `HttpVersion.Version11` and `HttpVersionPolicy.RequestVersionOrHigher` will result in the same behavior as above, while `HttpVersionPolicy.RequestVersionExact` will disallow upgrade/downgrade of HTTP version.

By default, `HttpVersion = HttpVersion.Version11` and `HttpVersionPolicy = HttpVersionPolicy.RequestVersionOrLower` are set, meaning that only HTTP/1.1 will be used.

The ability to multiplex WebSocket connections and HTTP requests over a single HTTP/2 connection is a crucial part of this feature. For it to work as expected, you need to pass and reuse the same `HttpMessageInvoker` instance (e.g. `HttpClient`) from your code when calling `ConnectAsync`, i.e. use [ClientWebSocket.ConnectAsync(Uri, HttpMessageInvoker, CancellationToken)](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.connectasync?view=net-7.0#system-net-websockets-clientwebsocket-connectasync(system-uri-system-net-http-httpmessageinvoker-system-threading-cancellationtoken)) overload. This will reuse the connection pool within the `HttpMessageInvoker` instance for the multiplexing.

# Final Notes

We try to pick the most interesting and impactful changes in the networking space. The article doesn't contain all the changes we did, but they can be found in [dotnet/runtime](https://github.com/dotnet/runtime) repository. As usual, performance improvements are mostly omitted as they get their spot in Stephens's article [Performance Improvements in .NET 7](https://devblogs.microsoft.com/dotnet/performance_improvements_in_net_7). We'd also like to hear from you, so if you encounter an issue or have any feedback, you can file it in [our GitHub](https://github.com/dotnet/runtime/issues). Finally, you can find similar blog posts for previous releases here: [.NET 6 Networking Improvements](https://devblogs.microsoft.com/dotnet/dotnet-6-networking-improvements/), [.NET 5 Networking Improvements](https://devblogs.microsoft.com/dotnet/net-5-new-networking-improvements/).