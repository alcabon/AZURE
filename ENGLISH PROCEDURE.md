Here is the corrected and complete flow, now in English.


<img width="1440" height="720" alt="image" src="https://github.com/user-attachments/assets/42db9b68-df95-44d1-a1a4-21675125c7cf" />


---

### What the diagram shows

**Azure Key Vault — Certificate `sf-jwt-callout`**

The Certificate object stores two elements together and inseparably:

- The **private key** (RSA 2048 bits) — used by Salesforce to sign JWT assertions.
- The **X.509 Certificate**, which is *not just a public key* — it is a structured envelope containing: the public key itself, the subject (`CN=salesforce-jwt-callout`), the issuer (Self or a CA), the validity period (not before / not after), the key usage (`digitalSignature`), and optionally Subject Alternative Names (SANs). This is the file you register on the Authorization Server so it can verify Salesforce's JWT signature.

---

**Step 1 — Export as PFX**

`az keyvault secret download --encoding base64` extracts both elements together as a single PKCS#12 (`.pfx`) file. You must go through the **secret** endpoint (not the certificate endpoint), because only the secret contains the private key.

**Step 2 — Convert PFX → JKS**

Salesforce's *Import from Keystore* only accepts **JKS** (Java KeyStore) format. The conversion is done with `keytool`:

```bash
keytool -importkeystore \
  -srckeystore sf-jwt-callout.pfx \
  -srcstoretype PKCS12 \
  -srcstorepass "" \
  -destkeystore sf-jwt-callout.jks \
  -deststoretype JKS \
  -deststorepass changeit
```

**Step 3 — Import into Salesforce**

Setup → **Certificate and Key Management** → **Import from Keystore** → select `sf-jwt-callout.jks` → enter the keystore password. Salesforce then secures the private key internally and exposes the certificate for use in the Named Credential with **JWT Token Exchange**.

---

Two distinct questions, two clear answers.

---

## 1. RSA 4096 bits — yes, but with trade-offs

Azure Key Vault supports RSA 2048, 3072, and 4096. The cert policy change is minimal:

```bash
# Only change keySize from 2048 to 4096
"keyProperties": {
  "exportable": true,
  "keyType": "RSA",
  "keySize": 4096,       # ← change here
  "reuseKey": false
}
```

Salesforce also supports 4096 bits in Certificate and Key Management. However, the practical impact is:

| | RSA 2048 | RSA 4096 |
|---|---|---|
| Security level | ~112 bits | ~140 bits |
| JWT signing time | ~1 ms | ~6–10 ms |
| Key size on wire | 256 bytes | 512 bytes |
| AKV cost (Premium HSM) | Standard rate | 2× rate (per key size tier) |
| Recommended for | Most OAuth2 callouts | High-security / compliance requirements |

For a Salesforce OAuth2 JWT callout, **RSA 2048 is the standard recommendation**. 4096 is justified only if your security policy explicitly requires it (FIPS compliance, financial/healthcare regulations). The added latency on every token exchange is the main practical cost.

---

## 2. What is `x5t` and its equivalent in Azure Key Vault

`x5t` is a **JWT header parameter** defined in RFC 7515. It is the **base64url-encoded SHA-1 thumbprint** of the DER-encoded X.509 certificate used to sign the JWT:

```
x5t = base64url( SHA-1( DER( X.509 certificate ) ) )
```

It allows the Authorization Server (or Salesforce) receiving the JWT to identify **which certificate** was used to sign, without embedding the full certificate in the token.

A signed JWT header using `x5t` looks like:

```json
{
  "alg": "RS256",
  "x5t": "aB3kLm9nQr2sT4uVwXyZ..."
}
```

**Its equivalent in Azure Key Vault** is the **certificate thumbprint**, exposed in two forms:

```bash
# SHA-1 thumbprint (= x5t)
az keyvault certificate show \
  --vault-name <vault> \
  --name sf-jwt-callout \
  --query "x509ThumbprintHex" \
  --output tsv

# Full details including thumbprint
az keyvault certificate show \
  --vault-name <vault> \
  --name sf-jwt-callout \
  --query "{thumbprint: x509ThumbprintHex, subject: policy.x509CertificateProperties.subject, expiry: attributes.expires}"
```

There is also `x5t#S256`, the **SHA-256 variant** (preferred over SHA-1 in modern systems):

```
x5t#S256 = base64url( SHA-256( DER( X.509 certificate ) ) )
```

| Parameter | Hash | Status |
|---|---|---|
| `x5t` | SHA-1 | RFC 7515 — still widely used, but SHA-1 is deprecated |
| `x5t#S256` | SHA-256 | RFC 7515 — recommended |

In the context of **Salesforce JWT Bearer Flow**, Salesforce automatically includes the correct `x5t` in the JWT header based on the certificate you select in the Named Credential — you do not need to set it manually. It uses the thumbprint of the imported JKS certificate to identify itself to the Authorization Server.
