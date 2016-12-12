<a href="https://developer.ibm.com/swift/wp-content/uploads/sites/69/2016/10/KituraSecurity2-1.png"><img src="https://developer.ibm.com/swift/wp-content/uploads/sites/69/2016/10/KituraSecurity2-1.png" alt="Securing Kitura" width="371" height="224" class="alignright size-full wp-image-2713" /></a>

<!--One of the biggest challenges that Swift faces in its road to becoming a cross-platform language is designing consistent APIs and functionality when the underlying frameworks are not shared across the platforms. A number of different approaches can be adopted, ranging from re-writing entire frameworks to be compatible on the new platforms (like the Foundation project that is currently being ported to Linux), to writing a common wrapper library over the different native frameworks. This latter approach is what we adopted when bringing Secure Socket Layer (SSL) and Transport Layer Security (TLS) support to Kitura. 
-->


<p>Having a consistent development experience for Swift across iOS, tvOS, macOS and now Linux helps to drive higher developer productivity as well as better reuse of Swift assets/libraries across these platforms.  The challenge then is to design and maintain consistent Swift APIs across these platforms while leveraging libraries and capabilities that might be specific to the given systems (like OpenSSL vs. Secure Transport or libcurl vs. CFNetwork).

<p>
In this post, we look at the security support in macOS and Linux and in particular transport layer security. We show how the difference in underlying frameworks has resulted in slightly different cross-platform behavior in <a href="https://github.com/IBM-Swift/BlueSSLService">BlueSSLService</a>, which is the underlying framework for Kitura's current SSL/TLS support. While we are ultimately driving for consistent APIs in our frameworks across platforms, some differences are still visible in how the developer interacts with these APIs.


<p>
By the end of this post you should better understand how our team at Swift@IBM has delivered consistent security APIs in Kitura across macOS and Linux. This is particularly helpful as a supplement to <a href="https://developer.ibm.com/swift/2016/09/22/securing-kitura-part-1-enabling-ssltls-on-your-swift-server/">Part one</a> of our blog series describing how a simple Kitura application can be served on top of TLS and thus provide private and authenticated communication.


<h1>BlueSSLService</h1>

<a href="https://github.com/IBM-Swift/BlueSSLService">BlueSSLService</a> is the underlying framework that integrates with Kitura web server to provide SSL/TLS. It was designed as an add-in framework for Sockets using a delegate/protocol model. BlueSSLService currently supports <a href="https://github.com/IBM-Swift/BlueSocket">BlueSocket</a> which implements low-level sockets and is used by Kitura; however, any Swift Socket framework that implements the <a href="https://github.com/IBM-Swift/BlueSocket/blob/master/Sources/SocketProtocols.swift#L90">SSLServiceDelegate</a> protocol can use BlueSSLService. 


<h2> Design Goals for BlueSSLService </h2>


<p>We designed BlueSSLService with the following goals in mind:

<ol>
<li>Support both macOS and Linux </li>
<li>Integrate with native security libraries on supported platforms so that the user is not responsible for installing and maintaining any other libraries; </li>
<li>Provide a consistent and unified Swift interface so that the developer can write simple, cross-platform applications; </li>
<li>Do all the heavy lifting so that the user just has to configure the service, assign it to a socket and forget about it. </li>
</ol>

<h2>Cross-Platform Support </h2>

<p>The greatest challenge in our design of BlueSSLService was the discrepancy between the native security libraries available on macOS and Linux that implemented SSL/TLS. After some investigation of the security support on the platforms, we decided on OpenSSL on Linux and Apple Secure Transport on macOS. 
In the following we will give a quick overview of the different security libraries on each platform and how some of their differences challenged our goal of providing consistent and unified APIs.

<h1> Background </h1>


<h2> Linux and OpenSSL </h2>

<p>OpenSSL is the most common and prevalent open source security library on Linux. As of 2014, <a href="https://news.netcraft.com/archives/2014/04/02/april-2014-web-server-survey.html">it was estimated </a>that at least 66% of the web servers on the Internet use OpenSSL (via Apache and nginx). 


<p>OpenSSL is a general-purpose security library with two primary (sub)libraries: <code>libssl</code> and <code>libcrypto</code>. <code>libcrypto</code> is a comprehensive and full-featured cryptographic library that provides the fundamental cryptographic routines used by <code>libssl</code>, which implements the SSL and TLS protocols. 


<p>An important feature of OpenSSL is that it contains the <a href="https://www.openssl.org/docs/fipsnotes.html">OpenSSL FIPS Object Module</a>, which is <a href="https://en.wikipedia.org/wiki/FIPS_140-2">FIPS 140-2</a> validated (not just compliant). FIPS is a US Government cryptographic standard and is a requirement by most government agencies and many enterprises. Being compliant does not prove that the library has no bug or vulnerability, rather that best practices were used.  

<p>While there are a number of alternative crypto and SSL/TLS libraries, the most closely aligned to OpenSSL is LibreSSL which came about as a result of code review in response to <a href="http://heartbleed.com">Heartbleed</a>. Its aim is to modernize the codebase and improve security and memory management of OpenSSL. One limitation of LibreSSL though, is that it is deliberately not FIPS compliant (for the library to be lean). 


<h2> macOS Platform and Security Frameworks </h2>

<p>In contrast to Linux's single, general-purpose library, macOS has an extensive number of libraries that together make up Apple's security frameworks. The APIs of these frameworks are exposed at different levels of the system with specific motivation and behavior in mind, including low-level crypto in CommonCrypto, high-level Core Foundation level APIs in the Security Framework, and app-level APIs in SecurityFoundation and SecurityInterface. The Figures below show an architectural stack  of Apple's security frameworks, as well as the components that make up the Security Framework.

[caption id="attachment_3445" align="aligncenter" width="826"]<a href="https://developer.ibm.com/swift/wp-content/uploads/sites/69/2016/12/WWDC12-SecurityFrameworkStack2.png"><img src="https://developer.ibm.com/swift/wp-content/uploads/sites/69/2016/12/WWDC12-SecurityFrameworkStack2.png" alt="This is a framework stack for security frameworks on macOS described in WWDC 2012&#039;s The Security Framework session." width="826" height="458" class="size-full wp-image-3445" /></a>  This is a framework stack for security frameworks on macOS described in WWDC 2012's The Security Framework session. This image was captured from WWDC 2012's <a href="https://developer.apple.com/videos/play/wwdc2012/704/">The Security Framework session.[/caption]

[caption id="attachment_3446" align="aligncenter" width="831"]<a href="https://developer.ibm.com/swift/wp-content/uploads/sites/69/2016/12/WWDC12-SecurityFrameworkStack.png"><img src="https://developer.ibm.com/swift/wp-content/uploads/sites/69/2016/12/WWDC12-SecurityFrameworkStack.png" alt="These are some of the basic components that make up the Security Framework on macOS. This image was captured from WWDC 2012&#039;s The Security Framework session" width="831" height="464" class="size-full wp-image-3446" /></a> These are some of the basic components that make up the Security Framework on macOS. This image was captured from WWDC 2012's <a href="https://developer.apple.com/videos/play/wwdc2012/704/">The Security Framework session</a>.[/caption]


SSL/TLS APIs are available at <a href="https://developer.apple.com/library/content/documentation/Security/Conceptual/cryptoservices/SecureNetworkCommunicationAPIs/SecureNetworkCommunicationAPIs.html">multiple interfaces</a>:

<p>
<ul>
	<li>At a high level, we can use an https URL with the URL Loading System (the NSURL class, CFURL functions, and related classes).</li>
	<li>At a lower level, we can use the CFNetwork API to negotiate an SSL or TLS connection.</li>
	<li>For maximum control, we can use the Secure Transport API in OS X or in iOS 5.0 and later.</li>
</ul>

<p>We chose to use the APIs at the <code>Secure Transport</code> layer in order to have maximum control and parity with the OpenSSL APIs.


<h3> OpenSSL on macOS </h3>

<p>Although OpenSSL is compatible with macOS, Apple deprecated the programmatic interface to OpenSSL in OS X v10.7, because OpenSSL did not provide API compatibility across versions. Apple <a href="https://developer.apple.com/library/content/documentation/Security/Conceptual/cryptoservices/SecureNetworkCommunicationAPIs/SecureNetworkCommunicationAPIs.html">advises</a> that ``use of the Apple-provided OpenSSL libraries by apps is strongly discouraged`` and instead points to 
the use of CommonCrypto and Secure Transport APIs as well as CFNetwork level APIs for Certificate, Key, and Trust services. 

<p>Also, since OpenSSL has been deprecated on macOS, it is no longer submitted for FIPS 104 validation.


<p>There are two options for a developer when an application requires OpenSSL:

<p>
<ol>
	<li>Application dynamically links to the latest OpenSSL: Application users are responsible for maintaining the latest copy of OpenSSL on their platform. This can lead to potential API incompatibility.</li>
	<li>Application (dynamically or statically) links to a specific version of OpenSSL: In this case, the application does not get OpenSSL updates for free. This is undesirable and critical when OpenSSL requires security updates.</li>
</ol>

<p>From these two options, only the first is secure (assuming the user is vigilant about always being up to date) but has the limitation that the application does not necessarily have binary compatibility with the OpenSSL library and therefore updating OpenSSL can cause the application to crash.


<h1> Challenges in Unifying APIs </h1>


<p>In order for BlueSSLService to provide a unified Swift API over OpenSSL in Linux and Secure Transport on macOS, we face a number of challenges. These stemmed from the difference in behavior by each library. In the sections below I will describe three main challenges we faced, and how they affected BlueSSLService. 

<h2> Certificate Format </h2>

<p>OpenSSL supports a number of different encodings for X.509 certificates, including:
<p>
<ul>
	<li>PEM: a Base64-encoded ASCII format of public and private data;</li>
	<li>DER: a binary blob of the public and private data;</li>
	<li>PKCS#7: a Base64-encoded ASCII of public data;</li>
	<li>PKCS#12: a binary blob of both public and private data that can be password encrypted. This is generally one blob that contains both the certificate and the key data</li>
</ul>

<p>In contrast, Apple platforms only support PKCS#12. In addition, they do not support conversion between PKCS#12 and other certificate formats. The developer needs to use OpenSSL or other existing tools to convert between the formats. 

<p>Because of this certificate mismatch, not all  constructors for SSL configuration struct work on macOS and indeed only the following is supported on macOS. This includes <code>Configuration</code> in BlueSSLService and <code>SSLConfig</code> in Kitura. Below is the only <code>SSLConfig</code> constructor that works on macOS:

[swift]
SSLConfig(withChainFilePath chainFilePath: String? = nil, withPassword password: String? = nil, usingSelfSignedCerts selfSigned: Bool = true, cipherSuite: String? = nil
[/swift]

<p>If you need to convert a PEM key/certificate pair to PKCS#12 (which converts two files into a single combined file), you can use the following OpenSSL command:

[bash]
openssl pkcs12 -export -in cert.pem -inkey key.pem -out certkey.pfx
[/bash]

<p>where <code>cert.pem</code> is the PEM encoded certificate, <code>key.pem</code> is the PEM encoded private key, and <code>certkey.pfx</code> is the new password-encrypted cert/key PKCS#12 container. 


<h2> SSL/TLS Cipher suite  </h2>

<p>Before a secure communication channel can be established between a client and a server, a “handshake” or negotiation phase occurs where the client and server decide what protocol version to use as well as which ciphers, or what cipher suite. This cipher suite defines what operations (such as key exchange, hashing, signing or encrypting) the client or server supports and what is their order of preference.

<p>An example cipher suite configuration is: <code>TLS_RSA_WITH_AES_128_CBC_SHA256</code>.
This means that an RSA key exchange is used in conjunction with AES-128-CBC (the symmetric cipher) and SHA256 hashing is used for message authentication.

<p>The choice of cipher suite defines the security operations of the TLS protocol, enables features such as forward secrecy, and disables weak operations such as RC4 or MD5. There is a trade off between higher levels of security and compatibility with legacy clients and performance. Older clients might not support newer algorithms and  extra security is sometimes not worth the computing cost in software. Great references for choosing appropriate cipher suites include <a href="https://wiki.mozilla.org/Security/Server_Side_TLS">Mozilla</a> and <a href="https://www.ssllabs.com/projects/best-practices/index.html">SSLLabs</a>.

<p>There are a number of differences in the way OpenSSL and Secure Transport treat cipher suite setup. The first difference is the name with which the algorithms are refered to. SSL/TLS specification and <a href="https://www.iana.org/assignments/tls-parameters/">Internet Assigned Numbers Authority</a> (IANA) define algorithm names with an associated code such as:
<p>
[su_table]<table>
<tr>
	<td>Code</td>
	<td>IANA Name</td>
</tr>
<tr>
	<td><code>0x00,0x6B</code></td>
	<td> <code>TLS_DHE_RSA_WITH_AES_256_CBC_SHA256</code></td>
</tr>
</table>[/su_table]

<p>Secure Transport identifies the cipher suite string using their IANA code. OpenSSL however defines its own set of names for the cipher suites. For example:

<p>
[su_table]<table>
<tr>
	<td>Code</td>
	<td>IANA Name</td>
	<td>OpenSSL Name</td>
	<td>macOS Name</td>
</tr>
<tr>
	<td><code>0x00,0x6B</code></td>
	<td> <code>TLS_DHE_RSA_WITH_AES_256_CBC_SHA256</code></td>
	<td>DHE-RSA-AES256-SHA256<code>0x00,0x6B</code></td>
	<td><code>6B</code></td>
</tr>
</table>[/su_table]

<p>Furthermore, OpenSSL defines logical operations in the cipher suite. The cipher strings which compose the cipher suite have a variety of properties:

<p>
<ul>
	<li>They can consist of a single cipher suite such as RC4-SHA.</li>
	<li>They can represent a list of cipher suites containing a certain algorithm, or cipher suites of a certain type. For example SHA1 represents all ciphers suites using the digest algorithm SHA1 and SSLv3 represents all SSL v3 algorithms.</li>
	<li>Lists of cipher suites can be combined in a single cipher string using the + character. This is used as a logical <code>AND</code> operation. For example SHA1+DES represents all cipher suites containing the SHA1 and the DES algorithms.</li>
	<li>Each cipher string can be optionally preceded by the characters !, - or +, which in respectively mean permanent delete, delete or move to end of list.</li>
</ul>

<p>Unfortunately there is not a  lot of data on how cipher suites can be set in Secure Transport and whether operators such as !, - or +, or equivalents, can be used.

<p>Because of the different way OpenSSL and Secure Transport handle cipher suites, currently a developer needs to be able to map their cipher strings across platforms.


<h1> Concluding </h1>

<p>In this post, we looked at the native security support on macOS and Linux and explored some of their differences. We explored some of the challenges that our Swift@IBM team faced when implementing a single Swift SSL/TLS library for Kitura which was cross-platform. While we are driving for consistent APIs that improve developer productivity, some differences are still visible. 

<p>However this is still the beginning of how security should be handled across platforms. We plan on taking our insights from our experience in building security support in Kitura  to the community and in particular the <a href="https://swift.org/blog/server-api-workgroup/">Server APIs Work Group</a> <a href="https://swift.org/server-apis/">Security Group</a> to build a standard means of providing cross-platform security.

<p>In the meantime, we would love to hear from you on your usage of BlueSSLService (e.g., by itself or through Kitura), any limitations you face, and the ideas you have for improving our services.

<p>Learn more about Swift@IBM by visiting our <a href="https://developer.ibm.com/swift/">DevCenter</a>. 

<p>Here are some past posts in our blog series on Securing Kitura:

<ul>
	<li><a href="https://developer.ibm.com/swift/2016/09/22/securing-kitura-part-1-enabling-ssltls-on-your-swift-server/">Securing Kitura Part 1: Enabling SSL/TLS on your Swift Server</a></li>
	<li><a href="https://developer.ibm.com/swift/2016/10/17/securing_kitura_basic_authentication/">Securing Kitura Part 2: Basic Authentication</a></li>

</ul>

<p>Swift On!



[pn-grid cols="8" class="column-gutter-0"]
[grid span="1"]
<script async defer src="https://buttons.github.io/buttons.js"></script>
<a class="github-button" href="https://github.com/IBM-Swift/Kitura" data-style="mega" data-count-href="/IBM-Swift/Kitura/stargazers" data-count-api="/repos/IBM-Swift/Kitura#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star IBM-Swift/Kitura on GitHub">Star</a>
[/grid]
[grid span="1"]
<script async defer src="https://swift-at-ibm-slack.mybluemix.net/slackin.js?large"></script>
[/grid]
[/pn-grid]




