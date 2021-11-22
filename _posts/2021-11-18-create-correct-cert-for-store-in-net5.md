---
title                    : "Create and add certificate to certificate store in .NET 5."
date                     : 2021-11-18 06:08:00 -0800
last_modified_at         : 2021-11-22 07:00:00 -0800
categories               : Coding DotNet5
permalinks               : /:categories/:year/:month/:day/:title.html
header:
  teaser                 : /assets/images/teaser-create-cert.jpg
---

This post talks about how to issue certificate with the private key and with the persist storage flag before adding it to the certificate store.

.NET 5 supports for [creating a self-signed certificate](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.certificaterequest.createselfsigned?view=net-5.0) and for [issuing a certificate](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.certificaterequest.create?view=net-5.0). Before the issued certificate is added to the certificate store, extra steps are needed to get the desired private key and storage flag settings.

First, let's create a certificate with which we can issue other certificates. In the sample codes, we create a self-signed certificate by using CertificateRequest. You can also load the existing certificate (with private key).

{% highlight csharp linenos %}
/// <summary>
/// Create a self-signed certificate.
/// </summary>
/// <param name="keySize">The RAS key size in bits.</param>
/// <param name="commonName">The certificate common name.</param>
/// <param name="notBefore">The certificate starting time.</param>
/// <param name="notAfter">The certificate expiration time.</param>
/// <returns>The X509Certificate2 certificate.</returns>
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

Then we use this self-signed certificate to issue a leaf certificate in the following sample codes.

{% highlight csharp linenos %}
/// <summary>
/// Issue a signed certificate by the parent cert.
/// </summary>
/// <param name="parentCert">The certificate used to sign this certificate.</param>
/// <param name="keySize">The RAS key size in bits.</param>
/// <param name="commonName">The certificate common name.</param>
/// <param name="flags">The certificate key usage flags.</param>
/// <param name="oidCollection">The enhanced key usages.</param>
/// <param name="notBefore">The certificate starting time.</param>
/// <param name="notAfter">The certificate expiration time.</param>
/// <param name="includePrivateKey">True to include the private key in the returned object.</param>
/// <returns>The X509Certificate2 certificate.</returns>
public static X509Certificate2 IssueSignedCert(X509Certificate2 parentCert, int keySize, string commonName,
    X509KeyUsageFlags flags, OidCollection oidCollection,
    System.DateTimeOffset notBefore, System.DateTimeOffset notAfter,
    bool includePrivateKey)
{
    using var rsa = RSA.Create(keySize);
    var request = new CertificateRequest(
        $"CN={commonName}",
        rsa,
        HashAlgorithmName.SHA512,
        RSASignaturePadding.Pkcs1);

    request.CertificateExtensions.Add(new X509BasicConstraintsExtension(
        false/*certificateAuthority*/,
        false/*hasPathLengthConstraint*/,
        0/*pathLengthConstraint*/,
        false/*critical*/));

    request.CertificateExtensions.Add(new X509KeyUsageExtension(
        flags/*keyUsages*/,
        false/*critical*/));

    request.CertificateExtensions.Add(new X509EnhancedKeyUsageExtension(
        oidCollection/*oidCollection*/,
        true/*critical*/));

    request.CertificateExtensions.Add(new X509SubjectKeyIdentifierExtension(
        request.PublicKey/*subjectKeyIdentifier*/,
        false/*critical*/));

    var subjectAlternativeNameBuilder = new SubjectAlternativeNameBuilder();
    subjectAlternativeNameBuilder.AddDnsName("test.com");
    subjectAlternativeNameBuilder.AddIpAddress(IPAddress.Loopback);
    request.CertificateExtensions.Add(subjectAlternativeNameBuilder.Build());

    var serialNumber = new byte[SerialNumberSizeInBytes];
    RandomNumberGenerator.Fill(serialNumber);
    var cert = request.Create(parentCert, notBefore, notAfter, serialNumber);
    if (!includePrivateKey)
    {
        return cert;
    }

    var certWithPrivateKey = cert.CopyWithPrivateKey(rsa);
    cert.Dispose();
    return certWithPrivateKey;
}
{% endhighlight %}

**Watch out!** CertificateRequest.Create method returns a X509Certificate2 with HasPrivateKey = false. To get the private key, X509Certificate2.CopyWithPrivateKey method is used. If X509Certificate2.CopyWithPrivateKey is not used, the X509Certificate2 object does not have the private key.
{: .notice--info}

At this point, the X509Certificate2 object still has a ephemeral private key. .NET so far does not let you set the private key storage flag yet. We need to get a new X509Certificate2 object with the correct private key storage flag.

{% highlight csharp linenos %}
/// <summary>
/// Get a certificate with the target key storage flags based on the original cert.
/// </summary>
/// <param name="cert">The original certificate.</param>
/// <param name="flags">The target key storage flags.</param>
/// <returns>The new  X509Certificate2 certificate.</returns>
public static X509Certificate2 GetCertWithStorageFlags(X509Certificate2 cert, X509KeyStorageFlags flags)
{
    return new X509Certificate2(
        cert.Export(X509ContentType.Pkcs12),
        string.Empty, flags
    );
}
{% endhighlight %}

Combining all of the above together, the following sample codes issue a certificate with the private key and with the PersisKeySet storage flag for TCP TLS.

{% highlight csharp linenos %}
var notBefore = DateTimeOffset.UtcNow.AddDays(-45);
var notAfter = DateTimeOffset.UtcNow.AddDays(365);
// Create a self-signed certificate.
using var rootCert = CertificateOperations.CreateSelfSignedCert(
    CertificateOperations.KeySizeInBits, "A test root",
    notBefore, notAfter);
// Use the self-signed certificate to issue a TCP TLS certificate with the private key.
// This certificate has Oid for both TCP server and TCP client.
using var cert = CertificateOperations.IssueSignedCert(rootCert,
    CertificateOperations.KeySizeInBits, "A test TLS cert",
    X509KeyUsageFlags.DigitalSignature | X509KeyUsageFlags.NonRepudiation |
    X509KeyUsageFlags.KeyEncipherment,
    new OidCollection
    {
        new Oid("1.3.6.1.5.5.7.3.1")/*id-kp-serverAuth*/,
        new Oid("1.3.6.1.5.5.7.3.2")/*id-kp-clientAuth*/
    },
    notBefore, notAfter, true);
// Get the certificate with the desired key storage flags.
using var newCert = CertificateOperations.GetCertWithStorageFlags(cert,
    X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet |
    X509KeyStorageFlags.Exportable);
{% endhighlight %}

See the complete sample codes at my [GitHub repository](https://github.com/charlehsin/net5-crypto-tutorial), specifically at [CertificateOperations.cs](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/Certificates/CertificateOperations.cs).

