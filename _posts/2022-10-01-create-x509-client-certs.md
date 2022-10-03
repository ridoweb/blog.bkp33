---
title: Create X509 client certificates
layout: post
date: 2022-10-01
---

When developing clients that use X509 certificates to authenticate to a server using TLS, you might need to create client certificates signed by your CA.

The _right_ way to create client certificates is by producing a Certificate Singing Request with `openssl`, send it to your CA, and process the response.

If you already have the private key of your CA you can simplify this flow by using the [`New-SelfSignedCertificate`](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate?view=windowsserver2022-ps) powershell cmdlet to create the signed certificate, and it will be added to your local certificate store. 

> Note: Despite the cmdlet, since it can be used to produced ca signed certificates, instead of self-signed.

The script below helps you with this process, and by using `openssl` though WSL it also produces the common `PEM/KEY` files to use the certificates in other environments where the certificate store might not be available.

## Load the CA certificate

Assuming you have the CA certificate already available in your `CurrentUser/My` certificate store, you can load it in a powershell session by its thumbprint:

```ps
$root = gci "cert:\CurrentUser\my\<CA-thumbprint>"
```

## Create your client certificate with a custom CN

```ps
$certName = "<Client-CN>"
$cert = New-SelfSignedCertificate `
        -CertStoreLocation cert:\CurrentUser\my `
        -Subject $certName `
        -Signer $root `
        -HashAlgorithm SHA256 `
        -NotAfter (Get-Date).AddMonths(24) `
        -KeyUsage KeyEncipherment, DataEncipherment, DigitalSignature, NonRepudiation
Write-Host $cert.Subject $cert.Thumbprint 
```

The certificate is now available in your local store (you can access the store by running `start certmgr.msc`)

## Export the public key to DER format

To export the public key certificate

```ps
Export-Certificate -Cert $cert -FilePath "$certname.bin.cer" -TYPE CERT
```

this file is a binary representation you can use in Windows machines, to convert the file to a PEM format use:

```cmd
certutil -encode "$certname.bin.cer" "$certname.pem"
```

## Export the private key as a PFX file

PFX files allows to export the private key, and protect they key with a password. In PowerShell you will require a `SecureString`

```ps
$keyPwd = ConvertTo-SecureString -String "<PFX-Password>" -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath "$certname.pfx" -Password $keyPwd
```

## Export the private key as a secure KEY file

`OpenSSL` allows to extract the PFX to a key file using the PEM format, and protect the key with another password. To automate the script you can use the `passin` arguments. Assuming you want to use the same password for both files the script will look like:

Note that OpenSSL does not support PowerShell secure strings, so the password must be converted to clear text by using the `PtrToStringAuto` method.

```ps
$txtPwd = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto([System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($keyPwd))
$bashCmd = "openssl pkcs12 -in '$certname.pfx' -out '$certname.secure.key' -nodes -passin pass:'$txtPwd' -passout pass:'$txtPwd'"
bash -c $bashCmd
```

### Export the private key as a RSA key

There are some cases where you might need the PEM/KEY files, but you don't want to protect the key with a password. Keep in mind that this option produces an **Unprotected Private Key*, so you must use it with caution:

```ps
$bashcmd = "openssl rsa -in '$certname.secure.key' -out '$certname.key'"
bash -c $bashCmd
```

## Summary

This script is a convenient tool to simplify your development process when you need to create client certificates signed by a custom CA without using the CSR flow.

The full script is available in this [gist](https://gist.github.com/ridomin/0ffb6a3c4c51eec7cdb020bedc8e0a5d)