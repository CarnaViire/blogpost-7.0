# QUIC

QUIC is a new, transport layer protocol. It has been recently standardized in [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html). It uses UDP as an underlying protocol and it's inherently secure as it mandates TLS 1.3 usage, see [RFC 9001](https://www.rfc-editor.org/rfc/rfc9001.html). Another interesting difference from well known transport protocols as TCP and UDP is that it offers stream multiplexing over a shared connection. The connection between two endpoints negotiates secrets and settings and then the individual streams exchange data between the peers. This allows to have multiple, concurrent, detached data streams that do not affect each other.

QUIC itself doesn't define any semantics for the exchanged data as it's a transport protocol but it's used in application layer protocols. For example in [HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html) or in [SMB over QUIC](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-over-quic). It can also be used for any custom defined protocol.

The protocol offers many advantages over TCP with TLS. For example, faster connection establishment as it doesn't require as many round trips as TCP with TLS on top. Or avoidance of head-of-line blocking problem where one lost datagram doesn't block data of all the other streams. On the other hand, there are disadvantages that come with using QUIC. As it is a new protocol, its adoption is still growing and is limited. Apart from that, QUIC traffic might be even blocked by some networking components.

## QUIC in .NET

We introduced QUIC implementation in .NET 5 in `System.Net.Quic` library. However up until now, the library was strictly internal and served only for own implementation of HTTP/3. With the release of .NET 7, we're making the library public and we're exposing its APIs. Since we had only `HttpClient` and Kestrel as consumers of the APIs, we decided to keep them as [preview feature](https://github.com/dotnet/designs/blob/main/accepted/2021/preview-features/preview-features.md). It gives us ability to tweak the API in the next release before we settle on the final shape.

From the implementation perspective, `System.Net.Quic` depends on [MsQuic](https://github.com/microsoft/msquic), native implementation of the QUIC protocol. As a result, `System.Net.Quic` platform support and dependencies are inherited from MsQuic and documented in [HTTP/3 Platform dependencies](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-http3#platform-dependencies). It is still possible to build MsQuic manually, whether against SChannel or OpenSSL, and use it with `System.Net.Quic`. However, these scenarios are not part of our testing matrix and unforeseen problems might occur.

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
Note that at the moment, both these properties are in sync and will report the same value, but that might change in the future. So we recommend to check [`QuicListener.IsSupported`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener?view=net-7.0) for server-scenarios and [`QuicConnection.IsSupported`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.issupported?view=net-7.0) for the client ones.

### QuicListener

[`QuicListener`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener?view=net-7.0) represents a server side class that accepts incoming connections from the clients. The listener is constructed and started with a static method [`QuicListener.ListenAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.listenasync?view=net-7.0). The method accepts an instance of [`QuicListenerOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclisteneroptions?view=net-7.0) class with all the settings necessary to start the listener and accept incoming connections. After that, listener is ready to hand out connections via [`AcceptConnectionAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.acceptconnectionasync?view=net-7.0). Connections returned by this method are always fully connected, meaning that the TLS handshake is finished and the connection is ready to be used. Finally to stop listening and release all resources, [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.disposeasync?view=net-7.0) must be called.

The sample usage of `QuicListener`:
```C#
using System.Net.Quic;

// First, check if QUIC is supported.
if (!QuicListener.IsSupported)
{
    Console.WriteLine("QUIC is not supported, check for presence of libmsquic and support of TLS 1.3.");
    return;
}

// We want the same configuration for each incoming connection, so we prepare the connection option upfront and reuse them.
// This represents the minimal configuration necessary to accept a connection.
var serverConnectionOptions = new QuicServerConnectionOptions()
{
    // Used to abort stream if it's not properly closed by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultStreamErrorCode = 123,

    // Used to close the connection if it's not done by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultCloseErrorCode = 456,

    // Same options as for server side SslStream.
    ServerAuthenticationOptions = new SslServerAuthenticationOptions
    {
        // List of supported application protocols, must be the same or subset of QuicListenerOptions.ApplicationProtocols.
        ApplicationProtocols = new List<SslApplicationProtocol>() { SslApplicationProtocol.Http3 },
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
    ApplicationProtocols = new List<SslApplicationProtocol>() { SslApplicationProtocol.Http3 },
    // Callback to provide options for the incoming connections, it gets once called per each of them.
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

[`QuicConnection`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection?view=net-7.0) is a class used for both server and client side QUIC connections. Server side connections are created internally by the listener and handed out via [`QuicListener.AcceptConnectionAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclistener.acceptconnectionasync?view=net-7.0). On the other hand, client side connections must be opened and connected to the server. As with the listener, there's a static method [`QuicConnection.ConnectAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.connectasync?view=net-7.0) that instantiates and connects the connection. It accepts an instance of [`QuicClientConnectionOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicclientconnectionoptions?view=net-7.0), an analogous class to [`QuicServerConnectionOptions`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicserverconnectionoptions?view=net-7.0) returned from [`QuicListenerOptions.ConnectionOptionsCallback`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quiclisteneroptions.connectionoptionscallback?view=net-7.0). After that, the work with the connection doesn't differ between client and server. It can open outgoing streams and accept incoming ones. It also provides interesting properties like [`LocalEndPoint`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.localendpoint?view=net-7.0), [`RemoteEndPoint`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.remoteendpoint?view=net-7.0), or [`RemoteCertificate`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.remotecertificate?view=net-7.0).

When the work with the connection is done, it needs to be closed and disposed. However, QUIC protocol mandates using an application layer code for immediate closure, see [RFC 9000 - 10.2. Immediate Close](https://www.rfc-editor.org/rfc/rfc9000.html#name-immediate-close). For that, [`CloseAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.closeasync?view=net-7.0) with application layer code can be called or if not, [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.disposeasync?view=net-7.0) will use the code provided in [`QuicConnectionOptions.DefaultCloseErrorCode`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnectionoptions.defaultcloseerrorcode?view=net-7.0#system-net-quic-quicconnectionoptions-defaultcloseerrorcode). Either way, [`DisposeAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.net.quic.quicconnection.disposeasync?view=net-7.0) must be called at the end of the work with the connection to fully release all the associated resources.

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
    DefaultStreamErrorCode = 321,

    // Used to close the connection if it's not done by the user.
    // See https://www.rfc-editor.org/rfc/rfc9000.html#name-application-protocol-error-
    DefaultCloseErrorCode = 654,

    // Optionally set limits for inbound streams.
    MaxInboundUnidirectionalStreams = 10,
    MaxInboundBidirectionalStreams = 100,

    // Same options as for client side SslStream.
    ClientAuthenticationOptions = new SslClientAuthenticationOptions()
    {
        // List of supported application protocols.
        ApplicationProtocols = new List<SslApplicationProtocol>() { SslApplicationProtocol.Http3 }
    }
};

// Initialize, configure and connect the connection.
var connection = await QuicConnection.ConnectAsync(clientConnectionOptions);

Console.WriteLine($"Connected {connection.LocalEndPoint} --> {connection.RemoteEndPoint}");

// Open a bidirectional (can write and can read) outbound stream.
var outgoingStream = await connection.OpenOutboundStreamAsync(QuicStreamType.Bidirectional);

// Work with the outgoing stream ...

// To accept any stream on a client connection at least one of MaxInboundBidirectionalStreams or MaxInboundUnidirectionalStreams of QuicConnectionOptions must be set.
while (isRunning)
{
    // Accept an inbound stream.
    var incomingStream = await connection.AcceptInboundStreamAsync();

    // Work with the incoming stream ...
}

// Close the connection with the custom code.
await connection.CloseAsync(789);

// Dispose the connection.
await connection.DisposeAsync();
```

More details about how this class was designed can be found in the [`QuicConnection` API Proposal](https://github.com/dotnet/runtime/issues/68902) issue.

### QuicStream