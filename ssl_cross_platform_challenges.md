# Swift Security on Apple and Linux


One of the biggest challenges that Swift faces in its road to becoming a cross-platform language is designing consistent APIs and functionality when the underlying frameworks are not shared across the platforms. A number of different approaches can be adopted, ranging from re-writing entire frameworks to be compatible on the new platforms (like the Foundation project that is currently being ported to Linux), to writing a common wrapper library over the different native frameworks. This latter approach is what we adopted when bringing Secure Socket Layer (SSL) and Transport Layer Security (TLS) support to Kitura. 

In this post, we look at the security support in macOS and Linux and in particular transport level security. We show how the difference in support has resulted in slightly different cross-platform behaviors in BlueSSLService, which is the underlying framework for Kitura's current SSL/TLS support. 

If you are interested in how a simple Kitura application can be served on top of TLS and thus provide private and authenticated communication, check out Part one  (https://developer.ibm.com/swift/2016/09/22/securing-kitura-part-1-enabling-ssltls-on-your-swift-server/)  of this series.

## BlueSSLService

BlueSSLService (https://github.com/IBM-Swift/BlueSSLService) is the underlying framework that integrates with Kitura web server to provide SSL/TLS. It was designed as an add-in framework for Sockets using a delegate/protocol model. BlueSSLService currently supports BlueSocket (https://github.com/IBM-Swift/BlueSocket) which implements low-level sockets and is used by Kitura; however, any Swift Socket framework that implements the SSLServiceDelegate (https://github.com/IBM-Swift/BlueSocket/blob/master/Sources/SocketProtocols.swift#L90) protocol can use BlueSSLService. 


### Design Goals for BlueSSLService


We designed BlueSSLService with the following goals in mind:

1 - Support both macOS and Linux

2 - Integrate with native security libraries on supported platforms so that the user is not responsible for installing and maintaining any other libraries;

3 - Provide a consistent and unified Swift interface so that the developer can write simple, cross-platform applications;

4 - Do all the heavy lifting so that the user just has to configure the service, assign it to a socket and forget about it.


### Cross-Platform Support

The greatest challenge in our design of BlueSSLService was the discrepancy between the native security libraries available on macOS and Linux that implemented SSL/TLS. After some investigation of the security support on the platforms, we decided on OpenSSL on Linux and Apple Secure Transport on macOS. 
In the following we will give a quick overview of the different security libraries on each platform and how some of their differences challenged our goal of providing consistent and unified APIs.

# Background


## Linux and OpenSSL

OpenSSL is the most common and prevalent open source security library on Linux. As of 2014, it was estimated that at least 66% of the web servers on the Internet use OpenSSL (via Apache and nginx) (https://news.netcraft.com/archives/2014/04/02/april-2014-web-server-survey.html). 


OpenSSL is a general-purpose security library with two primary (sub)libraries: `libssl` and `libcrypto`. `libcrypto` is a comprehensive and full-featured cryptographic library that provides the fundamental cryptographic routines used by `libssl`, which implements the SSL and TLS protocols. 


<!--
OpenSSL is written in C and is compatible across a number of platforms, including Unix, Windows, and macOS.
There are a lot of tools using OpenSSL’s libraries to secure data or establish secure connections
This does not count other server side software that use OpenSSL, such as email servers (SMTP, POP and IMAP protocols), chat servers (XMPP protocol), virtual private networks (SSL VPNs) or network appliances.  
-->


An important feature of OpenSSL is that it contains the OpenSSL FIPS Object Module (https://www.openssl.org/docs/fipsnotes.html), which is FIPS 140-2 (https://en.wikipedia.org/wiki/FIPS_140-2) validated (not just compliant). FIPS is a US Government cryptographic standard and is a requirement by most government agencies and many enterprises. Being compliant does not prove that the library has no bug or vulnerability, rather that best practices were used.  



<!--
; in fact OpenSSL is one of only two open sourced projects that are FIPS validated. 
-->

<!-- that provides a benchmark for implementing cryptographic software 
The standard specifies best practices for implementing crypto algorithms, handling key material and data buffers, and working with the operating system. -->

While there are a number of alternative crypto and SSL/TLS libraries, the most closely aligned to OpenSSL is LibreSSL which came about as a result of code review in response to Heartbleed  (http://heartbleed.com). Its aim is to modernize the codebase and improve security and memory management of OpenSSL. One limitation of LibreSSL though, is that it is deliberately not FIPS compliant (for the library to be lean). 

<!--
This page does a good job of comparing OpenSSL and LibreSSL http://resources.infosecinstitute.com/libressl-the-secure-openssl-alternative/


One of the main limitations of OpenSSL is the large bulk of code it has inherited in its 20 years of life. This bloat includes support for legacy functions, legacy architecture and insecure defaults. 
-->


## macOS Platform and Security Frameworks

In contrast to Linux's single, general-purpose library, macOS has an extensive number of libraries that together make up Apple's security frameworks. The APIs of these frameworks are exposed at different levels of the system with specific motivation and behavior in mind, including low-level crypto in CommonCrypto, high-level Core Foundation level APIs in the Security Framework, and app-level APIs in SecurityFoundation and SecurityInterface. Figure XX shows an architectural stack  of Apple's security frameworks from 2012. 
Figure XX illustrates the components that make up the Security Framework.


IMAGE - https://developer.apple.com/videos/play/wwdc2012/704/ - This is a framework stack for security frameworks on macOS described in WWDC 2012's The Security Framework session. 


IMAGE - These are some of the basic components that make up the Security Framework on macOS. This image was captured from WWDC 2012's The Security Framework session.

SSL/TLS APIs are available at multiple interfaces (https://developer.apple.com/library/content/documentation/Security/Conceptual/cryptoservices/SecureNetworkCommunicationAPIs/SecureNetworkCommunicationAPIs.html):

- At a high level, we can use an https URL with the URL Loading System (the NSURL class, CFURL functions, and related classes).
- At a lower level, we can use the CFNetwork API to negotiate an SSL or TLS connection.
- For maximum control, we can use the Secure Transport API in OS X or in iOS 5.0 and later.

We chose to use the APIs at the Secure Transport layer in order to have maximum control and parity with the OpenSSL APIs.


<!--For server security, the two fundamental libraries of interest are `CommonCrypto` which implements low level crypto functions, and `Secure Transport`  which implements SSL/TLS and Datagram Transport Layer Security (DTLS). Both these libraries inherit from `corecrypto` which is FIPS 140-2 Level 1 validated.
-->


### OpenSSL on macOS

Although OpenSSL is compatible with macOS, Apple deprecated the programmatic interface to OpenSSL in OS X v10.7, because OpenSSL did not provide API compatibility across versions. Apple advises that ``use of the Apple-provided OpenSSL libraries by apps is strongly discouraged`` (https://developer.apple.com/library/content/documentation/Security/Conceptual/cryptoservices/SecureNetworkCommunicationAPIs/SecureNetworkCommunicationAPIs.html) and instead points to 
the use of `CommonCrypto` and `Secure Transport` APIs as well as `CFNetwork` level APIs for Certificate, Key, and Trust services. 
Also, since OpenSSL has been deprecated on macOS, it is no longer submitted for FIPS 104 validation.

<!--According to Apple, applications that require OpenSSL should not use the system’s supplied `libcrypto.dylib`, and instead applications need to build their own version of OpenSSL and include it in the app.-->

There are two options for a developer when an application requires OpenSSL:

1 - Application dynamically links to the latest OpenSSL: Application users are responsible for maintaining the latest copy of OpenSSL on their platform. This can lead to potential API incompatibility. 

2 - Application (dynamically or statically) links to a specific version of OpenSSL: In this case, the application does not get OpenSSL updates for free. This is undesirable and critical when OpenSSL requires security updates.


From these two options, only the first is secure (assuming the user is vigilant about always being up to date) but has the limitation that the application does not necessarily have binary compatibility with the OpenSSL library and therefore updating OpenSSL can cause the application to crash.


http://rentzsch.tumblr.com/post/33696323211/wherein-i-write-apples-technote-about-openssl-on

# Challenges in Unifying APIs


In order for BlueSSLService to provide a unified Swift API over OpenSSL in Linux and Secure Transport on macOS, we face a number of challenges. These stemmed from the difference in behavior by each library. In the sections below I will describe three main challenges we faced, and how they affected BlueSSLService. 

### Certificate Format

OpenSSL supports a number of different encodings for X.509 certificates, including:

- PEM: a Base64-encoded ASCII format of public and private data,
- DER: a binary blob of the public and private data,
- PKCS#7: a Base64-encoded ASCII of public data,
- PKCS#12: a binary blob of both public and private data that can be password encrypted. This is generally one blob that contains both the certificate and the key data.    

In contrast, Apple platforms only support PKCS#12. In addition, they do not support conversion between PKCS#12 and other certificate formats. The developer needs to use OpenSSL or other existing tools to convert between the formats. 

Because of this certificate mismatch, not all  constructors for SSL configuration struct `Configuration` in BlueSSLService and `SSLConfig` in Kitura, work on macOS and indeed only the following is supported on macOS: XXX

```
SSLConfig(withChainFilePath chainFilePath: String? = nil, withPassword password: String? = nil, usingSelfSignedCerts selfSigned: Bool = true, cipherSuite: String? = nil
```

If you need to convert a PEM key/certificate pair to PKCS#12 (a single combined file), you can use the following OpenSSL command:

```
openssl pkcs12 -export -in cert.pem -inkey key.pem -out certkey.pfx
```

where cert.pem is the PEM encoded certificate, key.pem is the PEM encoded private key, and certkey.pfx is the new password-encrypted cert/key PKCS#12 container. 


### SSL/TLS Cipher suite 

Before a secure communication channel can be established between a client and a server, a “handshake” or negotiation phase occurs where the client and server decide what protocol version to use as well as which ciphers, or what cipher suite. This cipher suite defines what operations (such as key exchange, hashing, signing or encrypting) the client or server supports and what is their order of preference.

An example cipher suite configuration is: `TLS_RSA_WITH_AES_128_CBC_SHA256`.
This means that an RSA key exchange is used in conjunction with AES-128-CBC (the symmetric cipher) and SHA256 hashing is used for message authentication.

The choice of cipher suite defines the security operations of the TLS protocol, enables features such as forward secrecy, and disables weak operations such as RC4 or MD5. There is a trade off between higher levels of security and compatibility with legacy clients and performance. Older clients might not support newer algorithms and  extra security is sometimes not worth the computing cost in software. Great references for choosing appropriate cipher suites include Mozilla (https://wiki.mozilla.org/Security/Server_Side_TLS) and SSLLabs (https://www.ssllabs.com/projects/best-practices/index.html).

There are a number of differences in the way OpenSSL and Secure Transport treat cipher suite setup. The first difference is the name with which the algorithms are refered to. SSL/TLS specification and Internet Assigned Numbers Authority (https://www.iana.org/assignments/tls-parameters/) define algorithm names with an associated code such as:

```
Code		IANA Name					
0x00,0x6B    TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
```

Secure Transport identifies the cipher suite string using their IANA code. OpenSSL however defines its own set of names for the cipher suites. For example:

```
Code	IANA Name							   OpenSSL Name
0x00,0x6B    TLS_DHE_RSA_WITH_AES_256_CBC_SHA256   DHE-RSA-AES256-SHA256
```

Furthermore, OpenSSL defines logical operations in the cipher suite. The cipher strings which compose the cipher suite have a variety of properties:

- They can consist of a single cipher suite such as RC4-SHA,
- They can represent a list of cipher suites containing a certain algorithm, or cipher suites of a certain type. For example SHA1 represents all ciphers suites using the digest algorithm SHA1 and SSLv3 represents all SSL v3 algorithms.
- Lists of cipher suites can be combined in a single cipher string using the + character. This is used as a logical and operation XXXXXXX??. For example SHA1+DES represents all cipher suites containing the SHA1 and the DES algorithms.
- Each cipher string can be optionally preceded by the characters !, - or +, which in respectively mean permanent delete, delete or move to end of list.

Unfortunately there is not a  lot of data on how cipher suites can be set in Secure Transport and whether operators such as !, - or +, or equivalents, can be used.

Because of the different way OpenSSL and Secure Transport handle cipher suites, currently a developer needs to be able to map their cipher strings across platforms.


## Concluding

In this post, we looked at the native security support on macOS and Linux and explored some of their differences. We explored some of the challenges that our Swift team faced when implementing a single Swift SSL/TLS library for Kitura which was cross platform. Although our intention is to provide a consistent and unified interface to transport security where the library does all the heavy lifting, we are limited by the underlying frameworks. 


However we are just starting, and the developement of our frameworks depends on use cases. 

We would love to hear from you on your usage of BlueSSLService (e.g., by itself or through Kitura), the limitations you face, and the ideas you have for improving our services.






The OpenSSL library on Linux provides a general library that is composed of both, low-level crypto APIs as well as transport-level APIs. On macOS, the security stack is distributed through multiple levels, from low-level primitives to transport and application-level XXXX application-level what? APIs? primitives?. The exposition XXX(is this the right word)??? of the APIs are defined with specific motivation and behavior in  mind. 

 


This difference in nature of the native security frameworks on Linux and XXX???

and is exposed as different components through out the stack. XXX???

One of the biggest challenges that Swift faces in its road to becoming a cross-platform language is designing consistent APIs and functionality when the underlying frameworks are not shared across the platforms. A number of different approaches can be adopted, ranging from re-writing entire frameworks to be compatible on the new platforms (Foundation project that is currently being ported to Linux), to writing a common wrapper library over the different native frameworks. This latter approach is what we adopted when bringing Secure Socket Layer (SSL) and Transport Layer Security (TLS) support to Kitura.  XXX(Same paragraphs in introduction?)

In this blog, we will first look at the security support in macOS and Linux and in particular transport level security. We show how the difference in support have resulted in slightly different cross-platform behaviors in BlueSSLService, which is the underlying framework for Kitura's current SSL/TLS support. 





http://sites.sas.upenn.edu/shaheenb/files/applied-crypto-hardening_1.pdf

https://github.com/apple/cups/blob/master/cups/tlscheck.c



ciphersuite registry: https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4

http://www.thesprawl.org/research/tls-and-ssl-cipher-suites/

http://www.tomshardware.com/news/bearssl-tls-library-ssl-openssl,32976.html

https://thycotic.com/company/blog/2014/05/16/ssl-beyond-the-basics-part-2-ciphers/



