
## The three Azure Key Vault object types

<img width="1440" height="1288" alt="image" src="https://github.com/user-attachments/assets/eca8b89d-03f3-49da-b65f-8f4df6c0c633" />

---

The diagram is a faithful English translation of your original image, with one addition at the bottom: the warning banner explains directly in the diagram itself why **Certificate** must be used instead of **Key** for your Salesforce JWT use case — so the diagram is self-contained and can be shared with your team without needing the surrounding explanation. Each header card is also clickable for more detail.

---

Here are the three types of objects in Azure Key Vault, their roles and their differences:

**Keys** are used to perform cryptographic operations — encrypt, decrypt, sign, verify. They never leave the vault. This is the lowest-level security primitive.

**Secrets** are simple key/value pairs stored securely — passwords, connection strings, API tokens. Your application reads and uses them directly in plain text.

**Certificates** combine both: a TLS/SSL certificate contains a public/private key pair plus validity metadata. Key Vault can manage their full lifecycle (creation, automatic renewal).

| | Keys | Secrets | Certificates |
|---|---|---|---|
| What it is | Cryptographic key | Sensitive text value | Full TLS/SSL object |
| App receives | Never the key (operates via API) | The value in plain text | Certificate + private key |
| Auto-renewal | ❌ | ❌ | ✅ |
| Extra cost (HSM) | ✅ Premium | ❌ | ❌ |

---

## Why Certificate, not Key — even for JWT signing

This is the core confusion, and it is worth being precise about it.

The diagram shows that Keys support **Sign / Verify** — so at first glance, a Key seems like the natural choice for signing a JWT. But there is a fundamental security rule that makes Keys incompatible with your use case:

> **The private key of a Key object NEVER leaves the vault — ever.**

With a Key, your application cannot export or possess the private key. It can only *ask* Key Vault to perform the signing operation via an API call, and Key Vault returns the signature. The key itself stays locked inside.

**This is exactly the opposite of what Salesforce needs.**

Salesforce Certificate and Key Management must **physically hold the private key** inside Salesforce in order to sign JWT assertions autonomously at runtime — without making an outbound call to Azure Key Vault on every single OAuth2 token request. If the private key cannot be exported, it cannot be imported into Salesforce, and the whole flow breaks.

This is why you must use a **Certificate** object instead:

| | Key | Certificate |
|---|---|---|
| Private key exportable | ❌ Never, by design | ✅ Yes, if `exportable: true` in policy |
| Export format | Public key JWK only | Full PFX / PEM (private key included) |
| Contains X.509 | ❌ | ✅ Required by Salesforce |
| Auto-renewal | ❌ | ✅ |

Salesforce *Import from Keystore* expects a **JKS** file (converted from PFX), which is a standard Java KeyStore format containing both the private key and the X.509 certificate. Only the Certificate object in Key Vault can produce this.

---

## The correct flow for your use case

```
1. Create a Certificate in Azure Key Vault
        (RSA 2048/4096 bits, self-signed or CA-signed)
        policy: exportable = true, contentType = application/x-pkcs12

2. Export the full PFX (private key + certificate)
        az keyvault secret download --encoding base64
        ← must use the SECRET endpoint, not the certificate endpoint
          because only the secret backing object holds the private key

3. Convert PFX → JKS (Salesforce only accepts JKS format)
        keytool -importkeystore -srcstoretype PKCS12 -deststoretype JKS

4. Import JKS into Salesforce
        Setup → Certificate and Key Management → Import from Keystore
        Salesforce now holds the private key securely

5. Configure the Named Credential
        → select sf-jwt-callout as the signing certificate
        → protocol: JWT Token Exchange

6. Salesforce signs the JWT with the private key (autonomously, at runtime)
        → sends the signed JWT to the Authorization Server (Entra ID, etc.)

7. The Authorization Server validates the signature
        using the public key registered against your Azure App Registration
        → returns an access_token

8. Salesforce uses the access_token to make the REST API callout
```

---

## Why `az keyvault secret download` and not `az keyvault certificate download`

This is the other point of confusion in the Azure CLI commands:

```bash
# ✅ Correct — exports the full PFX including the private key
az keyvault secret download \
  --vault-name <vault> \
  --name "salesforce-jwt-cert" \
  --encoding base64 \
  --file salesforce-jwt-cert.pfx

# ❌ Wrong for this use case — exports the public certificate only (.cer)
az keyvault certificate download \
  --vault-name <vault> \
  --name "salesforce-jwt-cert" \
  --file salesforce-jwt-cert.cer
```

When Azure Key Vault creates a Certificate, it internally creates **three linked objects** under the same name: a Certificate (the X.509 metadata), a Key (the RSA key pair), and a **Secret** (the PKCS#12 bundle containing both). The `az keyvault secret download` command is the only one that reaches the PKCS#12 bundle — and therefore the only one that gives you the private key in exportable form.
