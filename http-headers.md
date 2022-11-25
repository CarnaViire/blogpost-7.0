# HttpHeaders Read Thread Safety

The `HttpHeaders` collections were never thread-safe. Accessing a header may force lazy parsing of its value, resulting in modifications to the underlying data structures.

Before .NET 6, reading from the collection concurrently happened to be thread-safe in *most* cases.
If the collection contained an invalid header value, that value would be removed during enumeration. In rare cases, this could cause problems from modifying the underlying `Dictionary` concurrently.

Starting with .NET 6, less locking was performed around header parsing as it was no longer needed internally.
Due to this change, multiple examples of users accessing the headers concurrently surfaced, for example, in [gRPC](https://github.com/dotnet/runtime/issues/55898) and [NewRelic](https://github.com/newrelic/newrelic-dotnet-agent/issues/803).
Violating thread safety in .NET 6 may result in the header values being duplicated/malformed or various exceptions being thrown during enumeration/header accesses.

.NET 7 makes the header behavior more intuitive. `HttpHeaders` collection now satisfies the following:
- concurrent reads are thread-safe and return the same result to all readers ([#68115](https://github.com/dotnet/runtime/pull/68115))
- concurrent write/modification/remove operations are not supported and their behavior continues to be *undefined*
- a "validating read" of an invalid value does not result in the removal of the invalid value. The invalid value will still be visible to the [NonValidated](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.headers.httpheaders.nonvalidated?view=net-7.0#system-net-http-headers-httpheaders-nonvalidated) view of the collection ([#67833](https://github.com/dotnet/runtime/pull/67833), thanks [@heathbm](https://github.com/heathbm))