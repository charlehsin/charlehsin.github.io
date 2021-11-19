---
title:  "Use a certificate with correct private key and storage flags for TCP TLS in .NET 5."
date:   2021-11-18 08:00:00 -0800
categories: coding net5
permalinks: /:categories/:year/:month/:day/:title.html
---

.NET 5 provides great support to [create TCP listener and client in TCP TLS](https://docs.microsoft.com/en-us/dotnet/api/system.net.security.sslstream?view=net-6.0). Related sample codes can be found at my [GitHub repository](https://github.com/charlehsin/net5-crypto-tutorial). Specifically, check the implementation of the [these classes](https://github.com/charlehsin/net5-crypto-tutorial/tree/main/app/TcpOperations). Find how to use those classes at the implementation of the following method at [Program.cs](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/Program.cs).

{% highlight csharp linenos %}
private static void TryTcpOperationsWithTls(bool useMutualAuthentication);
{% endhighlight %}

The implementation of the method above creates the correct certificate for TCP TLS. For more details, see my [post]({% post_url 2021-11-18-create-correct-cert-for-store-in-net5 %}) about how to create the certificate. The created certificate has the private key with the persist key storage flag. This is important for TCP TLS. If your certificate has ephemeral private key, the TLS handshaking will fail.

For example, if the following codes are removed from the implementation of TryTcpOperationsWithTls method above, your certificate will have ephemeral private key.
{% highlight csharp linenos %}
// using var newCert = CertificateOperations.GetCertWithStorageFlags(cert,
//    X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet |
//    X509KeyStorageFlags.Exportable);
{% endhighlight %}

The TLS handshaking will fail with the following exception.

{% highlight console linenos %}
System.ComponentModel.Win32Exception (0x8009030E): No credentials are available in the security package
    at System.Net.SSPIWrapper.AcquireCredentialsHandle(ISSPIInterface secModule, String package, CredentialUse intent, SCH_CREDENTIALS* scc)
    at System.Net.Security.SslStreamPal.AcquireCredentialsHandle(CredentialUse credUsage, SCH_CREDENTIALS* secureCredential)
    at System.Net.Security.SslStreamPal.AcquireCredentialsHandleSchCredentials(X509Certificate certificate, SslProtocols protocols, EncryptionPolicy policy, Boolean isServer)
    at System.Net.Security.SslStreamPal.AcquireCredentialsHandle(SslStreamCertificateContext certificateContext, SslProtocols protocols, EncryptionPolicy policy, Boolean isServer)
    at System.Net.Security.SecureChannel.AcquireServerCredentials(Byte[]& thumbPrint)
    at System.Net.Security.SecureChannel.GenerateToken(ReadOnlySpan`1 inputBuffer, Byte[]& output)
    at System.Net.Security.SecureChannel.NextMessage(ReadOnlySpan`1 incomingBuffer)
    at System.Net.Security.SslStream.ProcessBlob(Int32 frameSize)
    at System.Net.Security.SslStream.ReceiveBlobAsync[TIOAdapter](TIOAdapter adapter)
    at System.Net.Security.SslStream.ForceAuthenticationAsync[TIOAdapter](TIOAdapter adapter, Boolean receiveFirst, Byte[] reAuthenticationData, Boolean isApm)
    at app.TcpOperations.MyTcpServer.ProcessClientAsync() in D:\Codes\net5-crypto-tutorial\app\TcpOperations\MyTcpServer.cs:line 207
{% endhighlight %}

Furthermore, since the TLS certificate is issued by our self-signed certificate, we need to use our custom certificate validation logic. Check ValidateServerCertificate method at [MyTcpClient class](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/TcpOperations/MyTcpClient.cs) for tutorial sample.