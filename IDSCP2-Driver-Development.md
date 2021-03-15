The IDSCP2 protocol allows the user to add custom driver implementations for the Remote Attestation, 
the DAPS, as well as for the underlying Secure Channel. For this purpose, the IDSCP2 core provides
several interfaces and abstract classes in *de.fhg.aisec.ids.idscp2/idscp_core/drivers/* that have
to be implemented by the custom driver implementations. 

The following section shows for each type of driver how you can provide your own implementation:

## Custom Daps Driver
The Daps Driver is responsible for providing and verifying dynamic attribute tokens (DATs).
In the case of the IDS, a dynamic attribute is a signed JsonWebToken (JWT) that contains information
about the DAT issuer, the subject, the expiration date, the audience and so on. Further, it could 
contain custom security relevant information about the IDSCP2 peer. The DapsDriver is a fixed
component of the IDSCP2 finite state machine and cannot be omitted.

```kotlin
package de.fhg.aisec.ids.idscp2.idscp_core.drivers
 
import java.security.cert.X509Certificate
  
interface DapsDriver {

      /*
       * Receive a token from the DapsDriver
       */
      val token: ByteArray
  
      /*
       * Verify a Daps token
       * The FSM will pass the peer's transport certificate as argument if available
       * 
       * Return the number of seconds, the DAT is valid
       */
      fun verifyToken(dat: ByteArray, peerCertificate: X509Certificate?): Long
}
```

Implementing a custom DAPS driver requires to implement the DapsDriver interface that ensures your
implementation to provide a ***token*** getter method, as well as a ***verifyToken*** function.

The ***token()*** should return the currently valid DAT of the local IDSCP2 peer. In the case of our 
DefaultDapsDriver implementation, it does the following steps:
- Create an empty DAT (JWT) 
- Set JWT values: Issuer, Subject, claims, ExpirationDate, IssuedAt, NotBeforeDate, Audience
- Sign the DAT with your private key
- Optional: Request the DAPS, which will attestate the DAT with further claims
- Return the DAT as a ByteArray


The ***verifyToken(dat)*** should check if all parts of the given DAT
are valid and trusted and should return the remaining validity period of the DAT in seconds.
It should do the following steps:
- Verify DAT signature
- Verify DAT attributes (Issuer, Subject, Audience, ...)
- Verify if DAT is currently valid
- Check Algorithm Constraints
- Verify custom security requirements if available
- Calculate and return the remaining validity time

If the DAT is not valid, you can return a non-positive value or throw an exception. Both will
be recognized by the finite state machine and will shut down the connection immediately.

Default implementations of the IDS DapsDriver, as well as the NullDaps can be found in 
*de.fhg.aisec.ids.idscp2/default_drivers/daps/*.

If you do not want to use a DAPS driver in your project, you can make use of a custom DapsDriver
with a ***verifyToken*** implementation that will always return a positive validity period. Keep in
mind that after the DAT validity period is over, the DAT timer triggers a DAT timeout in the FSM that
will trigger a new DAT request, as well as the re-attestation of the remote peer. 
If the usage of a DapsDriver is not wanted, the return value of the ***verifyToken*** method should be
chosen quite large (e.g. one day = 86.400 seconds) to keep the overhead small.

## Custom Secure Channel
The underlying secure channel is responsible for providing an integrity-, authenticity- and confidentiality-protected
communication channel between a client and a server, such as TLSv1.3 does. It consists of three different interfaces,
which is why it is a bit more complex than the DapsDriver:

### SecureChannelDriver
```kotlin
package de.fhg.aisec.ids.idscp2.idscp_core.drivers

import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2Connection
import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.Idscp2Configuration
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_server.SecureChannelInitListener
import de.fhg.aisec.ids.idscp2.idscp_core.secure_channel.SecureChannel
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_server.ServerConnectionListener
import java.util.concurrent.CompletableFuture

interface SecureChannelDriver<CC : Idscp2Connection, SecureChannelConfiguration> {
    /**
     * Asynchronous method to create a secure connection to a secure server
     */
    fun connect(connectionFactory: (SecureChannel, Idscp2Configuration) -> CC,
                     configuration: Idscp2Configuration,
                     secureChannelConfig: SecureChannelConfiguration): CompletableFuture<CC>

    /**
     * Starting a secure channel listener
     */
    fun listen(channelInitListener: SecureChannelInitListener<CC>,
               serverListenerPromise: CompletableFuture<ServerConnectionListener<CC>>,
                secureChannelConfig: SecureChannelConfiguration): SecureServer
}
```
The SecureChannelDriver is responsible for providing the IDSCP2 core an API
for connecting to a secure listener (client-side) and listening for incoming connections (server-side).

#### SecureChannelDriver - Client
Let's start with the client side. The goal of the ***connect*** method is to create a 
client for a secure channel implementation (e.g. TLSv1.3 client) that will connect to a secure channel server
(e.g. TLSv1.3 server). After the secure channel has been established, the new IDSCP2 connection will be initiated
and will be passed to the IDSCP2 core. 

As arguments, we have a connectionFactory, an Idscp2Configuration and the generic SecureChannelConfiguration given 
by the IDSCP2 core. The connectionFactory together with the Idscp2Configuration can be used for creating 
the IDSCP2 connection when the secure connection has been established, while the SecureChannelConfig
could contain customized information for the SecureChannelDriver to create a secure channel. Such information
could be the server address and port, keystore and truststore information, certificates, and so on.
The return value of the ***connect*** function is a completable future that passes the new Idscp2Connection
to the core.

In the following you can see a very basic example how the ***connect*** functionality of the SecureChannelDriver
could be implemented. First of all we have to create the connection future, which we will immidiately return.
In a separate thread or any further async kotlin mechanism (e.g. coroutines) we create the client
and connect it to the secure channel server using the generic SecureChannelConfig. 
When the client has been connected, we create the secure channel and register it to the client.
The SecureChannel class can be found in *de.fhg.aisec.ids.idscp2/idscp_core/secure_channel/*. It is the 
interface between the finite state machine of the IDSCP2 and the secure channel endpoint (client or server thread).
For this, it is important that your client implements the SecureChannelEndpoint interface, which is described below.
Using the secure channel and the Idscp2Configuration given as an argument, the Idscp2Connction can be
created via the connection factory. This IDSCP2 connection will then complete the future.

In the end we have to check if the completable future has already been cancelled by the core. If so, we have to
close the connection again to lock the connection and cleanup the secure channel.

The SecureChannel takes optionally the remote peer certificate (X509Certificate) as argument. The
certificate is stored in the FSM and might be used in the DAPS driver for validating the peer against
its DAT.

```kotlin
class CustomSecureChannelDriver<CC: Idscp2Connection> : SecureChannelDriver<CC, CustomScConfig> {
    override fun connect(connectionFactory: (SecureChannel, Idscp2Configuration) -> CC,
                                      configuration: Idscp2Configuration,
                                      secureChannelConfig: CustomScConfig): CompletableFuture<CC> { 
    val connectionFuture = CompletableFuture<CC>() //completable future for Idscp2Connection 
    // async create and connect secure channel client to secure channel server given the secureChannelConfig
    thread {
        val client = MyClient()
        val secureSession = client.connect(secureChannelConfig.host, secureChannelConfig.serverPort)
        // securely connected
        // create a secure channel with client as endpoint and optionally the peer certificate and register it to the client
        val secureChannel = SecureChannel(client, secureSession.peerCertificate)
        client.setSecureChannel(secureChannel)
        // create the connection via the connection factory
        val idscp2Connection = connectionFactory(secureChannel, configuration)
        // complete the feature
        connectionFuture.complete(idscp2Connection)
        if (connectionFuture.isCancelled)
            idscp2Connection.close()
    }

    return connectionFuture
    }
}
```

#### SecureChannelDriver - Server
On the server side it is a little different, since there is not the one Idscp2Connection
that will be created, but a server that will accept multiple connections.

Similar to the client-side we have a custom secure channel config, which provides
information required for establishing secure connections, such as the 
port, the server should listen on, or keystore and truststore information.
Further, it takes a SecureChannelInitListener, which is actually the Idscp2Server, that expects a 
new SecureChannel for every new ServerThread that has been started at the SecureServer. 

The idea is the following: 
The ***listen*** method creates a new secure server and runs it asynchronous. 
```kotlin
class CustomSecureChannelDriver<CC : Idscp2Connection> : SecureChannelDriver<CC, CustomScConfig> {
    override fun listen(channelInitListener: SecureChannelInitListener<CC>,
               serverListenerPromise: CompletableFuture<ServerConnectionListener<CC>>,
                secureChannelConfig: CustomScConfig): SecureServer {
        // create a secure server and run it async
        val secureServer = MySecureServer(secureChannelConfig, channelInitListener, serverListenerPromise)
        thread {
          secureServer.run()
        }
        return secureServer
    }
}
```
The driver developer himself is responsible for handling new secure connections at the secure server.
Usually, any kind of ServerConnectionThread is created whenever a new client connects to the secure server.
Such a ServerConnectionThread, which has to implement again the SecureChannelEndpoint interface, 
first waits until the secure handshake is done and thus the connection is trusted.

```kotlin
class CustomSecureServerThread<CC : Idscp2Connection> internal constructor(
        private val connectionSocket: Socket,
        private val channelInitListener: SecureChannelInitListener<CC>,
        private val serverListenerPromise: CompletableFuture<ServerConnectionListener<CC>>) :
        Thread(), SecureChannelEndpoint {
    
    private lateinit var secureChannel:  SecureChannelListener
    // run method of server thread
    override fun run() {
        //TODO do any kind of secure handshake ...
        // securely connected
        // create a secure channel with the server thread itself as endpoint and the optional peer cert
        secureChannel = SecureChannel(this, secureSession.peerCertificate) 
        channelInitListener.onSecureChannel(secureChannel, serverListenerPromise) // create the new idscp connection
        
        //TODO start listening on socket and listening on messages from the secureChannel ...
    }    
    
}
```
Afterwards, it will create a SecureChannel using itself as SecureChannelEndpoint.
The Idscp2Connection will be created by the ***channelInitListener.onSecureChannel***. The SecureChannelInitListener
is actually the Idscp2Server, which will register the new Idscp2Connection to a connection listener via
the serverListenerPromise.

See *.fhg.aisec.ids.idscp2/default_drivers/secure_channel/tlsv1_3/NativeTlsDriver.kt*
for an example implementation of the SecureChannelDriver interface.

### SecureChannelEndpoint

We have already seen the use of the SecureChannelEndpoint interface above. 
Both, the secure client and the secure server thread (they are the instances
that acts, i.e. read and write, directly on the connection socket) have to implement
the SecureChannelEndpoint interface. The idea is that the SecureChannelEndpoint abstracts the underlying
socket, such that the IDSCP2 core is able to interact with any kind of underlying endpoint.

```kotlin
package de.fhg.aisec.ids.idscp2.idscp_core.drivers

interface SecureChannelEndpoint {

    /**
     * API to close the secure channel endpoint
     */
    fun close()

    /**
     * Send data from the secure channel endpoint to the peer connector
     * return true when data has been sent, else false
     */
    fun send(bytes: ByteArray): Boolean

    /**
     * check if the endpoint is connected
     */
    val isConnected: Boolean
}
```

The interface implements a ***close*** methode, that should close the socket and stop the endpoint thread, 
a ***send*** method, that should write messages to the socket and an ***isConnected*** method, that checks if the secureChannelEndpoint
is still connected. 

When receiving new messages from the remote peer on the socket, as well as EOF or any socket error
this should be passed directly to the SecureChannelListener.

In *.fhg.aisec.ids.idscp2/default_drivers/secure_channel/tlsv1_3/* see
*client/TLSClient.kt* as well as *server/TLSServerThread* as examples
for the SecureChannelEndpoint.


### SecureServer 
The last component of the secure channel driver is the secure server, which 
we have also seen above in the ***SecureChannelDriver.listen*** method. The implementation is
largely up to the driver developer. He should only provide a ***safeStop***
functionality to stop the secure server, as well as an ***isRunning*** checker.
```kotlin
package de.fhg.aisec.ids.idscp2.idscp_core.drivers

interface SecureServer {
    /**
     * Terminate the secure server
     */
    fun safeStop()

    /**
     * Check if the secure server is running
     */
    val isRunning: Boolean
}
```

See *.fhg.aisec.ids.idscp2/default_drivers/secure_channel/tlsv1_3/server/TLSServer* for a
secure TLSv1.3 server example.

## Custom Rat Driver
The Remote Attestation Drivers (RAT) are responsible for verifying whether the remote peer is in a trusted
state (RAT Verifier), as well as proving its own trusted state to the remote peer (RAT Prover).
This is why every IDSCP2 peer needs two different RAT drivers, one verifier and one prover.

Since both drivers implements almost the same abstract class (up to some small restrictions to avoid a prover
to act as a verifier and the other way around), only the implementation of a custom
RAT verifier will be shown here.

The RatVerifierDriver and RatProverDriver classes extend a Thread, thus the custom drivers have to override
the ***run*** method, which is actually the entry point of the drivers. When the IDSCP2 core
receives a new message from the remote peers counterpart driver for the local peers RAT driver, it will delegate the message
to the local RAT driver via the ***delegate*** function. The driver itself can communicate with its counterpart driver, 
as well as with the FSM by sending RAT messages to the finite state machine via the RatVerifierFsmListener.
There are three types of messages: The RAT_OK, the RAT_FAILED and the RAT_MSG. The RAT_OK signals 
the FSM that (in case of the verifier) that the RAT verification succeed. In contrast, the RAT_FAILURE 
message tells the FSM that the verification has failed and thus the peer is not trusted.
The RAT_MSG is not directed to the FSM, but to the remote counterpart RAT driver to exchange verification data.

When the driver has been terminated by the IDSCP2 core via the ***terminate*** functionality, it is not longer able to send
messages to the finite state machine, since the FSM will block the driver. Therefore, the driver
should terminate itself and exit the ***run*** method.

The generic RAT config is full customized and provides the possibility to initialize
the RAT driver with pre-configured attributes. The ***setConfig*** method will always be called
after the driver has been created and before it will be executed.  

```kotlin
package de.fhg.aisec.ids.idscp2.idscp_core.drivers
import de.fhg.aisec.ids.idscp2.idscp_core.fsm.fsmListeners.RatVerifierFsmListener

abstract class RatVerifierDriver<in VC>(protected val fsmListener: RatVerifierFsmListener) : Thread() {
    protected var running = true

    /**
     * Delegate the IDSCP2 message to the RatVerifier driver
     */
    open fun delegate(message: ByteArray) {}

    /**
     * Terminate and cancel the RatVerifier driver
     */
    fun terminate() {
        running = false
        interrupt()
    }

    open fun setConfig(config: VC) {
        LOG.warn("Method 'setConfig' for RatVerifierDriver is not implemented")
    }
}
```

In the following, the implementation of the NullRat is shown.
The basic concept is a message queue that stores the delegated messages from the FSM.
When a message from the remote RAT prover is available, we will return a RAT_MSG and will
then send the RAT Ok the FSM, which means that the verification succeed.

```kotlin
package de.fhg.aisec.ids.idscp2.default_drivers.rat.`null`

import de.fhg.aisec.ids.idscp2.idscp_core.drivers.RatVerifierDriver
import de.fhg.aisec.ids.idscp2.idscp_core.fsm.InternalControlMessage
import de.fhg.aisec.ids.idscp2.idscp_core.fsm.fsmListeners.RatVerifierFsmListener
import java.util.concurrent.BlockingQueue
import java.util.concurrent.LinkedBlockingQueue

class NullRatVerifier(fsmListener: RatVerifierFsmListener) : RatVerifierDriver<Unit>(fsmListener) {
    private val queue: BlockingQueue<ByteArray> = LinkedBlockingQueue()
    override fun delegate(message: ByteArray) {
        queue.add(message)
    }

    override fun run() {
        try {
            queue.take() //wait until message available
        } catch (e: InterruptedException) {
            // was interrupted, check if we are still running or if the FSM terminates us
            if (running) {
                // we are still running, tell the FSM we have failed
                fsmListener.onRatVerifierMessage(InternalControlMessage.RAT_VERIFIER_FAILED)
            }
            return
        }
        // send one message and the RAT OK
        fsmListener.onRatVerifierMessage(InternalControlMessage.RAT_VERIFIER_MSG, "".toByteArray())
        fsmListener.onRatVerifierMessage(InternalControlMessage.RAT_VERIFIER_OK)
    }
}
```

The driver developer has to make sure, that his RAT prover and RAT verifier implementations
can communicate with each other correctly. Otherwise, the IDSCP2 handshake will fail.