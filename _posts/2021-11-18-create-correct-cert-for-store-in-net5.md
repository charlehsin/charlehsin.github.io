---
title:  "Create a certificate with correct private key and storage flags to be added to certificate store in .NET 5."
date:   2021-11-18 06:08:00 -0800
categories: coding net5
permalinks: /:categories/:year/:month/:day/:title.html
---

.NET 5 provides great support to [create a self-signed certificate](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.certificaterequest.createselfsigned?view=net-5.0) and to [issue a certificate](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.certificaterequest.create?view=net-5.0). Related sample codes can be found at my [GitHub repository](https://github.com/charlehsin/net5-crypto-tutorial). The issued certificate by .NET 5 by default may not have the desired private key and storage flag settings to be added into the certificate store.  This post talks about how to make sure that the issued certificate have the private key and have the persistent storage flag before it is added to the certificate store.

First, let's create a self-signed certificate by using CertificateRequest. Please check the implementation of the following method at [CertificateOperations class](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/Certificates/CertificateOperations.cs). 

{% highlight csharp linenos %}
public static X509Certificate2 CreateSelfSignedCert(int keySize, string commonName,
    System.DateTimeOffset notBefore, System.DateTimeOffset notAfter);
{% endhighlight %}

Then we will use this self-signed certificate to issue a leaf certificate. Please check the implementation of the following method at [CertificateOperations class](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/Certificates/CertificateOperations.cs). In the implementation, CertificateRequest.Create method returns a X509Certificate2 with HasPrivateKey = false. To get the private key, X509Certificate2.CopyWithPrivateKey method is used.

{% highlight csharp linenos %}
public static X509Certificate2 IssueSignedCert(X509Certificate2 parentCert, 
    int keySize, string commonName,
    X509KeyUsageFlags flags, OidCollection oidCollection,
    System.DateTimeOffset notBefore, System.DateTimeOffset notAfter,
    bool includePrivateKey);
{% endhighlight %}

At this point, the X509Certificate2 object still has a ephemeral private key. So we need to get a new X509Certificate2 object with the correct private key storage flag. Please check the implementation of the following method at [CertificateOperations class](https://github.com/charlehsin/net5-crypto-tutorial/blob/main/app/Certificates/CertificateOperations.cs).

{% highlight csharp linenos %}
public static X509Certificate2 GetCertWithStorageFlags(X509Certificate2 cert,
    X509KeyStorageFlags flags);
{% endhighlight %}

Combining all of the above together, the following sample code block issues a certificate with the private key and with the PersisKeySet storage flag for TCP TLS.

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

