# Glossary — JWT · Azure Key Vault · Salesforce

The glossary covers all terms used throughout our conversation, organized in 5 categories and searchable in real time. Here is a quick summary of what is covered:

**Azure** — AKV, Entra ID, HSM, Managed Identity, RBAC, Service Principal

**Salesforce** — CKM, Named Credential, Remote Site Settings, Protected Custom Setting

**OAuth2 / JWT** — JWT, JWS, JWK, x5t, x5t#S256, alg, RS256, all JWT claims (iss, sub, aud, exp, iat, jti), Client Credentials grant, JWT Bearer grant

**Cryptography** — RSA, EC, X.509, CN, SAN, CA, DER, SHA, TLS/SSL, mTLS, FIPS

**File formats** — PFX/.p12, JKS, PEM, .cer/.crt, PKCS

---

## Azure

| Acronym | Full name | Definition |
|---|---|---|
| **AKV** | Azure Key Vault | Microsoft Azure managed service for securely storing and managing cryptographic keys, secrets, and certificates. Contains three object types: Keys, Secrets, Certificates. |
| **Entra ID** | Microsoft Entra ID *(formerly Azure Active Directory / AAD)* | Microsoft's cloud identity and access management service. Acts as the Authorization Server for OAuth2 / JWT Bearer flows. |
| **HSM** | Hardware Security Module | Physical tamper-resistant device that generates and protects cryptographic keys. AKV Premium tier uses FIPS 140-2 Level 2/3 validated HSMs. Keys stored in HSM never leave the hardware boundary. |
| **MI** | Managed Identity | Azure feature that provides an Azure service (VM, App Service, etc.) with an automatically managed identity in Entra ID. Eliminates the need for a `client_secret` to authenticate to AKV. |
| **RBAC** | Role-Based Access Control | Access model used in AKV to grant permissions (Key Vault Secrets Officer, Key Vault Crypto User, etc.) to users or service principals on vault objects. |
| **SP** | Service Principal | An identity created for an Azure App Registration. Used by applications (like Salesforce) to authenticate to Entra ID and access AKV or other Azure resources. |

---

## Salesforce

| Acronym | Full name | Definition |
|---|---|---|
| **CKM** | Certificate and Key Management | Salesforce feature (Setup → Certificate and Key Management) where certificates and their private keys are imported and stored. Used by Named Credentials to sign JWT assertions. |
| **NC** | Named Credential | Salesforce configuration object that stores an external endpoint URL and its authentication settings. Apex callouts reference it as `callout:MyNamedCredential/path`. Handles token acquisition automatically. |
| **RSS** | Remote Site Settings | Salesforce whitelist of external URLs that Apex code is permitted to call out to. Must include the token endpoint and API endpoint domains before any `Http.send()` call. |
| **PCS** | Protected Custom Setting | Salesforce Custom Setting marked as Protected. Values are encrypted at rest and not visible in setup UI or exportable via Data Loader. Used to store sensitive values like `client_secret` or tenant IDs. |

---

## OAuth2 / JWT

| Acronym | Full name | Definition |
|---|---|---|
| **JWT** | JSON Web Token *(RFC 7519)* | Compact, URL-safe token format composed of three base64url-encoded parts: **header** (alg, typ, x5t), **payload** (claims: iss, sub, aud, exp, iat, jti), and **signature**. |
| **JWS** | JSON Web Signature *(RFC 7515)* | Standard that defines how a JWT is signed. Defines header parameters including `alg`, `x5t`, and `x5t#S256`. |
| **JWK** | JSON Web Key *(RFC 7517)* | JSON representation of a cryptographic key (public or private). AKV Key objects export their public key in JWK format only — the private key is never exported from a Key object. |
| **x5t** | X.509 Certificate Thumbprint | Optional JWT header parameter (RFC 7515). A base64url-encoded SHA-1 fingerprint of the DER-encoded X.509 certificate used to sign the JWT. Used by the Authorization Server to identify which registered public key to use for signature verification. Not involved in the signing operation itself. |
| **x5t#S256** | X.509 Certificate SHA-256 Thumbprint | SHA-256 variant of `x5t` (RFC 7515). Preferred over `x5t` as SHA-1 is deprecated. Same purpose: key discovery hint for the Authorization Server. |
| **alg** | Algorithm | JWT header parameter defining the cryptographic algorithm used to sign the token. Common values: `RS256` (RSA + SHA-256), `RS512`, `ES256` (Elliptic Curve). Sufficient alone if only one certificate is registered on the Authorization Server. |
| **RS256** | RSA Signature with SHA-256 | JWT signing algorithm. The signing input (header.payload) is hashed with SHA-256 then the digest is signed with the RSA private key using RSASSA-PKCS1-v1_5 padding. |
| **iss** | Issuer | JWT claim identifying who issued the token. In client credentials / JWT Bearer flows: set to the `client_id` of the Azure App Registration. |
| **sub** | Subject | JWT claim identifying the principal. In client credentials flows: same value as `iss` (the `client_id`). In user-delegated flows: the user's unique identifier. |
| **aud** | Audience | JWT claim specifying the intended recipient. Must match the token endpoint URL of the Authorization Server (e.g. `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`). |
| **exp** | Expiration | JWT claim (Unix timestamp in seconds) after which the token must be rejected. Typically set to `iat + 300` (5 minutes). Authorization Servers reject tokens with `exp` too far in the future. |
| **iat** | Issued At | JWT claim (Unix timestamp) recording when the token was created. Used by the Authorization Server to validate the token is not stale. |
| **jti** | JWT ID | Unique identifier for the JWT (RFC 7519). Prevents replay attacks — the Authorization Server can reject a JWT whose `jti` has already been seen. Generated as a random hex string per request. |
| **CC** | Client Credentials | OAuth2 grant type (`grant_type=client_credentials`). Server-to-server authentication using `client_id` + `client_secret`. Simplest flow — no JWT signing required. |
| **JB** | JWT Bearer | OAuth2 grant type (`grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`). The client sends a signed JWT as its credential instead of a `client_secret`. Requires a certificate registered on the Authorization Server. |

---

## Cryptography

| Acronym | Full name | Definition |
|---|---|---|
| **RSA** | Rivest-Shamir-Adleman | Asymmetric cryptographic algorithm based on the difficulty of factoring large integers. Supported key sizes in AKV: 2048, 3072, 4096 bits. 2048 is standard; 4096 doubles signing time with minimal extra security for most use cases. |
| **EC** | Elliptic Curve | Asymmetric cryptography based on elliptic curves. Smaller key sizes for equivalent security vs RSA (P-256 ≈ RSA 3072). Not all Authorization Servers support EC for JWT signing. |
| **X.509** | X.509 Certificate | Standard format for public key certificates (RFC 5280). Contains: **public key**, subject DN (CN, O, C…), issuer, validity period (notBefore / notAfter), key usage, SANs, and a CA signature. Not just the public key — the public key embedded in a structured, signed envelope. |
| **CN** | Common Name | Part of the X.509 certificate Subject Distinguished Name (DN). For TLS certs: the domain name. For service identity certs (like `sf-jwt-callout`): a descriptive name identifying the application or service. |
| **SAN** | Subject Alternative Name | X.509 certificate extension listing additional identities (DNS names, IP addresses, email addresses). Mandatory for TLS certificates; optional for JWT signing certificates. |
| **CA** | Certificate Authority | Trusted entity that signs and issues X.509 certificates. AKV integrates with public CAs (DigiCert, GlobalSign) for automatic issuance. Self-signed certificates act as their own CA and are valid for JWT Bearer flows. |
| **DER** | Distinguished Encoding Rules | Binary encoding format for ASN.1 data structures including X.509 certificates. The `x5t` thumbprint is computed as `SHA-1(DER-encoded certificate)`. Contrast with PEM which is base64 of DER wrapped in headers. |
| **SHA** | Secure Hash Algorithm | SHA-1 (160-bit, deprecated) and SHA-256 (256-bit, recommended) are used for: (1) computing the `x5t` thumbprint, (2) hashing the JWT signing input before RSA signing (RS256 = RSA + SHA-256). |
| **TLS/SSL** | Transport Layer Security / Secure Sockets Layer | Protocol securing network communications. TLS certificates (stored as AKV Certificates) authenticate servers and encrypt traffic. mTLS (mutual TLS) authenticates both client and server. |
| **mTLS** | Mutual TLS | TLS variant where both the client and server present certificates for authentication. An alternative to JWT Bearer for API security. |
| **FIPS** | Federal Information Processing Standards | US government security standards. FIPS 140-2 Level 2/3 defines requirements for cryptographic modules. AKV Premium HSM-backed keys are FIPS 140-2 validated — relevant when 4096-bit RSA keys are required by compliance policy. |

---

## File formats

| Format | Full name | Definition |
|---|---|---|
| **PFX / .p12** | PKCS#12 / Personal Information Exchange | Binary file format that bundles a private key + X.509 certificate (+ optional chain) into a single password-protected file. Retrieved from AKV via `az keyvault secret download`. Must be converted to JKS for Salesforce import. |
| **JKS** | Java KeyStore | Java-native keystore format (`.jks`). The only format accepted by Salesforce's *Import from Keystore*. Converted from PFX using `keytool -importkeystore -srcstoretype PKCS12 -deststoretype JKS`. |
| **PEM** | Privacy Enhanced Mail | Base64-encoded DER certificate wrapped between `-----BEGIN CERTIFICATE-----` headers. Used to share the public certificate with the Authorization Server (Azure App Registration → Certificates & secrets → Upload). |
| **.cer / .crt** | Certificate file | Public certificate only (no private key). Exported via `az keyvault certificate download`. Registered on the Authorization Server so it can verify JWT signatures. Safe to share — contains no secret material. |
| **PKCS** | Public Key Cryptography Standards | Family of standards by RSA Security. Relevant here: PKCS#1 (RSA key format), PKCS#8 (private key format), PKCS#12 (PFX bundle). AKV uses `application/x-pkcs12` as the secret content type for Certificate objects. |

---

*Generated from the JWT · Azure Key Vault · Salesforce conversation — March 2026*
