# QUIC

QUIC is a new, transport layer protocol. It has been recently standardized in [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html). It uses UDP as an underlying protocol and it's inherently secure as it mandates TLS 1.3 usage, see [RFC 9001](https://www.rfc-editor.org/rfc/rfc9001.html). Another interesting difference from well known transport protocols as TCP and UDP is that it offers stream multiplexing over a shared connection. The connection between two endpoints negotiates secrets and settings and then the individual streams exchange data between the peers. This allows to have multiple, concurrent, detached data streams that do not affect each other.

QUIC itself doesn't define any semantics for the exchanged data as it's a transport protocol but it's used in application layer protocols. For example in [HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html) or in [SMB over QUIC](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-over-quic). It can also be used for any custom defined protocol.

The protocol offers many advantages over TCP with TLS. For example, faster connection establishment as it doesn't require as many round trips as TCP with TLS on top. Or avoidance of head-of-line blocking problem where one lost datagram doesn't block data of all the other streams. On the other hand, there are disadvantages that come with using QUIC. As it is a new protocol, its adoption is still growing and is limited. Apart from that, QUIC traffic might be even blocked by some networking components.

## QUIC in .NET

We introduced QUIC implementation in .NET 5 in `System.Net.Quic` library. However up until now, the library was strictly internal and served only for own implementation of HTTP/3. With the release of .NET 7, we're making the library public and we're exposing its APIs. Since we had only `HttpClient` and Kestrel as consumers of the APIs, we decided to keep them as [preview feature](https://github.com/dotnet/designs/blob/main/accepted/2021/preview-features/preview-features.md). It gives us ability to tweak the API in the next release before we settle on the final shape.

From the implementation perspective, `System.Net.Quic` depends on [MsQuic](https://github.com/microsoft/msquic), native implementation of the QUIC protocol. As a result, `System.Net.Quic` platform support and dependencies are inherited from MsQuic and documented in [HTTP/3 Platform dependencies](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-http3#platform-dependencies). It is still possible to build MsQuic manually, whether against SChannel or OpenSSL, and use it with `System.Net.Quic`. However, these scenarios are not part of our testing matrix and unforeseen problems might occur.

## API Overview

`System.Net.Quic` brings three major classes that enable usage of QUIC protocol:
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
Note that at the moment, both these properties are in sync and will report the same value, but that might change in the future. So we recommend to check `QuicListener.IsSupported` for server-scenarios and `QuicConnection.IsSupported` for the client ones.

### QuicListener

`QuicListener` represents a server side class that accepts incoming connections from the clients. When it is initialized and started,

### QuicConnection

### QuicStream