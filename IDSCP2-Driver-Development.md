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

The ***token*** getter should return the currently valid DAT of the local IDSCP2 peer. In the case of our 
AisecDaps implementation, it does the following steps:
- Create an empty DAT (JWT) 
- Set JWT values: Issuer, Subject, claims, ExpirationDate, IssuedAt, NotBeforeDate, Audience
- Sign the DAT with your private key
- Request the DAPS, which will attestate the DAT with further claims
- Return the DAT as a ByteArray


The ***verifyToken(dat, peerCertificate?)*** should check if all parts of the given DAT
are valid and trusted and should return the remaining validity period of the DAT in seconds.
It should do the following steps:
- Verify DAT signature
- Verify DAT attributes (Issuer, Subject, Audience, ...)
- Verify if DAT is currently valid
- Check Algorithm Constraints
- Optional: Verify the peer certificate against allowed fingerprints if available
- Optional: Verify custom security requirements if available
- Calculate and return the remaining validity time

If the DAT is not valid, you should throw a DatException. This will be recognized by the finite state machine and will shut down the connection immediately.

Default implementations of the AISEC DapsDriver, as well as the NullDaps can be found in 
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

import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.Idscp2Configuration
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2Connection
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_server.ServerConnectionListener
import de.fhg.aisec.ids.idscp2.idscp_core.fsm.FSM
import java.util.concurrent.CompletableFuture

interface SecureChannelDriver<CC : Idscp2Connection, SecureChannelConfiguration> {
    /**
     * Asynchronous method to create a secure connection to a secure server
     */
    fun connect(
        connectionFactory: (FSM, String) -> CC,
        configuration: Idscp2Configuration,
        secureChannelConfig: SecureChannelConfiguration
    ): CompletableFuture<CC>

    /**
     * Starting a secure channel listener
     */
    fun listen(
        connectionListenerPromise: CompletableFuture<ServerConnectionListener<CC>>,
        secureChannelConfig: SecureChannelConfiguration,
        serverConfiguration: Idscp2Configuration,
        connectionFactory: (FSM, String) -> CC
    ): SecureServer
}
```
The SecureChannelDriver is responsible for providing the IDSCP2 core an API
for connecting to a secure listener (client-side) and listening for incoming connections (server-side).

#### SecureChannelDriver - Client
Let's start with the client side. The goal of the ***connect*** method is to create a 
client for a secure channel implementation (e.g. TLSv1.3 client) that will connect to a secure channel server
(e.g. TLSv1.3 server). 

As arguments, we have a connectionFactory, an Idscp2Configuration and the generic SecureChannelConfiguration given by the IDSCP2 core. The connectionFactory together with the Idscp2Configuration can be used for creating 
the IDSCP2 connection when the secure connection has been established, while the SecureChannelConfig
could contain customized information for the SecureChannelDriver to create a secure channel. Such information
could be the server address and port, keystore and truststore information, certificates, and so on.
The return value of the ***connect*** function is a completable future that passes the new Idscp2Connection
to the core when the IDSCP2 handshake has succeed.

In the following you can see an example how the ***connect*** functionality of the SecureChannelDriver could be implemented. First of all we have to create the connection future, which we will immediately return. In a separate thread or any further async Kotlin mechanism (e.g. coroutines) we create the client and connect it to the secure channel server using the generic SecureChannelConfig. 
When the client has been connected, the connection future has not been cancelled by the user and the hostname verification succeed, we create the secure channel and register it to the client. The SecureChannel class can be found in *de.fhg.aisec.ids.idscp2/idscp_core/secure_channel/*. It is the interface between the finite state machine of the IDSCP2 and the secure channel endpoint (client or server thread). For this, it is important that your client implements the SecureChannelEndpoint interface, which is described below.

Further, the SecureChannel optionally takes the remote peer certificate (X509) as argument. The certificate is stored in the FSM and might be used in the DAPS for validating the peer against the DAT. 

If any of the previous steps have failed, the connectionFuture has to be completed exceptionally and the client has to be cleaned up.

Using the secure channel, the Idscp2Configuration, the connectionFactory and the connectionFuture, the IDSCP2 handshake can be initiated using the AsyncIdscp2Factory, located at *de.fhg.aisec.ids.idscp2/idscp_core/fsm/*, using the **initiateIdscp2Connection** method. It returns false, if the connection has been cancelled by the user, otherwise true. When it fails, the secure channel will automatically be cleaned up and the future will be completed exceptionally, so the driver does not have to do anything else. When it succeeds, the driver can start the client's socket listener thread.

```kotlin
class CustomSecureChannelDriver<CC: Idscp2Connection> : SecureChannelDriver<CC, CustomScConfig> {
    override fun connect(
        connectionFactory: (FSM, String) -> CC,
        configuration: Idscp2Configuration,
        secureChannelConfig: CustomScConfig
    ): CompletableFuture<CC> { 
        
        // create the connection future for async connect
        val connectionFuture = CompletableFuture<CC>() //completable future for Idscp2Connection 
        
        // run async connect
        thread {

            try {
                // create TLS client and connect to server, throws exception on failure
                val client = MyClient()
                val secureSession = client.connect(secureChannelConfig.host, secureChannelConfig.serverPort)
            
                if (connectionFuture.isCancelled) {
                    client.disconnect()
                    return
                }

                // hostname verification, throws exception if not trusted
                customHostnameVerification(secureChannelConfig.host, secureSession.peerCertificate)

                // create a secure channel with client as endpoint and optionally the peer certificate 
                val secureChannel = SecureChannel(client, secureSession.peerCertificate)
                
                // register secure channel as communication listener to the client
                client.setSecureChannel(secureChannel)

                // init the IDSCP2 handshake using the AsyncIdscp2Factory
                // AsyncIdscp2Factory will return false, if the connectionFuture has been cancelled by the user
                // on failure it will close the secure channel
                // this is the correct time for starting the client's socket listener thread
                val success = AsyncIdscp2Factory.initiateIdscp2Connection(
                                secureChannel, configuration, connectionFactory, connectionFuture)

                if (success) {
                    client.startInputListener()
                }
            } catch (e: Exception) {
                // something went wrong, close connection and complete future exceptionally
                client.disconnect()
                connectionFuture.completeExceptionally(
                    Idscp2Exception("Secure connection failed", e)
                )
            }
        }

        // return future immediately
        return connectionFuture
    }
}
```

#### SecureChannelDriver - Server
On the server side it is a little different, since a server that will accept multiple connections. Similar to the client-side we have the custom secure channel configuration, which provides information required for establishing secure connections, such as the port, the server should listen on, or Keystore and Truststore information, the IDSCP2 configuration that is used for initiating the FSM of a connection and a IDSCP2 connectionFactory. Further, it takes a future for a ServerConnectionListener, which is actually the future Idscp2Server that is listen on new IDSCP2 connections.

The idea is the following: 
The ***listen*** method creates a new secure server and runs it asynchronous. 
```kotlin
class CustomSecureChannelDriver<CC : Idscp2Connection> : SecureChannelDriver<CC, CustomScConfig> {

    override fun listen(        
        connectionListenerPromise: CompletableFuture<ServerConnectionListener<CC>>,
        secureChannelConfig: NativeTlsConfiguration,
        serverConfiguration: Idscp2Configuration,
        connectionFactory: (FSM, String) -> CC
    ): SecureServer {
        // create a secure server and run it async
        val secureServer = MySecureServer(connectionListenerPromise, secureChannelConfig, serverConfiguration, connectionFactory)
        secureServer.runAsync()
        return secureServer
    }

}
```
The driver developer himself is responsible for handling new secure connections at the secure server.
Usually, any kind of ServerConnectionThread is created whenever a new client connects to the secure server. Further, for each new ServerConnectionThread, the secure server has to create a connectionFuture. It will be passed to the ServerConnectionThread and should register the new Idscp2Connection to the Idscp2Server on completion using the ServerConnectionListener:

```kotlin
class CustomSecureServer<CC : Idscp2Connection>(
    private val connectionListenerPromise: CompletableFuture<ServerConnectionListener<CC>>,
    private val scConfig: CustomScConfig,
    private val serverConfiguration: Idscp2Configuration,
    private val connectionFactory: (FSM, String) -> CC
) {
    
    ...

    fun runAsync() {
        Thread {
            while(isRunning) {
                
                // wait for new incoming connections
                val connectionSocket = serverSocket.accept()
             
                // create a connectionFuture that registers the connection at the Idscp2Server
                val connectionFuture = CompletableFuture<CC>()
                connectionFuture.thenAccept { connection ->
                    connectionListenerPromise.thenAccept { listener ->
                        listener.onConnectionCreated(connection)
                    }
                }.exceptionally {
                    // no IDSCP2 connection has been created
                    LOG.warn("Idscp2Connection creation failed: {}", it)
                    null
                }

                // created the ServerConnectionThread
                val thread = SecureConnectionThread(connectionSocket, scConfig, connectionFuture, serverConfig, connectionFactory)
                thread.run()
            }
        }
    }

    ... 
}
```

Such a ServerConnectionThread, which has to implement again the SecureChannelEndpoint interface, first waits until the secure handshake is done and thus the connection is trusted. Then the implementation is very similar to the client-side. First we create the secure channel using the ServerConnectionThread as endpoint and the peerCertificate as optional parameter. Then the Idscp2Connection creation is initiated via the AsyncIdscp2Factory. On error, the connectionFuture will be completed exceptionally and the ServerConnectionThread will terminate:

```kotlin
class CustomSecureServerThread<CC : Idscp2Connection> internal constructor(
    private val socket: CustomSocket,
    private val connectionFuture: CompletableFuture<CC>,
    private val scConfig: CustomScConfig,
    private val configuration: Idscp2Configuration,
    private val connectionFactory: (FSM, String) -> CC
) : Thread(), SecureChannelEndpoint {
    
    private lateinit var secureChannel:  SecureChannelListener

    // run method of server thread
    override fun run() {

        try {
            // do handshake and wait until channel is secure
            val secureSession = socket.doHandshake()

            // TODO optionally hostname verification on server-side via static IPs, DNS, TTP

            // (lateinit) create a secure channel with this thread as endpoint and optionally the peer certificate
            secureChannel = SecureChannel(this, secureSession.peerCertificate)
              
            // init the IDSCP2 handshake using the AsyncIdscp2Factory
            // AsyncIdscp2Factory will return false, if the connectionFuture has been cancelled by the user
            // on failure it will close the secure channel
            // this is the correct time for starting the client's socket listener thread
            val success = AsyncIdscp2Factory.initiateIdscp2Connection(
                                secureChannel, configuration, connectionFactory, connectionFuture)

            if (!success) {
                return
            }
        } catch (e: Exception) {
            // something went wrong, close connection and complete future exceptionally
            this.disconnect()
            connectionFuture.completeExceptionally(
                Idscp2Exception("Secure connection failed", e)
            )
            return
        }

        // TODO start listening on socket and pass messages to the secureChannel
    }    
    
}
```

See *.fhg.aisec.ids.idscp2/default_drivers/secure_channel/tlsv1_3/NativeTlsDriver.kt*
for an example implementation of the SecureChannelDriver interface.

### SecureChannelEndpoint

We have already seen the use of the SecureChannelEndpoint interface above. Both, the secure client and the secure server thread (they are the instances that acts, i.e. read and write, directly on the connection socket) have to implement the SecureChannelEndpoint interface. The idea is that the SecureChannelEndpoint abstracts the underlying socket, such that the IDSCP2 core is able to interact with any kind of underlying endpoint.

```kotlin
package de.fhg.aisec.ids.idscp2.idscp_core.drivers

interface SecureChannelEndpoint {

    /**
     * API to close the secure channel endpoint
     */
    fun close()

    /**
     * Send data from the secure channel endpoint to the peer connector
     *
     * ATTENTION: The developer must ensure not to trigger the FSM by another event from the thread
     * that actually executes the SecureChannelEndpoint.send(bytes) method, but must simply return
     * true or false, regarding the success of this method. The issue which occurs by using the thread
     * of the send() method is that the FSM is blocked within a transition until send() returns
     * and the the following state of the FSM depends on the return value. When this thread would trigger
     * the FSM again within the current transition then first the new transition would be executed
     * but than it would be overwritten by the current one. To avoid this case of misuse, the FSM
     * will throw a Runtime Exception to let the driver developer know, that this is not a good idea.
     * For more information, see the checkForFsmCycles() method within the FSM
     *
     * return true when data has been sent, else false
     */
    fun send(bytes: ByteArray): Boolean

    /**
     * check if the endpoint is connected
     */
    val isConnected: Boolean
}
```

The interface implements a ***close*** methode, that should close the socket and stop the endpoint thread, a ***send*** method, that should write messages to the socket and an ***isConnected*** method, that checks if the secureChannelEndpoint is still connected. 

When receiving new messages from the remote peer on the socket, as well as EOF or any socket error
this should be passed directly to the SecureChannel.

In *.fhg.aisec.ids.idscp2/default_drivers/secure_channel/tlsv1_3/* see *client/TLSClient.kt* as well as *server/TLSServerThread* as examples for the SecureChannelEndpoint.

### SecureServer 
The last component of the secure channel driver is the secure server, which we have also seen above in the ***SecureChannelDriver.listen*** method. The implementation is largely up to the driver developer. He should only provide a ***safeStop*** functionality to stop the secure server, as well as an ***isRunning*** checker.

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
as well as with the FSM by sending RAT messages to the finite state machine via the RatVerifierFsmListener interface.
There are three types of messages: The RAT_OK, the RAT_FAILED and the RAT_MSG. The RAT_OK signals 
the FSM that (in case of the verifier) that the RAT verification succeed. In contrast, the RAT_FAILURE 
message tells the FSM that the verification has failed and thus the peer is not trusted.
The RAT_MSG is not directed to the FSM, but to the remote counterpart RAT driver to exchange verification data.

A RAT driver implementation might have to access the verified and trusted dynamic attribute token from the remote peer, when it contains information that are required for a successful attestation (e.g. TPM golden values). The RatVerifierFsmListener interface provides a getter method for this. (This is not supported by the RAT prover, only for the RAT verifier!)

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