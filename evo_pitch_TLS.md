Hi all,

I’d like to pitch some of the ideas that have been discussed in the server working group around security. To get more information on the server API group and its goals, see https://swift.org/server-apis/. (TL;DR version is to come up with a set of foundational APIs that work cross-platform, on all platforms supported by Swift.) 

For security, we have divided the scope into SSL/TLS support and crypto support. Our first goal and the subject of this pitch is TLS. This current pitch is the result of various discussions that have taken place over the past several weeks/months and several projects by groups such as Vapor, IBM, Zewo, etc. 

Our plan is to start with the main ideas presented here and work on a standard Swift library that we prototype and iterate on before finalizing on a specific interface. 

# Problem

Currently there is no standard Swift SSL/TLS library that is compatible on both Apple and Linux. Swift projects use their TLS library of choice (such as OpenSSL, LibreSSL, Security framework, etc), resulting in:
- fragmentation of the space as well as incompatibility of project dependencies if more than one security package is needed by different modules (a project cannot have  both OpenSSL and LibreSSL in its dependency graph)
- insecurity (using an unpatched or deprectaed library such as OpenSSL on macOS)
- unmaintainablity (using non-standard or non-native libraries)
- more complex code (using different APIs for each platform).

This motivates the necessity for defining a standard set of protocols that define the behavior of the TLS service and how the application and the server and networking layers beneath it interact with the TLS service.

# Design goals

It is ideal to have the following design goals as we try to come up with a solution for the above problem:

- Provide a consistent and unified Swift interface so that the developer can write simple, cross-platform applications;
- Don't implement new crypto functionality and instead use existing functions in underlying libraries;
- Plug-n-play architecture which allows the developer to decide on underlying security library of choice, e.g., OpenSSL vs LibreSSL;
- Library should be agnostic of the transport mechanism (e.g., socket, etc), whilst allowing for both blocking and non-blocking connections;
- Developers should be able to use the same TLS library for both client and server applications


# Proposal


The proposed solution basically defines a number of protocols that abstracts away:
- transport management
- TLS management

A basic diagram that shows the relationship between the proposed protocols is shown below:
![alt text](https://raw.githubusercontent.com/gtaban/blogs/master/TLSServiceArchitecture.png "Architecture of TLSService modules")


The transport management protocol essentially would be a combination of an connection object (e.g., a socket pointer, a file descriptor, etc) and a connection type.

This is the connection object that gets passed to implementation of the TLS service protocol, which also handles the read and write callbacks to the connection object.

The TLS service protocol would define a sets of methods that deal with TLS setup (certificates, server/client, etc), and TLS events (such as receiving data, encrypting and writing to connection object or reading from a connection object, decrypting and returning the data).

# Source Compatibility:

What complicates this proposal is that the Swift in server space is new and none of the interfaces have yet been defined. Although the proposed protocols are designed to be as non-breaking as possible, they do place several assumptions on the rest of the transport/application stack. 

- The application layer must import and instantiate a TLS service object which implements the TLS service protocol if it wants to enable TLS service.
- Every framework layer above the transport management layer but below the application layer (which includes HTTP and server frameworks) has an optional object that implements the TLS service protocol. If the TLS object exists, it is passed down to each layer below.
- The HTTP layer which sets up the transport management layer assigns the TLS object to the transport management delegate if the object exists. 
- The transport management layer which sets up the I/O communication implements the transport management protocol.
- The transport management layer which sets up the I/O communication and deals with the low level C system I/O, calls the appropriate TLS service protocol methods whenever I/O data needs to be secured.
- The long term goal for the location of the TLS service protocol is within the Foundation framework. In the short term, the protocol can live within the transport management framework.


