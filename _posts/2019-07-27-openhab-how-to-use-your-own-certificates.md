---
layout: post
title:  OpenHAB How to use your own certificates
description: >
categories: [openhab, cryptography]
---
# openHAB: How to use your own certifcate authority
![openhab](https://vuln.dev/assets/img/openhab_logo.svg)

openHAB has mainly three different ways to remote access it via an encrypted communication. All three of them are well documented whether because there are lot of tutorials in general (e.g. for VPN) or within the documentation of openHAB itself. You can find those ways [here](https://www.openhab.org/docs/installation/security.html).

## Use your own certificate authority
The way for using your own certificates directly in openHAB and without the use of a reverse proxy itself is unfortunately not so well documented. By default, openHAB uses a self-signed certificate which will be generated while the instance is started for the first time. This prevents every openHAB server out there from using the same certificate, but it does not provide real security. Self-signed certificates should never be used for security purposes.

The better way for development and/or test instances is e.g. the project  [mkcert](https://github.com/FiloSottile/mkcert) of Filippo Valsorda. I strongly recommend to use it in order to have valid certificates for these systems as well.

If it goes to productive instances and as well for your own home server you should use a certificate authority whether your own or an official one.

## Create the certificate authority
Because there are a lot of manuals on the internet that deal with creating your own CA with OpenSSL, I will go through it very fast.

First you have to generate a secret private key:
~~~
$ openssl genrsa -aes256 -out ca-key.pem 4096
~~~
The key is stored in the file ca-key.pem and has a length of 4096 bits. You can also use longer or shorter keys but the recommended minimum size is 2048 bits. By using the option **_-aes256_** the key will be protected with a password. This password must be entered each time the key will be used.

The next step is to create the root certificate of the CA. This certificate can then be imported into browsers and operating systems. The certificate could be generated as follows:
~~~
$ openssl req -x509 -new -nodes -extensions v3_ca -key ca-key.pem -days 3650 -out ca-root.pem -sha384
~~~
This certificate is valid for 10 years and is using sha384 as hash algorithm. The attributes of the CA can be as follows:
~~~
Country Name (2 letter code) [AU]: DE
State or Province Name (full name) [Some-State]: Berlin
Locality Name (eg, city) []: Berlin
Organization Name (eg, company) [Internet Widgits Pty Ltd]: foo
Organizational Unit Name (eg, section) []: bar
Common Name (eg, YOUR name) []: example.net
Email Address []: foo@example.net
~~~
## Issue a new certificate
The next thing you have to do is make yourself a private key again.
~~~
$ openssl genrsa -aes256 -out cert-key.pem 4096
~~~
You should use a password here as well especially if you want to save the key after integrating it into openHABs keystore. 

Now you have to create a certificate signing request. The common name of this attribute must be the same as the hostname of the server it is issued to. Alternatively, you can use the IP address if you only want to call the instance via IP.
~~~
$ openssl req -new -key cert-key.pem -out cert.csr -sha384
~~~
When you are done, you get the certificate request in **_cert.csr_**, which can be further processed by your already created CA.

With the rootCAs public certificate, its key and the certificate signing request you can create a signed certificate which is valid for 5 years.
~~~
$ openssl x509 -req -in cert.csr -CA ca-root.pem -CAkey ca-key.pem -CAcreateserial -out cert-pub.pem -days 1825 -sha384
~~~
OpenSSL asks for the password of the rootCA. If a signed certificate has been created you can delete the certificate signing request afterwards. The public certificate can be found in the file **_cert-pub.pem_**.

## Create the keystore for openHAB
All steps so far can be made on any pc where OpenSSL is installed. From now on you need the tool **_keytool_** as well. It should be already installed if you use one of openHABs official images. As I mentioned at the beginning openHAB is using a self signed certificate. This is stored in **_${USER_DATA}etc/keystore_**. You can list its content with:
~~~
$ keytool -list -keystore keystore
~~~
You will be prompted for a password which is not required for listing so that you can simply press enter. You will see somthing like:

~~~
Enter keystore password:

*****************  WARNING WARNING WARNING  *****************
* The integrity of the information stored in your keystore  *
* has NOT been verified!  In order to verify its integrity, *
* you must provide your keystore password.                  *
*****************  WARNING WARNING WARNING  *****************

Keystore type: jks
Keystore provider: SUN

Your keystore contains 1 entry

mykey, Jul 27, 2019, PrivateKeyEntry,
Certificate fingerprint (SHA1): 57:D8:70:44:16:53:A3:C7:72:1D:C0:06:F7:7F:D2:53:F3:09:29:17

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore keystore -destkeystore keystore -deststoretype pkcs12".
~~~
Before adding the new certifcate and key into the keystore the actual one should be moved to save it:
~~~
$ mv ${USER_DATA}etc/keystore ${USER_DATA}etc/keystore.bak
~~~

#### Loading Keys and Certificates via PKCS12
Because we have the certificate and key in different files, we need to combine them into a PKCS12 format file to load them into a new keystore. The following command combines the key in **_cert-key.pem_** and the certificate in the **_cert-pub.pem_** file into the **_openhab.example.net.p12_** file:
~~~
$ openssl pkcs12 -inkey cert-key.pem -in cert-pub.pem -export -out openhab.example.net.p12 -password pass:openhab
~~~
In case you have a chain of certificates, because your CA is an intermediary, build the PKCS12 file as follows:
~~~
$ cat example.crt intermediate.crt [intermediate2.crt] ... rootCA.crt > cert-chain.txt
$ openssl pkcs12 -export -inkey example.key -in cert-chain.txt -out openhab.example.net.p12 -password pass:openhab
~~~~
As you maybe recognised the p12 file will be protected with the password __openhab__. Now you can create the new keystore with:
~~~
$ keytool -importkeystore -srckeystore openhab.example.net.p12 -srcstoretype PKCS12 -destkeystore keystore -destalias mykey -srcalias openhab.example.net
~~~
You will be asked for the new destination keystores password first two times and for the p12 password afterwards:
~~~
Importing keystore openhab.example.net.p12 to keystore...
Enter destination keystore password: openhab
Re-enter new password: openhab
Enter source keystore password: openhab

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore keystore -destkeystore keystore -deststoretype pkcs12".
~~~
Now make sure that the **_keystore_** file has the correct permissions and restart your openhab instance. 

**Do not forget to import your rootCAs public certificate into your browser!**
