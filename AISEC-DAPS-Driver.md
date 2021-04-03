The AISEC DAPS driver (*idscp2/default_drivers/daps/aisec_daps/SecurityRequirements.kt*) is used for requesting and verifying dynamic attribute tokens from the DAPS. During initialization it constructs a SSL socket factory from the given security parameters in the configuration and extracts the local connector UUID from the local certificate.

When the IDSCP2 FSM is requesting a new local DAT via the DAPS driver, the driver will first check if there is a local DAT cached from a previous request. When a DAT is cached and the remaining DAT lifetime is larger than a threshold (relative to the overall DAT lifetime, e.g. 66,6%), the cached DAT is issued instead of a new one to reduce communication overhead with the DAPS. When there is no such DAT cached or the remaining DAT lifetime is too small, a new DAT is requested from the DAPS, cached in the driver and returned to the FSM.

The verifying method first checks the signature, the DAT issuer, the target audience and the expiration time. Afterwards, the peer's transport certificate is checked against the expected certificate fingerprints from the DAT. This is done to ensure that the DAT is owned by the IDSCP2 peer that owns the transport certificate. Finally, the security requirements are validated against the provided security profile from the peer, attested by the DAPS.

## Configuration

The DAPS driver can be configured using the implementation-specific configuration class at *idscp2/default_drivers/daps/aisec_daps/AisecDapsDriverConfig.kt*. It provides a Builder pattern with the following API:

Set the URL for the DAPS:
```kotlin
fun setDapsUrl(dapsUrl: String): Builder 
```

Set the keystore path, key-alias and passwords, used for secure communication with the DAPS:
```kotlin
fun setKeyStorePath(path: Path): Builder
fun setKeyStorePassword(password: CharArray): Builder
fun setKeyAlias(alias: String): Builder
fun setKeyPassword(password: CharArray): Builder
```

Set the truststore that contains the root certificate of the DAPS:
```kotlin
fun setTrustStorePath(path: Path): Builder
fun setTrustStorePassword(password: CharArray): Builder
```

Set the expected connector security profile that is required for accepting connections from other IDSCP peers.
```kotlin
fun setSecurityRequirements(securityRequirements: SecurityRequirements): Builder
```

Set the threshold that defines how long a local DAT should be cached in the DAPS driver until a new DAT is requested from the DAPS. The threshold must be between zero and one and is relative to the lifetime of the issued DAT.
```kotlin
fun setTokenRenewalThreshold(threshold: Float): Builder
```

Build the configuration:
```kotlin
fun build(): AisecDapsDriverConfig
```

## Security Requirements

The security requirements (*idscp2/default_drivers/daps/aisec_daps/SecurityRequirements.kt*) contain the security profile that are expected for connecting with the local peer. A Builder pattern is provided with the following API:

Set the expected security profile:
```kotlin
fun setRequiredSecurityLevel(securityProfile: SecurityProfile): Builder
```

Build the security requirements:
```kotlin
fun build(): SecurityRequirements
```
----
The possible security profiles are:

```kotlin
enum class SecurityProfile {
    
    INVALID,       // used for internal parsing errors only, must not be used by the user
    BASE,          // bind to idsc:BASE_CONNECTOR_SECURITY_PROFILE
    TRUSTED,       // bind to idsc:TRUSTED_CONNECTOR_SECURITY_PROFILE
    TRUSTED_PLUS;  // bind to idsc:TRUSTED_CONNECTOR_PLUS_SECURITY_PROFILE

}
```

----
The security requirements can be updated dynamically on the DAPS driver instance:
```kotlin
fun updateSecurityRequirements(securityRequirements: SecurityRequirements)
```

## How to use the AISEC DAPS driver in your project

```kotlin
val securityRequirements = SecurityRequirements.Builder()
        .setRequiredSecurityLevel(SecurityProfile.TRUSTED_PLUS)
        .build()

val dapsConfig = AisecDapsDriverConfig.Builder()
        .setDapsUrl("https://daps.aisec.fraunhofer.de")
        .setKeyStorePath("path-to-your-keystore")
        .setKeyStorePassword("password")
        .setKeyAlias("1")
        .setKeyPassword("password")
        .setTrustStorePath("path-to-your-truststore")
        .setTrustStorePassword("password")
        .setSecurityRequirements(securityRequirements)
        .build()

val dapsDriver = AisecDapsDriver(dapsConfig)
```