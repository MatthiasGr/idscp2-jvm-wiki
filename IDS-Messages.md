While the IDSCP2 protocol can be used as a communication channel for any application, its primary
use case is within International Data Spaces.
This section aims to explain the basics of the IDS Message Format and how it can be used.

## IDS Messages

An IDSCP2 [AppLayerConnection](https://github.com/industrial-data-space/idscp2-jvm/blob/develop/idscp2-app-layer/src/main/kotlin/de/fhg/aisec/ids/idscp2/applayer/AppLayerConnection.kt)
can be used to exchange IDS messages.
An AppLayer message consists of a primary header, a payload and a set of additional headers.
When IDS messages are used, the primary header must be a JSON-LD encoded IDS infomodel message as
defined [here](https://htmlpreview.github.io/?https://github.com/IndustrialDataSpace/InformationModel/blob/feature/message_taxonomy_description/model/communication/Message_Description.htm).
In Java, infomodel datatypes derive from the `de.fraunhofer.iais.eis.Message` interface.

## Using IDS Messages with Apache Camel

In order to instruct an IDSCP2 Camel component to use IDS messages, the `useIdsMessages` parameter
must be set to true.
When sending an IDS message using an IDS-enabled Camel component, the route must set the header
`idscp2-header` to an instance of the java representation of an IDS Infomodel message or the
builder of such a message.
Builder classes for infomodel messages are provided in the `de.fraunhofer.iais.eis` package.
When the header is set to a message builder, some fields including the component's DAT are supplied
by IDSCP2.

In the [Trusted Connector project](https://github.com/Fraunhofer-AISEC/trusted-connector), message
builders are created using Camel processors.
To submit an `ArtifactRequestMessage` for example, a route must first set the `artifactUri`
property and then invoke an `ArtifactRequestCreationProcessor`.
The processor then creates and initializes an `ArtifactRequestMessageBuilder` using the
`artifactUri` property.
It then finishes processing by assigning the message builder to `idscp2 header`.
The route can then send the message using an IDSCP2 component.
