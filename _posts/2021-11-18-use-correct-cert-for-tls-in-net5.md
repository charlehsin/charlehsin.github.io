---
title                    : "Valid certificate for TCP TLS in .NET 5."
date                     : 2021-11-18 08:00:00 -0800
last_modified_at         : 2021-11-22 08:00:00 -0800
categories               : Coding DotNet5
permalinks               : /:categories/:year/:month/:day/:title.html
header:
  teaser                 : /assets/images/teaser-tcptls-cred-error.jpg
---

This post discusses how to create valid certificate for TCP TLS.

.NET 5 supports for [creating TCP listener and client in TCP TLS](https://docs.microsoft.com/en-us/dotnet/api/system.net.security.sslstream?view=net-6.0). Related sample codes can be found at [these classes](https://github.com/charlehsin/net5-crypto-tutorial/tree/main/app/TcpOperations) in my [GitHub repository](https://github.com/charlehsin/net5-crypto-tutorial). Find how to use those classes at the TryTcpOperationsWithTls method at [Program.cs](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/Program.cs).

The implementation of the method above creates the correct certificate for TCP TLS. For more details, see my [post]({% post_url 2021-11-18-create-correct-cert-for-store-in-net5 %}). The created certificate has the private key with the persist key storage flag. This is important for TCP TLS. If your certificate has ephemeral private key, the TLS handshaking will fail.

For example, if we comment out the CertificateOperations.GetCertWithStorageFlags call in the implementation of TryTcpOperationsWithTls method below, the certificate will have ephemeral private key.

{% highlight csharp linenos %}
/// <summary>
/// Try TCP operations with TLS.
/// </summary>
/// <param name="useMutualAuthentication">True if the client certificate is required.</param>
private static void TryTcpOperationsWithTls(bool useMutualAuthentication)
{
    // Create TLS certificates.
    // For tutorial purpose, we use the same certs for server and for client.
    var notBefore = DateTimeOffset.UtcNow.AddDays(-45);
    var notAfter = DateTimeOffset.UtcNow.AddDays(365);
    using var rootCert = CertificateOperations.CreateSelfSignedCert(CertificateOperations.KeySizeInBits, "A test root",
        notBefore, notAfter);
    using var cert = CertificateOperations.IssueSignedCert(rootCert, CertificateOperations.KeySizeInBits, "A test TLS cert",
        X509KeyUsageFlags.DigitalSignature | X509KeyUsageFlags.NonRepudiation | X509KeyUsageFlags.KeyEncipherment,
        new OidCollection
        {
            new Oid("1.3.6.1.5.5.7.3.1")/*id-kp-serverAuth*/,
            new Oid("1.3.6.1.5.5.7.3.2")/*id-kp-clientAuth*/
        },
        notBefore, notAfter, true);
    // For TLS, we cannot use ephemeral key, so we need to set the storage flag correctly.
    //using var newCert = CertificateOperations.GetCertWithStorageFlags(cert,
    //        X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet | X509KeyStorageFlags.Exportable);

    TryTcpOperations(newCert, rootCert, useMutualAuthentication ? newCert : null, useMutualAuthentication ? rootCert : null);
}
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

**Watch out!** Since the TLS certificate is issued by our self-signed certificate, we need to use our custom certificate validation logic in TLS handshaking. Check ValidateServerCertificate method at [MyTcpClient class](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/TcpOperations/MyTcpClient.cs) for tutorial sample.
{: .notice--info}