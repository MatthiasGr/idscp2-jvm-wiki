## Native TLS Driver

The NativeTlsDriver is located at *idscp2/default_drivers/secure_channel/tlsv1_3/*.

## Configuration

The Native TLS Driver can be configured using the NativeTlsConfiguration. It provides the following API:

Set the server host and port:
```kotlin
fun setHost(host: String): Builder            // only used for IDSCP2 clients
fun setServerPort(serverPort: Int): Builder
```

Set the keystore information:
```kotlin
fun setKeyStorePath(path: Path): Builder
fun setKeyStorePassword(pwd: CharArray): Builder
fun setCertificateAlias(alias: String): Builder
fun setKeyStoreKeyType(keyType: String): Builder
fun setKeyPassword(pwd: CharArray): Builder
```

Set the truststore information:
```kotlin
fun setTrustStorePath(path: Path): Builder
fun setTrustStorePassword(pwd: CharArray): Builder
```

Set the socket timeout, which is used for stopping the secure channel:
```kotlin
fun setServerSocketTimeout(timeout: Int): Builder
```

Disable hostname verification (should not be disabled for productive usage):
```kotlin
fun dangerDisableHostnameVerification(): Builder
```

Build the configuration:
```kotlin
fun build(): NativeTlsConfiguration
```

## TLS Server

## TLS Client

## How to use the Native TLS Driver
```kotlin
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTlsConfiguration
import de.fhg.aisec.ids.idscp2.default_drivers.secure_channel.tlsv1_3.NativeTlsDriver
```

```kotlin
val tlsConfig = NativeTlsConfiguration.Builder()
        .setHost("localhost")
        .setServerPort(1234)
        .setKeyStorePath("path-to-key-stor")
        .setKeyStorePassword("password")
        .setKeyStoreKeyType("RSA")
        .setCertificateAlias("1")
        .setKeyPassword("password")
        .setTrustStorePath("path-to-trust-store")
        .setTrustStorePassword("password")
        .build()

val tlsDriver = NativeTlsDriver<Idscp2Connection>()
```