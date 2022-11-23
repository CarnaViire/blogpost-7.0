# Detect HTTP/2.0 and HTTP/3.0 protocol errors

HTTP2/2 and HTTP/3 defines protocol-level error codes in [RFC 7540 section 7](https://www.rfc-editor.org/rfc/rfc7540#section-7) and [RFC 9114 section 8.1](https://www.rfc-editor.org/rfc/rfc9114.html#section-8.1) (not to be confused with HTTP status codes!). Such low-level error information is unimportant for most `HttpClient` users, however it might be helpful in advanced HTTP/2 or HTTP/3 scenarios, notably grpc-dotnet, where distinguishing protocol errors is vital to implement client retries.

This is now possible by filtering for [`HttpProtocolException`](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpprotocolexception) when calling `HttpClient`:

```csharp
try
{
    var response = await myHttpClientInstance.GetStringAsync(".");
}
catch (HttpRequestException ex) when (ex.InnerException is HttpProtocolException pex)
{
    Console.WriteLine("HTTP error code: " + pex.ErrorCode)
}
```

Or by catching it directly when working with `HttpContent`'s response stream:

```csharp
try
{
    await responseStream.ReadAsync(buffer);
}
catch (HttpProtocolException pex)
{
    Console.WriteLine("HTTP error code: " + pex.ErrorCode)
}
```