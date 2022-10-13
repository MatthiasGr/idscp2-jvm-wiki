IDSCP2 can be used as a communication protocol with [Apache Camel](https://camel.apache.org).
This section provides an overview over the usage of IDSCP2 with Camel as well as documentation for
the various options that can be applied to each component.

## Using IDSCP2 components

Camel can use IDSCP2 in a server as well as a client role.
The respective Camel components are referred to as `idscp2server` and `idscp2client`.
Both components can act as a Producer and Consumer, meaning that they can be used in both the
`<to />` and `<from />` nodes of Camel routes.

### Using an IDSCP2 client to send data

The following excerpt constructs a simple IDSCP2 client sending a ping message to an example
server every ten seconds:
```xml
<route id="ping">
    <from uri="timer://tenSecondsTimer?fixedRate=true&amp;period=10000"/>
    <setBody>
        <simple>PING</simple>
    </setBody>
    <log message="Sending PING message"/>
    <to uri="idscp2client://example-server/?sslContextParameters=#clientSslContext"/>
</route>
```
This route will connect to an IDSCP2 server on the host `example-server` and default port 29292
using the TLS parameters set in `clientSslContext`.
It will then send a simple message containing PING every ten seconds.

### Receive data using an IDSCP2 server

A corresponding server replying to every ping can be created like this:
```xml
<route id="pong">
    <from uri="idscp2server://0.0.0.0/?sslContextParameters=#serverSslContext"/>
    <log message="Received message from client: ${body}; Sending PONG message"/>
    <setBody>
        <simple>PONG</simple>
    </setBody>
</pong>
```
This route will open an IDSCP2 server listening on port 29292.
The server will use the TLS parameters from `serverSslContext`.
When an IDSCP2 client connects and sends a message, this route will log the message and reply with
PONG.

### Bidirectional communication using an IDSCP2 client.

In order to enable bi-directional communication, an IDSCP2 connection can be shared between
multiple routes.
We can modify the client from before in order to react to the server pong message in a second
route:
```xml
<route id="ping">
    <from uri="timer://tenSecondsTimer?fixedRate=true&amp;period=10000"/>
    <setBody>
        <simple>PING</simple>
    </setBody>
    <log message="Sending PING message"/>
    <to uri="idscp2client://example-server/?connectionShareId=pingPongClient,sslContextParameters=#clientSslContext"/>
</route>

<route id="logPong">
    <from uri="idscp2client://example-server/?connectionShareId=pingPongClient,sslContextParameters=#clientSslContext"/>
    <log message="Received message from server: ${body}">
    <setBody>
        <simple>${null}</simple>
    </setBody>
</route>
```
In the first route, we added the `connectionShareId` option.
This allows another route to reuse the same connection by specifying the same ID, as is
demonstrated by the second route.
The second route listens on the connection until the server sends a message.
When a message is received, it logs its content and sets the message body to zero, indicating, that
the message should not be sent back to the server.

### Send broadcast messages using an IDSCP2 server

A server component can also be used to send messages to all connected clients.
The following route sends a message to all connected clients every ten seconds.
```xml
<route id="serverBroadcast">
    <from uri="timer://tenSecondsTimer?fixedRate=true&amp;period=10000"/>
    <setBody>
        <simple>BROADCAST</simple>
    </setBody>
    <to uri="idscp2server://0.0.0.0?sslContextParameters=#serverSslContext">
</route>
```

## Additional component parameters

In addition to the options already described in the overview above, the IDSCP2 components can be configured using the URI parameters documented in this section.
Those parameters that can be applied to both client and server components are listed under the common section.

### Common

#### `transportSslContextParameters`

SSL context parameters used to establish the underlying TLS connection.
This parameter expects an object of type
[SSLContextParameters](https://www.javadoc.io/doc/org.apache.camel/camel-api/latest/org/apache/camel/support/jsse/SSLContextParameters.html).

#### `dapsSslContextParameters`

SSL context parameters used when communicating with the DAPS.
Like the `transportSslContextParameters`, this parameter accepts an
[SSLContextParameters](https://www.javadoc.io/doc/org.apache.camel/camel-api/latest/org/apache/camel/support/jsse/SSLContextParameters.html) object.

#### `sslContextParameters`

The SSL context parameters used instead of `transportSslContextParameters` or `dapsSslContextParameters`,
if the respective option is missing.
This parameter is deprecated.

#### `dapsKeyAlias`

The alias of the key used to sign token requests sent to the DAPS.
The key is selected from the key store specified in the `dapsSslContextParameters` or `sslContextParameters` parameter.

#### `dapsRaTimeoutDelay`

The time in milliseconds before the component requests a new DAT and re-attestation from the peer.

#### `useIdsMessages`

If set to `true`, the component will use IDS messages to communicate.
For more information about IDS messages, see the [IDS Messages page](IDS-Messages) of this Wiki.

For an example on how IDS messages can be used, see
[this](https://github.com/Fraunhofer-AISEC/trusted-connector/blob/master/examples/route-examples/demo-route.xml)
example route definition in the trusted connector repository.

#### `supportedRaSuites`

The IDs of remote attestation suites supported by this component.
Multiple values are separated by `|`.

#### `expectedRaSuites`

The IDs of remote attestation suites accepted by this component.
Multiple values are separated by `|`.
The selected attestation suite is the first value in this parameter that is also in the peers `supporetedRaSuite` parameter.

#### `copyHeadersRegex`

A regular expression used to select headers to copy between IDSCP2 and Camel messages.

### Server

#### `tlsClientHostnameVerification`

If set to `true`, the clients hostname or IP address is verified against their certificate.
This is enabled by default.

### Client

#### `connectionShareId`

An arbitrary ID used to share an IDSCP2 connection between multiple clients.
Another client may reuse the connection by specifying the same ID.

#### `awaitResponse`

If set to `true`, the client will wait for a response message from the server when used as a producer (As a `<to />`-node in a Camel route).

#### `responseTimeout`

Timeout in milliseconds when waiting for a response from the server.

#### `maxRetries`

Maximum number of connection attempts the client undertakes when connecting to a server.

#### `retryDelayMs`

The delay in milliseconds after a failed connection attempt.