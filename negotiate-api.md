# Negotiate APIs

[Windows Authentication](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/windows-authentication-overview) is an encompassing term for multiple technologies used in enterprises to authenticate users and applications against a central authority, usually the domain controller. It enables scenarios like single sign-on to email services or intranet applications. The underlying technologies used for the authentication are Kerberos, NTLM, and the encompassing Negotiate protocol, where the most suitable technology is chosen for a specific authentication scenario.

Prior to .NET 7 the Windows Authentication was exposed in the high-level APIs, like `HttpClient` (`Negotiate` and `NTLM` authentication schemes), `SmtpClient` (`GSSAPI` and `NTLM` authentication schemes), `NegotiateStream`, [ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/windowsauth?view=aspnetcore-7.0), and the SQL Server client libraries. While that covers most scenarios for end users, it is limiting for library authors. Third-party libraries like the [Npgsql PostgreSQL client](https://www.npgsql.org/), [MailKit](https://github.com/jstedfast/MailKit), [Apache Kudu client](https://github.com/xqrzd/kudu-client-net) and others needed to resort to various tricks to implement the same authentication schemes for low-level protocols that were not built on HTTP or other available high-level building blocks.

In .NET 7, a new API is introduced that provides the low-level building block to allow library authors to perform the authentication exchange for the above protocols. As with all the other APIs in .NET it is built with cross-platform interoperability in mind. On Linux, macOS, iOS, and other similar platforms, it uses the GSSAPI system library. On Windows, it relies on the [SSPI](https://learn.microsoft.com/en-us/windows/win32/rpc/sspi-architectural-overview) library. For platforms where system implementation is not available, such as Android and tvOS, a limited client-only implementation is present.

## How to use the API

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

The authentication starts with the client producing a challenge token. Then the server produces a response. The client processes the response, and a new challenge is sent to the server. This challenge/response exchange could happen multiple times. It finishes when either party rejects the authentication or when both parties accept the authentication. The format of the tokens is defined by the Windows Authentication protocols, while the encapsulation is part of the high-level protocol specification. In this example, the SMTP protocol prepends the `334` code to tell the client that the server produced an authentication response, and the `235` code indicates successful authentication.

The bulk of the new API centers around a new `NegotiateAuthentication` class. It is used to instantiate the context for client-side or server-side authentication. There are various options that can specify requirements for establishing the authenticated session, like requiring encryption or determining that a specific protocol (Negotiate, Kerberos, or NTLM) is to be used. Once the parameters are specified, the authentication proceeds by exchanging the authentication challenges/responses between the client and the server. The `GetOutgoingBlob` method is used for that. It can work either on byte spans or base64-encoded strings. 

The following code will perform both the client and server part of the authentication of the current user on the same machine:
```csharp
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
