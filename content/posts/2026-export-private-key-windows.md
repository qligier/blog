---
title: "Exporting a non-exportable private key from the Windows Certificate Store"
date: 2026-09-20
draft: false
tags: ['Tip', 'Software', 'PowerShell']
description: "The blog post is describing how I exported a client certificate's private key that the Windows 
certificate store had marked as non-exportable, once I realised the key was CNG-backed, not CAPI-backed."
---

I recently switched laptops while traveling, and I had to migrate a client certificate, used for mTLS, that I had 
saved in the Windows Certificate Store.
Of course, when I imported it initially, I didn't select "Mark this key as exportable", and Windows was (logically)
refusing to let me export it now.
While I could have waited to get back home, where I stored an offline copy of the private key, I was ready to spend 
some time to find if there was a way to do it sooner.

After a quick Kagi search, I found many solutions, especially on the usual knowledge sources: Stack Overflow and Super
User.
After excluding the stupid responses, I tried the remaining solutions: [mimikatz](https://github.com/gentilkiwi/mimikatz)/
[mimicertz](https://github.com/maaaaz/mimicertz), [exportrsa](https://github.com/luipir/ExportNotExportablePrivateKey),
[looking for the private key in the registry](https://www.yuenx.com/2022/certificate-security-export-cert-with-non-exportable-private-key-marked-as-not-exportable-windows-pki/).
Nothing worked.

It turns out my certificate's private key was not
[CAPI](https://learn.microsoft.com/en-us/windows/win32/seccng/cng-portal)-backed, but CNG-backed, which is the newer stack 
("Cryptography API: Next Generation").
If you want to check which stack a certificate uses, `certutil -store -user My` lists the provider for each one.

CNG is supported (in theory) by `exportrsa`, and the
[accompanying paper](https://github.com/luipir/ExportNotExportablePrivateKey/blob/master/doc/exporting_non-exportable_rsa_keys.pdf)
describes how to modify the flag in-memory and which API to call to export the key.

With a bit of help from Claude, I ended up with the following
[PowerShell script](https://gist.github.com/qligier/402689d2c30fade56c8752f7b9ee57b3) that does just the same:
list the certificates with a private-key, present them to the user for selection, load the private key in memory, 
set the export policy to `NCRYPT_ALLOW_EXPORT_FLAG | NCRYPT_ALLOW_PLAINTEXT_EXPORT_FLAG` and export it as PKCS#8.
It worked for my private key, mission accomplished!

```powershell {title="Exporting a CNG-backed private key as PKCS#8 PEM"}
# Prints the private key of a certificate from the current user's store as an
# unencrypted PEM (PKCS#8). The key must be CNG-backed.

# Add-Type compiles inline C# with [DllImport] P/Invoke stubs, exposing native
# Win32 ncrypt.dll (CNG) calls to PowerShell.
Add-Type @"
using System;
using System.Runtime.InteropServices;
public class NCrypt {
    // https://learn.microsoft.com/en-us/windows/win32/api/ncrypt/nf-ncrypt-ncryptsetproperty
    [DllImport("ncrypt.dll", CharSet = CharSet.Unicode)]
    public static extern int NCryptSetProperty(IntPtr hObject, string pszProperty,
        byte[] pbInput, int cbInput, int dwFlags);

    // https://learn.microsoft.com/en-us/windows/win32/api/ncrypt/nf-ncrypt-ncryptexportkey
    [DllImport("ncrypt.dll", CharSet = CharSet.Unicode)]
    public static extern int NCryptExportKey(
        IntPtr hKey, IntPtr hExportKey, string pszBlobType,
        IntPtr pParameterList, byte[] pbOutput, int cbOutput,
        out int pcbResult, int dwFlags);

    public const string EXPORT_POLICY = "Export Policy";
    public const string PKCS8_BLOB    = "PKCS8_PRIVATEKEY";
    public static readonly byte[] ALLOW = BitConverter.GetBytes(3); // ALLOW_EXPORT | ALLOW_PLAINTEXT_EXPORT
}
"@

function ConvertToPem {
    param([byte[]]$Der)
    $b64 = [Convert]::ToBase64String($Der)
    $lines = for ($i = 0; $i -lt $b64.Length; $i += 64) {
        $b64.Substring($i, [Math]::Min(64, $b64.Length - $i))
    }
    "-----BEGIN PRIVATE KEY-----`n" + ($lines -join "`n") + "`n-----END PRIVATE KEY-----`n"
}

# Pick a certificate that has a private key
$cert = Get-ChildItem Cert:\CurrentUser\My |
        Where-Object { $_.HasPrivateKey } |
        Select-Object Subject, Thumbprint |
        Out-GridView -PassThru | # This will show all certificates found, asking the user to select one
        ForEach-Object { Get-ChildItem "Cert:\CurrentUser\My\$($_.Thumbprint)" }

if (-not $cert) { throw 'No certificate selected.' }

$cngKey = [System.Security.Cryptography.X509Certificates.RSACertificateExtensions]::GetRSAPrivateKey($cert)
if (-not $cngKey.Key) { throw 'The selected key is not CNG-backed; this export method does not apply.' }
$hKey = $cngKey.Key.Handle.DangerousGetHandle()

# Flip the in-memory export policy to allow plaintext export
[NCrypt]::NCryptSetProperty($hKey, [NCrypt]::EXPORT_POLICY, [NCrypt]::ALLOW, 4, 0) | Out-Null

# The private key can now be exported, let's find its size
$keySize = 0
[NCrypt]::NCryptExportKey($hKey, [IntPtr]::Zero, [NCrypt]::PKCS8_BLOB, [IntPtr]::Zero, $null, 0, [ref]$keySize, 0) | Out-Null

# And perform the actual export now
$keyBytes = New-Object byte[] $keySize
$result   = [NCrypt]::NCryptExportKey($hKey, [IntPtr]::Zero, [NCrypt]::PKCS8_BLOB, [IntPtr]::Zero, $keyBytes, $keySize, [ref]$keySize, 0)
if ($result -ne 0) {throw ("NCryptExportKey failed: 0x{0:X8}" -f $result) }

Write-Host "`n# PKCS#8 private key for: $($cert.Subject)`n" -ForegroundColor Cyan
Write-Host (ConvertToPem $keyBytes)
```

A few things are worth keeping in mind:

- The output is a __plaintext private key__. Treat it like the secret it is.
- If you also need the certificate that goes with that private key (and you probably do), you can export it through the
  regular Windows export.
- The "non-exportable" flag never encrypts the key against that user; it merely tells the software not to offer an 
  export. It's a convenience guardrail, not a security boundary — a good thing to keep in mind when reasoning about 
  where your keys are actually safe.
