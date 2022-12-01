# HTTP/3

HTTP/3 support in `HttpClient` was already feature complete in the previous .NET release, so we mostly concentrated our efforts in this space on `System.Net.Quic` itself. Despite that we did introduce few fixes and changes in .NET 7.

The most important change is that HTTP/3 is now enabled by default ([PR #73153](https://github.com/dotnet/runtime/pull/73153)). It doesn't mean that all HTTP requests will from now on prefer HTTP/3, but in certain cases they might upgrade to it. For it to happen, the request must opt into version upgrade via [`HttpRequestMessage.VersionPolicy`](https://learn.microsoft.com/cs-cz/dotnet/api/system.net.http.httprequestmessage.versionpolicy?view=net-7.0) set to `RequestVersionOrHigher`. Then, if the server announces HTTP/3 authority in `Alt-Svc` header, `HttpClient` will use it for further requests, see [RFC 9114 - 3.1.1. HTTP Alternative Services](https://www.rfc-editor.org/rfc/rfc9114.html#name-http-alternative-services).

To name a few other interesting changes:
- HTTP telemetry was extended to cover HTTP/3 as well - [Issue #40896](https://github.com/dotnet/runtime/issues/40896).
- Improved exception details for if QUIC connection cannot be established - [Issue #70949](https://github.com/dotnet/runtime/issues/70949)
- Proper usage of Host header for SNI for HTTP/3 - [Issue #57169](https://github.com/dotnet/runtime/issues/57169)
