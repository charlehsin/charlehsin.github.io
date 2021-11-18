---
title:  "Use a certificate with correct private key and storage flags for TCP TLS in .NET 5."
date:   2021-11-18 08:00:00 -0800
categories: coding net5
permalinks: /:categories/:year/:month/:day/:title.html
---

.NET 5 provides great support to [create a self-signed certificate](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.certificaterequest.createselfsigned?view=net-5.0), to [issue a certificate](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.certificaterequest.create?view=net-5.0), and to [create TCP listener and client in TCP TLS](https://docs.microsoft.com/en-us/dotnet/api/system.net.security.sslstream?view=net-6.0). Related sample codes can be found at my [GitHub repository](https://github.com/charlehsin/net5-crypto-tutorial). However, if the issed certificate does not have correct private key and storage flag settings, then the TCP TLS will not work.

First, let's create a self-signed certificate by using CertificateRequest.
{% highlight csharp linenos %}
public static X509Certificate2 CreateSelfSignedCert(int keySize, string commonName,
    System.DateTimeOffset notBefore, System.DateTimeOffset notAfter)
{
    using var rsa = RSA.Create(keySize);
    var request = new CertificateRequest(
        $"CN={commonName}",
        rsa,
        HashAlgorithmName.SHA512,
        RSASignaturePadding.Pkcs1);

    request.CertificateExtensions.Add(new X509BasicConstraintsExtension(
        true/*certificateAuthority*/,
        false/*hasPathLengthConstraint*/,
        0/*pathLengthConstraint*/,
        true/*critical*/));

    request.CertificateExtensions.Add(new X509KeyUsageExtension(
        X509KeyUsageFlags.KeyCertSign | X509KeyUsageFlags.CrlSign/*keyUsages*/,
        false/*critical*/));

    request.CertificateExtensions.Add(new X509SubjectKeyIdentifierExtension(
        request.PublicKey/*subjectKeyIdentifier*/,
        false/*critical*/));

    var subjectAlternativeNameBuilder = new SubjectAlternativeNameBuilder();
    subjectAlternativeNameBuilder.AddDnsName("test.com");
    subjectAlternativeNameBuilder.AddIpAddress(IPAddress.Loopback);
    request.CertificateExtensions.Add(subjectAlternativeNameBuilder.Build());

    return request.CreateSelfSigned(notBefore, notAfter);
}
{% endhighlight %}

Then we will use this self-signed certificate to issue a leaft certificate.