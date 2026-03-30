Here is the corrected and complete flow, now in English.




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
