# WebSockets

## Upgrade Response Details

Prior to .NET 7, WebSocket's HTTP upgrade response details were hidden inside `ClientWebSocket` implementation, and all connect errors would surface as `WebSocketException` without much details beside the exception message. However, the information about response headers and status code might be important in both failure and success scenarios.

In case of failure, HTTP status code can help to distinguish between retriable and non-retriable errors (e.g. server doesn't support WebSockets at all, or it was just a transient error). Headers might also contain additional information on how to handle the situation. The headers are also useful even in case of a successful WebSocket connect, e.g. they can contain token tied to a session, information related to subprotocol version, or that the server can go down soon.

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