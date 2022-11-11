# QUIC

QUIC is a new transport layer protocol. It has been recently standardized in [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html). It uses UDP as an underlying protocol and it's inherently secure as it mandates TLS 1.3 usage. Major difference from TCP and UDP is that it's multiplexed. It defines a connection between two endpoints, for which secrets and settings are negotiated. Then, the connection can have multiple streams on which data are exchanged between the peers.

Up until .NET 7, `System.Net.Quic` library was private and only used internally by `HttpClient` and ASP.NET Core server for HTTP/3. With this release, we're making the library public and we're exposing its APIs. Since we had only `HttpClient` and Kestrel as consumers of the APIs, we decided to make them [preview feature](https://github.com/dotnet/designs/blob/main/accepted/2021/preview-features/preview-features.md). It gives us ability to tweak the API in the next release (.NET 8) before we settle on the final shape.


## QuicListener

## QuicConnection

## QuicStream