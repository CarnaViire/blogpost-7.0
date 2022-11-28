# Security

## Negotiate API
[Windows Authentication](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-overview) is an encompassing term for multiple technologies used in enterprises to authenticate users and applications against a central authority, usually the domain controller. It enables scenarios like single sign-on to email services or intranet applications. The underlying technologies used for the authentication are Kerberos, NTLM, and the encompassing Negotiate protocol, where the most suitable technology is chosen for a specific authentication scenario.

Prior to .NET 7, Windows Authentication was exposed in the high-level APIs, like `HttpClient` (`Negotiate` and `NTLM` authentication schemes), `SmtpClient` (`GSSAPI` and `NTLM` authentication schemes), `NegotiateStream`, [ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/windowsauth?view=aspnetcore-7.0), and the SQL Server client libraries. While that covers most scenarios for end users, it is limiting for library authors. Third-party libraries like the [Npgsql PostgreSQL client](https://www.npgsql.org/), [MailKit](https://github.com/jstedfast/MailKit), [Apache Kudu client](https://github.com/xqrzd/kudu-client-net) and others needed to resort to various tricks to implement the same authentication schemes for low-level protocols that were not built on HTTP or other available high-level building blocks.

.NET 7 introduces new API providing low-level building blocks to perform the authentication exchange for the above mentioned protocols, see [Issue #69920](https://github.com/dotnet/runtime/issues/69920). As with all the other APIs in .NET, it is built with cross-platform interoperability in mind. On Linux, macOS, iOS, and other similar platforms, it uses the GSSAPI system library. On Windows, it relies on the [SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/sspi-architectural-overview) library. For platforms where system implementation is not available, such as Android and tvOS, a limited client-only implementation is present.

### How to use the API

In order to understand how the authentication API works, let's start with an example of how the authentication session looks in a high-level protocol like SMTP. The example is taken from [Microsoft protocol documentation](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-ssean/0fa15a99-af88-428a-a51d-742b8450d877) that explains it in greater detail.

```
S: 220 server.contoso.com Authenticated Receive Connector
C: EHLO client.contoso.com
S: 250-server-contoso.com Hello [203.0.113.1]
S: 250-AUTH GSSAPI NTLM
S: 250 OK
C: AUTH GSSAPI <token1>
S: 334 <token2>
C: <token3>
S: 235 2.7.0 Authentication successful
```

The authentication starts with the client producing a challenge token. Then the server produces a response. The client processes the response, and a new challenge is sent to the server. This challenge/response exchange can happen multiple times. It finishes when either party rejects the authentication or when both parties accept the authentication. The format of the tokens is defined by the Windows Authentication protocols, while the encapsulation is part of the high-level protocol specification. In this example, the SMTP protocol prepends the `334` code to tell the client that the server produced an authentication response, and the `235` code indicates successful authentication.

The bulk of the new API centers around a new [`NegotiateAuthentication`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication?view=net-7.0) class. It is used to instantiate the context for client-side or server-side authentication. There are various options to specify requirements for establishing the authenticated session, like requiring encryption or determining that a specific protocol (Negotiate, Kerberos, or NTLM) is to be used. Once the parameters are specified, the authentication proceeds by exchanging the authentication challenges/responses between the client and the server. The [`GetOutgoingBlob`](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication.getoutgoingblob?view=net-7.0) method is used for that. It can work either on byte spans or base64-encoded strings.

The following code will perform both, client and server, part of the authentication for the current user on the same machine:

```C#
using System.Net;
using System.Net.Security;

var serverAuthentication = new NegotiateAuthentication(new NegotiateAuthenticationServerOptions { });
var clientAuthentication = new NegotiateAuthentication(
    new NegotiateAuthenticationClientOptions
    {
        Package = "Negotiate",
        Credential = CredentialCache.DefaultNetworkCredentials,
        TargetName = "HTTP/localhost",
        RequiredProtectionLevel = ProtectionLevel.Sign
    });

string? serverBlob = null;
while (!clientAuthentication.IsAuthenticated)
{
    // Client produces the authentication challenge, or response to server's challenge
    string? clientBlob = clientAuthentication.GetOutgoingBlob(serverBlob, out var clientStatusCode);
    if (clientStatusCode == NegotiateAuthenticationStatusCode.ContinueNeeded)
    {
        // Send the client blob to the server; this would normally happen over a network
        Console.WriteLine($"C: {clientBlob}");
        serverBlob = serverAuthentication.GetOutgoingBlob(clientBlob, out var serverStatusCode);
        if (serverStatusCode != NegotiateAuthenticationStatusCode.Completed &&
            serverStatusCode != NegotiateAuthenticationStatusCode.ContinueNeeded)
        {
            Console.WriteLine($"Server authentication failed with status code {serverStatusCode}");
            break;
        }
        Console.WriteLine($"S: {serverBlob}");
    }
    else
    {
        Console.WriteLine(
            clientStatusCode == NegotiateAuthenticationStatusCode.Completed ?
            "Successfully authenticated" :
            $"Authentication failed with status code {clientStatusCode}");
        break;
    }
}
```

Once the authenticated session is established, the `NegotiateAuthentication` instance can be used to sign/encrypt the outgoing messages and verify/decrypt the incoming messages. This is done through the [Wrap](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication.wrap?view=net-7.0) and [Unwrap](https://learn.microsoft.com/en-us/dotnet/api/system.net.security.negotiateauthentication.unwrap?view=net-7.0) methods.

This change was done as well as this part of the article was written by a community contributor [@filipnavara](https://github.com/filipnavara).


## Performance

Most of the networking perfomance improvements in .NET 7 are covered by Stephen's article [Performance Improvements in .NET 7 - Networking](https://devblogs.microsoft.com/dotnet/performance_improvements_in_net_7/#networking), but some of the security ones are worth mentioning again.

### TLS Resume

Establishing new TLS connection is fairly expensive operation as it requires multiple steps and several roundtrips. In scenarios where connection to the same server is re-created very often, time consumed by the handshakes will add up. TLS offers feature to mitigate this called Session Resumption, see [RFC 5246 - 7.3.  Handshake Protocol Overview](https://www.rfc-editor.org/rfc/rfc5246.html#section-7.3) and [RFC 8446 - 2.2.  Resumption and Pre-Shared Key](https://www.rfc-editor.org/rfc/rfc8446#section-2.2). Even though the mechanics differ for different TLS versions, the end goal is the same, save a round-trip and some CPU time when re-establishing connection to a previously connected server. This feature is automatically provided by SChannel on Windows, but with OpenSSL on Linux it required several changes to enable this:
- Server side (stateless): [PR #57079](https://github.com/dotnet/runtime/pull/57079) and [PR #63030](https://github.com/dotnet/runtime/pull/63030)
- Client side: [PR #64369](https://github.com/dotnet/runtime/pull/64369)
- Cache size control: [PR #69065](https://github.com/dotnet/runtime/pull/69065)

These changes bring .NET 7 Linux support of TLS resume on par with Windows capabilities.

### OCSP Stapling

Online Certificate Status Protocol (OCSP) Stapling is a mechanism for server to provide signed and timestamped proof (OCSP response) that the sent certificate has not been revoked, see [RFC 6961](https://www.rfc-editor.org/rfc/rfc6961). As a result, client doesn't need to contact the OCSP server itself, reducing the number of requests necessary to establish the connection as well as the load exerted on the OCSP server. And as the OCSP response needs to be signed by the certificate authority (CA), it cannot forged by the server providing the certififcate. We didn't take advantage of this TLS feature up until this release, for more details see [Issue #33377](https://github.com/dotnet/runtime/issues/33377).




---
https://learn.microsoft.com/en-us/dotnet/core/compatibility/networking/7.0/allowrenegotiation-default