# Improved handling of failing connection attempts

In versions prior to .NET 6, in case there is no connection immediately available in the connection pool, a new HTTP request will always spin up a new connection attempt and wait for it. The downside to this is, if it takes a while to establish that connection and another connection becomes available in the meantime, that request would continue waiting for the connection it spawned, hurting latency. In .NET 6.0 [we changed this](https://devblogs.microsoft.com/dotnet/dotnet-6-networking-improvements/#other-http-changes) to process requests on whichever connection that becomes available first, whether that’s a newly established one or one that became ready to handle the request in the meantime.

Unfortunately, the .NET 6.0 implementation turned out to be problematic for some users: a failing connection attempt also fails the request on the top of the request queue, which  may lead to unexpected request failures in [certain scenarios](https://github.com/dotnet/runtime/issues/60654#issue-1030717005). Moreover, if there is a pending connection in the pool that will never become usable eg. because of a misbehaving server or a network issue, new incoming requests associated with it will also stall and may fail unexpectedly.

In .NET 7.0 we implemented the following changes to address these issues:

- A failing connection attempt can only fail it's initiating request, and never an unrelated one. If the original request has been handled by the time a connection fails, the connection failure is ignored ([dotnet/runtime#62935](https://github.com/dotnet/runtime/pull/62935)).
- If a request initiates a new connection, but then becomes handled by another connection from the pool, the new pending connection attempt will time out after a short period, regardless of `ConnectTimeout`. With this change, stalled connections will not stall unrelated requests ([dotnet/runtime#71785](https://github.com/dotnet/runtime/pull/71785)).
