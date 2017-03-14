#Security Service

## Introduction

This proposal presents the design of _TLS service_, which is a collection of Swift APIs that provide SSL/TLS functionality with a consistent API surface for both Apple and Linux platforms. 

In this proposal, we use the term TLS to refer to all SSL and TLS related functionality.



## Motivation

Currently each platform has its own set of security libraries with either Swift or C APIs. 

There is also no standard Swift TLS library which has resulted in projects implementing  their own Swift security functionality or using their security library of choice (be it OpenSSL, LibreSSL, etc.). This has resulted in fragmentation of the space as well as incompatibility of project dependencies if more than one security package is needed by different modules.

This motivates the necessity for defining a standard set of protocols that define the behavior of the TLS service and how the application and the server and networking layers beneath it interact with the TLS service.


## Proposed solution

We propose a new model that is easily extensible and supports a plug-n-play architecture which allows an application to choose the underlying security framework of choice. 

![alt text](https://raw.githubusercontent.com/gtaban/blogs/master/TLSServiceArchitecture.png "Architecture of TLSService modules")

### Assumptions

At the time of writing this proposal, the Swift Server API work groups has not yet defined any standards for the base networking or application layers. For our design, we have assumed a network stack that consists of:

- Foundation's transport layer (CFNetwork, BSD Sockets, or Open Transport) 
- socket management
- HTTP request management 
- (Optional) Web server
- Application

Our security stack is composed of:

- Underlying security framework (SecureTransport library in Security.framework  on Apple and OpenSSL or alternatives on Linux)
- TLS service

In our diagram we omit both Foundation's transport libraries as well as the underlying security library and show the relationship between the TLS service protocol and the various application layers.


### Design Description
TLS service protocol defines a set of methods that interface with the transport layer (for example, sockets).

These methods are implemented by the TLS service which in turn uses its choice an of underlying security library. As an example, the TLS service uses SecurityTransport library on Apple platform and OpenSSL or alternatively LibreSSL on Linux. 

If an application requires TLS for its use case, it creates a TLS service object and configures it based on its requirements. Note that the configuration of the TLS service is outside the scope of the current document.

The application then passes the TLS service object to its lower level frameworks that deal with networking and communication. Each lower level framework maintains an optional instant variable of type TLS service protocol. If the optional variable exists, it is further passed down until it gets to the lowest level that deals with the Swift socket APIs (in the diagram above, this is the HTTP Management layer). When this layer creates the connecting sockets, it assigns the TLS service object to the Socket delegate. The Swift socket layer is then responsible for calling the TLS service protocol methods that handle the TLS functionality at the appropriate times. 
 

### Benefits

There are several benefits provided by this model:

- It allows the application to choose the underlying security library of choice for their TLS by passing in the appropriate TLS service object. 
  
   As long as the TLS service object conforms to the specification of the TLS service protocol, it can implement its methods using any underlying security library. The application can then create the TLS service object of its choice and simply pass the object to the appropriate methods that invoke transport-level functionality. 
   
   This model is specially important for projects that involve multiple dependencies that use conflicting security libraries (e.g., LibreSSL and OpenSSL which share the same API surface and building a project with both of these libraries results in link errors). Now the application can choose its own TLS service object and pass it to all frameworks that require TLS.

- Application do not have dependency on underlying security framework if TLS is not enabled.

  By mandating socket/HTTP management layers to be dependent only on the TLS service protocol, they are agnostic of the underlying security framework and hence do not need to have them in the project if they are not used.

- It allows users to use the same TLS library for both client and server applications.

  This is especially important for many server side applications which need to behave as both clients and servers depending on external services they connect to. Also by decoupling the TLS service client and server objects, debugging an application with multiple types of connections will be made easy.

- The underlying socket/HTTP management layers are agnostic of the underlying security functionality and simply hand off the TLS service object to the socket delegate.

- The underlying socket management layer is responsible for supporting blocking and non-blocking connections as long as the underlying security library supports both types of connection.




### A note on creating a Swifty TLS service

There are a number of cases in low-level systems programming where similar functionality is differentiated by a flag or type, such as `SOCK_STREAM` vs. `SOCK_DGRAM` or `SSLv23_server_method` vs. `SSLv23_client_method`.  

The latter is an example of our OpenSSL differentiates between the server side or the client side of a TLS connection. The client is the initiator of a TLS session. The server is the entity that accepts the requests for TLS sessions made by clients. Depending on the connection type (server or client), different actions and policies are supported such as cipher suite selection during a TLS handshake, request for handshake renegotiations and trust validation policies for server name and certificates.  


While using a flag or a type is convenient in languages such as `C`, it's not very `Swift`-like and does not adhere to the single responsibility principle [3] of object oriented paradigm. We recommend that different interfaces should be created that decouple the client and server concepts as far as the higher application layers are concerned. This is generalized approach that can be adopted by the other layers of the application stack.



<!--Although using a server flag to specify different behaviors is a normal paradigm in C, this is not in the true spirit of object oriented design and the single responsibility principle [3], inherent to Swift. Since the responsibility of the class changes depending on the value of the server flag, this violates the single responsibility principle and a better design that we recommend, will be separating out the TLS service objects that implement the TLS service protocol into two separate classes: TLS client service and TLS server service. This leads to a more maintainable and debuggable code. Given the large amount of code overlap between a TLS client service and a TLS server service, it is further recommended that a generic TLS service object be first implemented and then different interfaces get created to decouple the client and server concepts as far as the rest of the networking and application layers are concerned.

, so we recommend that different interfaces should be created instead -->


## Detailed design

### TLS service protocol

The TLSServiceDelegate protocol describes the methods that the transport layer calls to handle transport-level events for the TLS service object.


#### onClientCreate
This will be called when a client socket connection is created and appropriate TLS connection needs to be configured, including context creation, the handshake and connection verification. 

This is a client only method.


```
	///
	/// Setup the contexts and process the TLSService configurations (certificates, etc).
	///

	func onClientCreate() throws
	
```

#### onServerCreate
This will be called when a server socket connection is created and appropriate TLS connection needs to be setup, including context creation, the handshake and connection verification. 

This is a server only method.

```
	///
	/// Setup the contexts and process the TLSService configurations (certificates, etc).
	///

	func onServerCreate() throws
	
```

### onDestroy
This will be called when a socket connection is closed and any remaining TLS context needs to be destroyed. 

```
	///
	/// Destroy any remaining contexts 
	///
	func onDestroy()
```

### onAccept
This will be called once a socket connection has been accepted, to setup the TLS connection, do the handshake and connection verification. 

```
	///
	/// Processing on acceptance from a listening socket
	/// 
	///
	/// - Parameter socket:	The connected Socket instance.
	///
	func onAccept(socket: Socket) throws
	
```

### onConnect
This will be called once a socket connection has been made, to setup the TLS connection, do the handshake and connection verification. 

```
	///
	/// Processing on connection to a listening socket
 	///
	/// - Parameter socket:	The connected Socket instance.
	///
	func onConnect(socket: Socket) throws
	
```

### onSend
This will be called when data is to be written to a socket. The input data buffer is written to the TLS connection associated with that socket.

```
	///
	/// Low level writer
	///
	/// - Parameters:
	///		- buffer:		Buffer pointer.
	///		- bufSize:		Size of the buffer.
	///
	///	- Returns the number of bytes written. Zero indicates TLS shutdown, less than zero indicates error.
	///
	func onSend(buffer: UnsafeRawPointer, bufSize: Int) throws -> Int
	
```

### onReceive
This will be called when data is to be read from a socket. Encrypted data is read from TLS connection associated with that socket and decrypted and written to the buffer passed in.

```
	///
	/// Low level reader
	///
	/// - Parameters:
	///		- buffer:		Buffer pointer.
	///		- bufSize:		Size of the buffer.
	///
	///	- Returns the number of bytes read. Zero indicates TLS shutdown, less than zero indicates error.
	///
	func onReceive(buffer: UnsafeMutableRawPointer, bufSize: Int) throws -> Int
	
```

## Non-goals

This proposal:

- DOES NOT describe the TLS service configuration, which includes information on certificate types, formats and chains, cipher suites, etc. We expect this to be specified in a future proposal.
- DOES NOT describe the TLS service trust policies, which define trust and validation policies of the incoming connection. We expect this to be specified in a future proposal.
- DOES NOT describe compatibility of the TLS service with non-Foundation based transport protocols on the Linux platform. It is possible to extend the current TLS service protocol to support other types of protocols in future proposals.


## Source compatibility

The proposed change is designed to be as non-breaking as possible, however it does place several assumptions on the rest of the network/application stack. 

- The application layer must import and instantiate a TLS service object which implements the TLS service protocol if it wants to enable TLS service.
- Every framework layer above the socket layer but below the application layer (which includes networking and server frameworks) has an optional object that implements the TLS service protocol. If the TLS object exists, it is passed down to each layer below.
- The networking layer which sets up the socket layer assigns the TLS object to the socket delegate if the TLS object exists. 
- The socket layer which sets up the socket communication and deals with the low level C sockets calls the appropriate TLS service protocol methods whenever socket data needs to be secured.
- The long term goal for the location of the TLS service protocol is within the Foundation framework. In the short term, the protocol can live within the socket framework.



## Alternatives considered

One of the original requirements we put on our list was making the ssl library be agnostic of transport layer mechanism. This is what SSLEngine in Java does by operating on byte streams with the idea that:
"By separating the SSL/TLS abstraction from the I/O transport mechanism, the SSLEngine can be used for a wide variety of I/O types, such as non-blocking I/O (polling), selectable non-blocking I/O, Socket and the traditional Input/OutputStreams, local ByteBuffers or byte arrays, future asynchronous I/O models , and so on." [1]


The Stream object in Foundation manages TLS streams implicitly via flags. [2]


SecureTransport assumes some kind of connection for its input via SSLSetConnection, based on Foundation's support CFNetwork, BSD Sockets, or Open Transport.
SecureTransport allows the app more flexibility in terms of configuration setup of the TLS connection. This level of configuration is  desired by many Server applications and therefore is preferred.

So it makes sense for us not to use Streams and instead use the SecureTransport approach and limit transport layer options to those supported by Foundation.

## References

1 - https://docs.oracle.com/javase/7/docs/api/javax/net/ssl/SSLEngine.html

2 - https://developer.apple.com/reference/foundation/nsstream

3 - https://web.archive.org/web/20150202200348/http://www.objectmentor.com/resources/articles/srp.pdf
