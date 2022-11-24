# WebSockets improvements

## Upgrade Response Details

Prior to .NET 7, WebSocket's HTTP upgrade response details were hidden inside `ClientWebSocket` implementation, and all connect errors would surface as `WebSocketException` without much details beside the exception message. However, the information about response headers and status code might be important in both failure and success scenarios.

In case of failure, HTTP status code can help to distinguish between retriable and non-retriable errors (e.g. server doesn't support WebSockets at all, or it was just a small transient error). Headers might also contain additional information on how to handle the situation. The headers are also useful even in case of a success WebSocket connect, e.g. they can contain a token tied to a session, or some info related to subprotocol version, or that the server can go down soon, etc.

.NET 7 adds a setting [ClientWebSocketOptions.CollectHttpResponseDetails](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.collecthttpresponsedetails?view=net-7.0#system-net-websockets-clientwebsocketoptions-collecthttpresponsedetails) that enables collecting upgrade response details in `ClientWebSocket` instance during `ClientWebSocket.ConnectAsync` call. You can later access the data using [ClientWebSocket.HttpStatusCode](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.httpstatuscode?view=net-7.0) and [ClientWebSocket.HttpResponseHeaders](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.httpresponseheaders?view=net-7.0) properties, even in case of `ClientWebSocket.ConnectAsync` throwing an exception. Note that in the exception case, the information might be unavailable, e.g. if the server never responded to the request.

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

In a default case, `ClientWebSocket` would use a cached static `HttpMessageInvoker` instance to execute HTTP Upgrade requests. However, if options such as `ClientWebSocketOptions.Proxy`, `ClientWebSocketOptions.ClientCertificates` or `ClientWebSocketOptions.Cookies` are passed, `HttpMessageInvoker`s with such parameters are not safe to be reused, so they would be created each time per `ClientWebSocket.ConnectAsync` call. This results in more allocations that could be necessary and makes potential reuse of `HttpMessageInvoker` instances and established HTTP connections impossible.

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

.NET 7 also adds an ability to use WebSocket protocol over HTTP/2, as described in [RFC 8441](https://www.rfc-editor.org/rfc/rfc8441). With that, WebSocket connection would be established over a single stream of HTTP/2 connection. This will allow for a single TCP connection to be shared between two protocols and to be used for several WebSocket connections and HTTP requests at the same time, resulting in more efficient use of the network.

To enable WebSockets over HTTP/2, you can set [ClientWebSocketOptions.HttpVersion](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.httpversion?view=net-7.0) option to `HttpVersion.Version20`. You can also enable upgrade/downgrade of HTTP version used by setting [ClientWebSocketOptions.HttpVersionPolicy](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocketoptions.httpversionpolicy?view=net-7.0) property. These options will behave in the same way `HttpRequestMessage.Version` and `HttpRequestMessage.VersionPolicy` behave.

For example, the following code would probe for HTTP/2 WebSockets, and if a WebSocket connection could not be established, it will fallback to HTTP/1.1:

```c#
var ws = new ClientWebSocket();
ws.Options.HttpVersion = HttpVersion.Version20;
ws.Options.HttpVersionPolicy = HttpVersionPolicy.RequestVersionOrLower;
await ws.ConnectAsync(uri, httpClient, cancellationToken);
```

The combination of `HttpVersion.Version11` and `HttpVersionPolicy.RequestVersionOrHigher` will result in the same behavior as above, while `HttpVersionPolicy.RequestVersionExact` will disallow upgrade/downgrade of HTTP version.

By default, `HttpVersion = HttpVersion.Version11` and `HttpVersionPolicy = HttpVersionPolicy.RequestVersionOrLower`, which means that by default only HTTP/1.1 will be used.

Note that, because the ability to multiplex WebSocket connections (and HTTP requests) over a single HTTP/2 connection is the crucial part of the feature, you can *only* use it when you pass an `HttpClient` of `HttpMessageInvoker` instance from your code (i.e. use [ClientWebSocket.ConnectAsync(Uri, HttpMessageInvoker, CancellationToken)](https://learn.microsoft.com/en-us/dotnet/api/system.net.websockets.clientwebsocket.connectasync?view=net-7.0#system-net-websockets-clientwebsocket-connectasync(system-uri-system-net-http-httpmessageinvoker-system-threading-cancellationtoken)) overload), so that the connection pool within this `HttpClient` of `HttpMessageInvoker` instance could be used for multiplexing.