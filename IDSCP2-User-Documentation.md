---------

This documentation should give a short introduction how to configure and run an IDSCP2 peer. It is split into
driver-selection and configuration in the first part, while the second part shows how to use the configuration
to start an IDSCP2 server or initiate an IDSCP2 connection as an IDSCP2 client.

---------

## IDSCP2 Driver Selection and Configuration

The IDSCP2 protocol consists of the following drivers:

* The DAPS driver, responsible for providing and verifying dynamic attribute authorization tokens (DATs)
* The RA prover and RA verifier drivers, responsible for the remote attestation between peers (e.g. TPM)
* The SecureChannel driver, which specifies the underlying protected communication channel (e.g. TLSv1.3)

When starting the IDSCP2 peer, the user has to select the driver implementation that should be used,
before all of those drivers have to be configured by their implementation-specific configurations.

### RA Driver Setup

The remote attestation drivers are used for proving and verifying the trusted state of the local peer to the remote
peer. Your application might support different RA driver implementations, such as TPM or Intel SGX.
The IDSCP2 protocol will then select the RA cipher suite regarding the preferences of the corresponding RA verifier.
The preferences / priorities are determined by the ordering of the RA cipher suites, i.e. the most preferred cipher
suite is placed at index zero, while the least preferred one must be at the end of the cipher suites array.
The cipher suites, together with the RA timeout interval, build the AttestationConfig, which will be used later for
configuring the IDSCP2 peer. The timeout interval specifies the interval between two attestations and will trigger a
re-attestation of the remote peer when the timeout occurs.

We make use of static RA registries that are used by the IDSCP2 protocol for creating new RA driver instances on the
fly when attestation is required. Thus, you have to ensure that all the drivers from your RA cipher suites are
registered to the registries with the same unique identifier. The protocol will then check if the driver is available
in the registry after the RA cipher suite is selected. Some driver implementation might use a driver-specific 
configuration that will be available to your driver before executing.
The RA registries are located at *idscp2/idscp_core/ra_registry/*, the AttestationConfig is located
at *idscp2/idscp_core/api/configuration/*.

In the following you see how it would look like to register an Intel SGX driver and a TPM driver. In this case, the SGX
driver is preferred over the TPM driver for both, the RA verifier and for the RA prover mechanism.
Therefore, the SGX driver has the lower index within the RA suites.
The RA timeout interval is set to one hour, which means that after every full hour, a re-attestation will take place
automatically.
Afterwards, the RA driver implementations are registered to the RA registries, using their unique cipher suite
identifier, their classes, as well as a driver-specific configuration. It is ensured that the later one is passed to
the RA driver immediately after creation and before execution. You can set it to null, if no RA driver configuration is
required for your selected driver.

```kotlin
import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.AttestationConfig
import de.fhg.aisec.ids.idscp2.idscp_core.ra_registry.RaProverDriverRegistry
import de.fhg.aisec.ids.idscp2.idscp_core.ra_registry.RaVerifierDriverRegistry

fun main() {

     // create the attestation config using the builder pattern
     val localAttestationConfig = AttestationConfig.Builder()
                     .setSupportedRaSuite(arrayOf(SGXProverDriver.ID, TPMProverDriver.ID))
                     .setExpectedRaSuite(arrayOf(SGXVerifierDriver.ID, TPMVerifierDriver.ID))
                     .setRaTimeoutDelay(60 * 60 * 1000) // ra timeout in ms, 1 hour
                     .build()

     // register supported RA prover driver implementations to the RA registries
     RaProverDriverRegistry.registerDriver(
            SGXProverDriver.ID, ::SGXProverDriver, sgxProverDriverConfig)

     RaProverDriverRegistry.registerDriver(
                 TPMProverDriver.ID, ::TPMProverDriver, tpmProverDriverConfig)

     // register supported RA verifier driver implementations to the RA registries
     RaVerifierDriverRegistry.registerDriver(
            SGXVerifierDriver.ID, ::SGXVerifierDriver, sgxVerifierDriverConfig)

     RaVerifierDriverRegistry.registerDriver(
                 TPMVerifierDriver.ID, ::TPMVerifierDriver, tpmVerifierDriverConfig)

     ...
}
```

### DAPS Driver Setup

Since the DAPS driver has to be created before the IDSCP2 peer has started, the protocol leaves the configuration of
the DAPS driver completely up to the driver developer. Thus, the DAPS driver generation
only depends on the implementation-specific configuration.

In the following you see how the Fraunhofer-AISEC DAPS (*idscp2/default_drivers/daps/aisec_daps/DefaultDapsDriver*) is created,
using the implementation-specific configuration.

```kotlin
import de.fhg.aisec.ids.idscp2.default_drivers.daps.aisec_daps.DefaultDapsDriver
import de.fhg.aisec.ids.idscp2.default_drivers.daps.aisec_daps.DefaultDapsDriverConfig
import de.fhg.aisec.ids.idscp2.default_drivers.daps.aisec_daps.SecurityRequirements
import de.fhg.aisec.ids.idscp2.default_drivers.daps.aisec_daps.SecurityProfile

fun main() {

     ...
     // create security requirements
     val securityRequirements = SecurityRequirements.Builder()
                .setRequiredSecurityLevel(SecurityProfile.TRUSTED)
                .build()

     // create a daps driver instance
     val dapsConfig = DefaultDapsDriverConfig.Builder()
                .setDapsUrl("https://daps.aisec.fraunhofer.de")
                .setKeyStorePath(keyStorePath)
                .setKeyStorePassword("password")
                .setTrustStorePath(trustStorePath)
                .setTrustStorePassword("password")
                .setSecurityRequirements(securityRequirements)
                .build()

     val dapsDriver: DapsDriver = DefaultDapsDriver(dapsConfig)

     ...
}
```

### Idscp2Configuration Generation

To keep it simple, the only configuration interface to the IDSCP2 protocol is the Idscp2Configuration, which
can be found at *idscp2/idscp_core/api/configuration/* .It takes the
DAPS driver and the attestation config from above, together with a handshake timeout interval and an acknowledgement
timeout interval. The handshake timeout interval specifies how long the different steps of an IDSCP2 handshake may take.
This might depend on your attestation mechanism and your DAPS driver. You have to ensure, that the attestation will not
take longer than your timeout allows, but of course you should restrict the handshake duration to a minimum to avoid
attacks, that might become possible due to unrestricted handshake durations.

The acknowledgement interval specifies the time to wait for an ack response after sending IDSCP2 data. When the
timeout occurs, the same data would be send again. This mechanism is used to ensure, that IDSCP2 data will not get lost
during re-attestations. (When re-attestation takes place, the peer is not in a trusted state anymore, which is why
IDSCP2 data messages will be ignored until the peer is proven to still be trusted.)

The default ack interval is set to 200 ms, the default handshake interval is set to five seconds.

```kotlin
import de.fhg.aisec.ids.idscp2.default_drivers.daps.aisec_daps.DefaultDapsDriver
import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.AttestationConfig
import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.Idscp2Configuration

fun main() {

     ...

     // create the IDSCP2 configuration using the builder pattern
     val idscp2Config = Idscp2Configuration.Builder()
                .setAttestationConfig(localAttestationConfig)
                .setDapsDriver(dapsDriver)
                .setHandshakeTimeoutDelay(3 * 1000) // 3 seconds
                .setAckTimeoutDelay(500)  // 500 ms

     ...
}
```

### SecureChannel Driver Setup

The SecureChannelDriver is responsible for connecting to and accepting new IDSCP2 peers.
Mostly you will need any kind of SecureChannelConfiguration, which keeps information about the server (e.g. host, port) 
as well as information about the security stuff (certificates, trust-chains, ..). Each SecureChannelDriver should provide 
its custom SecureChannelConfiguration.

In the following, you see how the TLSv1.3 secure channel (*idscp2/default_drivers/secure_channel/tlsv1_3/*) 
can be configured:

```kotlin
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTlsConfiguration

fun main() {

     ...
     // create TLSv1.3 config using the builder pattern
     val nativeTlsConfiguration = NativeTlsConfiguration.Builder()
                 .setKeyStorePath("path-to-key-store")
                 .setTrustStorePath("path-to-trust-store")
                 .setKeyStorePassword("password")
                 .setTrustStorePassword("password")
                 .setCertificateAlias("1.0.1")
                 .setServerPort(12345)
                 .setHost("consumer-core")
                 .setServerSocketTimeout(3000)
                 .build()

     ...
}
```

As a last step, you have to create the instance of the selected SecureChannelDriver, that will be used for
connecting and accepting new IDSCP2 peers:

```kotlin
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTlsConfiguration
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2Connection

fun main() {

     ...
     // create TLSv1.3 secure channel driver instance
     val secureChannelDriver = NativeTLSDriver<Idscp2Connection>()

     ...
}
```

## IDSCP2 Peer

The IDSCP2 peer can be seen as the combination of a client and a server component. 
Both make use of the SecureChannelDriver to establish new IDSCP2 connections.

### IDSCP2 Client

The IDSCP2 client component allows the peer to initiate an IDSCP2 connection by connecting to an
IDSCP2 server. In the configuration part above we have already seen how the SecureChannelDriver and the 
implementation-specific SecureChannelConfiguration, containing the server address and security settings, can be
created. The SecureChannelDriver supports an asynchronous **connect** method, which takes the Idscp2Configuration 
and the SecureChannel configuration and returns a completable connection future. On completion of the future, 
the connection will become available to the initiator via a future completion handler.

IDSCP2 makes use of asynchronous connection and message listeners, that have to be registered to the new connection
to ensure to be notified when events or new data are available. This can be done via the **addConnectionListener**
and **addMessageListener** methods of the Idscp2Connection. When the listeners have been registered, messaging has
to be unlocked via the **unlockMessaging** method. This is used to avoid race conditions that could occur when 
messages are received or connection events are available but the listeners have not been registered yet.

See the full connection API at *idscp2/idscp_core/api/idscp_connection/Idscp2Connection*.

In the following you can see how this could be done using the NativeTlsDriver: First we connect to the server via the
NativeTlsDriver using the Idscp2Configuration and the NativeTlsConfiguration as described above. Since connecting is 
async, we have to register an async completion handler to the returning connection future, which will be triggered
when the connection future is successfully completed. 
Within this handler, we have to register our connection listener that will be notified on new IDSCP2 messages 
(Idscp2MessageListener), as well as on errors and connection closure (Idscp2ConnectionListener). Afterwards, we have to unlock messaging.

You might pass the IDSCP2 connection to any connection wrapper or local variable outside of the future handler to make it available for further usage, such as sending data to the peer.

```kotlin
import java.util.concurrent.CompletableFuture
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2ConnectionImpl
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2Connection
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2ConnectionAdapter
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTLSDriver
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTlsConfiguration
import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.Idscp2Configuration

fun main() {

     ...
     
     // async connect using the TLSv1.3 secure channel driver and the idscp2 and tls configs
     var connectionFuture: CompletableFuture<Idscp2Connection> = 
                  secureChannelDriver.connect(::Idscp2ConnectionImpl, idscp2Config, nativeTlsConfiguration)          
     
     // handle async connection future 
     connectionFuture.thenAccept { connection: Idscp2Connection ->
            
            // add a connection listener that keeps track of errors and closures
            connection.addConnectionListener(object : Idscp2ConnectionAdapter() {
                override fun onError(t: Throwable) {
                    //TODO handle connection errors
                }

                override fun onClose() {
                    //TODO handle connection closures
                }
            })
            
            // add a message listener that will receive the Idscp2 data
            connection.addMessageListener { connection: Idscp2Connection, data: ByteArray ->
                    //TODO handle data
            }
            
            // unlock messaging, since all listeners have been registered now
            connection.unlockMessaging()

     }.exceptionally {t: Throwable? ->
            // failure
            null
     }

     val connection = connectionFuture.get()     
     // TODO act on connection
     ...
}
```

### IDSCP2 Server

The IDSCP2 server component is responsible for accepting new IDSCP2 connection requests from other peers. For this
purpose, the IDSCP2 protocol provides an IDSCP2EndpointListener, which can be found at *idscp2/idscp_core/api/*.
It implements the **onConnection** method, which is triggered when a new IDSCP2 peer has connected to the IDSCP2 server.

Similar to the IDSCP2 client component, you have to register the connection and message listeners to the new connection.
Further, you might wrap or pass the connection to any service, to make it available outside of the **onConnection** method, from where your application can act on the connection, e.g. sending data.

In the following you can see a custom endpoint listener that implements the Idscp2EndpointListener interface.

```kotlin
import de.fhg.aisec.ids.idscp2.idscp_core.api.Idscp2EndpointListener
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2ConnectionAdapter
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2Connection

class CustomIdscp2EndpointListener : Idscp2EndpointListener<Idscp2Connection> {

    override fun onConnection(connection: Idscp2Connection) {
        
        // add a connection listener that keeps track of errors and closures
        connection.addConnectionListener(object : Idscp2ConnectionAdapter() {
            override fun onError(t: Throwable) {
                //TODO handle connection errors
            }

            override fun onClose() {
                //TODO handle connection closures
            }
        })
        
        // add a message listener that will receive the Idscp2 data
        connection.addMessageListener { connection: Idscp2Connection, data: ByteArray ->
            //TODO handle data
        }
        
        //TODO act on connection
    }
}
```

For creating an Idscp2Server using the Idscp2ServerFactory (*idscp2/idscp_core/api/idscp_server/Idscp2ServerFactory*), 
you have to create such a custom endpoint listener instance, which will then be registered to the Idscp2Server during
Idscp2Server construction. This endpoint listener can be seen as the single instance that will be notified when new connections are received at the Idscp2Server.

Beside the Idscp2EndpointListener, the Idscp2ServerFactory takes the Idscp2Configuration, the SecureChannelDriver and the custom SecureChannel configuration as arguments and returns the factory that can be used for creating a running Idscp2Server via the **listen** method. It can be terminated again later by the **terminate** method.

```kotlin
import de.fhg.aisec.ids.idscp2.idscp_core.api.Idscp2EndpointListener
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2ConnectionImpl
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_connection.Idscp2Connection
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTLSDriver
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTlsConfiguration
import de.fhg.aisec.ids.idscp2.idscp_core.api.configuration.Idscp2Configuration
import de.fhg.aisec.ids.idscp2.idscp_core.api.idscp_server.Idscp2ServerFactory

fun main() {

     ...
     
     // create the endpoint listener that will be notified on new connections
     val endpointListener = CustomIdscp2EndpointListener()
     
     // configure the Idscp2 server factory
     val serverConfig = Idscp2ServerFactory(
          ::Idscp2ConnectionImpl,
          enpointListener,        
          idscp2Config,
          secureChannelDriver,
          nativeTlsConfiguration
     )
     
     // start the IDSCP2 server using the factory
     val idscp2Server = serverConfig.listen()
     
     ...
     
     // terminate the server
     idscp2Server.terminate()
}
```
