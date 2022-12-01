# HttpHeaders Read Thread Safety

The `HttpHeaders` collections were never thread-safe. Accessing a header may force lazy parsing of its value, resulting in modifications to the underlying data structures.

Before .NET 6, reading from the collection concurrently happened to be thread-safe in *most* cases.

Starting with .NET 6, less locking was performed around header parsing as it was no longer needed internally.
Due to this change, multiple examples of users accessing the headers concurrently surfaced, for example, in [gRPC](https://github.com/dotnet/runtime/issues/55898), [NewRelic](https://github.com/newrelic/newrelic-dotnet-agent/issues/803), or even [HttpClient itself](https://github.com/dotnet/runtime/issues/65379).
Violating thread safety in .NET 6 may result in the header values being duplicated/malformed or various exceptions being thrown during enumeration/header accesses.

.NET 7 makes the header behavior more intuitive. The `HttpHeaders` collection now matches the thread-safety guarantees of a `Dictionary`:
> The collection can support multiple readers concurrently, as long as it is not modified. In the rare case where an enumeration contends with write accesses, the collection must be locked during the entire enumeration. To allow the collection to be accessed by multiple threads for reading and writing, you must implement your own synchronization.