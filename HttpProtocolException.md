# Detect HTTP/2 and HTTP/3 Protocol Errors

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
