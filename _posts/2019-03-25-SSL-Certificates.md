---
layout: post
title: TLS Web Server Certificate Manipulation
---

This tech note might help you if you are experiencing trouble importing an SSL server certifcate to be used e.g. in a web or application server such as apache2 httpd.

[This site helped me to get it right](https://www.sslshopper.com/ssl-certificate-tools.html){:target='_blank'}.

##  Generate a Certificate Signing Request (CSR)

If you do not have certificate yet you will either generate a self-signed certificate or order one from a Certificate Authority (CA). The latter type of certificates avoids nasty messages saying the certificate is not good. To get such a certificate you will need to generate a CSR (Certificate Signing Request). Note that you will want to also generate the private key for the certificate you are going to request with that CSR.

[What is a CSR](https://www.sslshopper.com/what-is-a-csr-certificate-signing-request.html){:target='_blank'}?

Generate a CSR for and on your server including a private key:
```
openssl req -new -newkey rsa:2048 \
            -nodes -out servername.csr \
            -keyout servername.key
```
You will be asked to provide a password to protect the private key, make sure to write this down. If you want the apache2 web server to come up automatically without having to provide that password you can strip the password from the generated private key with:
```
openssl rsa -in privateKey.pem \
            -out newPrivateKey.pem
```
Note that there might be security implications with removing the password from the private key.

[Common openssl operators](https://www.sslshopper.com/article-most-common-openssl-commands.html){:target='_blank'}.

## Generate, download and inspect the certificate

With this CSR I generated a certificate at the IBM Internal CA and download it as CRT File.

![](../images/M11.png)

Then I could check the error_log and this clearly indicated that the certificate was unreadable. Something in the style of Expected `---BLABLA CERTIFICATE`. When I searched for the error on the Internet it came back with the advice to read the certificate in non-binary mode using VIM (`vi -b file.ext`). Normally the output should read something like `---BLABLABLA CERTIFICATE`.

So... when I did this with the certificate I downloaded as from the IBM Internal CA CRT I got:

![](../images/M12.png)

This clearly indicated that the certificate I downloaded as CRT was not in a good format. 

## Converting a certificate to the needed format

I decided then to download it as PKCS7b format.  To add to the confusion, the download came with a PEM extension (a P7B extension would have been more logical).

I then converted the PKCS7b format to standard PEM with [the online converter tool](https://www.sslshopper.com/ssl-converter.html){:target='_blank'} (but this too can probably be done using openssl):

![](../images/M13.png)

As you can see below that did the trick... `vi -b "cert (1).pem"` now resulted in:

![](../images/M14.png)

This resulting format I could use in my apache2 server without errors:

![](../images/M15.png)

### Note

Firefox has its own certificate store (Chrome and Safari use the OS certificate store), so in my use case one will also have to manually install the IBM Internal CA Intermediate and Root certs manually in that store, because the Internal CA is not published in the wild.

![](../images/M16.png)
