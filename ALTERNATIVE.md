Yes, absolutely — and this is actually the **architecturally cleaner approach**. This is the true use case for the **Key** object (not Certificate) in Azure Key Vault. The private key never leaves AKV, ever.

---

## How it works

Instead of Salesforce holding the private key, Salesforce:
1. Builds the JWT header + payload
2. Calls the **Azure Key Vault Sign API** to sign the digest
3. Gets the signature back
4. Assembles the complete JWT
5. Exchanges it for an access token at the Authorization Server

The private key stays in AKV. Salesforce only ever sees the signature.

```
SALESFORCE                    AZURE KEY VAULT              AUTH SERVER (Entra ID)
──────────                    ───────────────              ──────────────────────

1. Build JWT header + payload
   (base64url encoded)

2. SHA-256 hash the signing input
   (Crypto.generateDigest)

3. POST /keys/sf-jwt-key/sign  ──►  Signs with private key
   { alg: RS256,                    (never leaves AKV)
     value: base64url(hash) }  ◄──  Returns base64url signature

4. Assemble JWT
   header.payload.signature

5. POST /token + JWT assertion  ──────────────────────────►  Validates signature
                                                              with registered pubkey
                                ◄──────────────────────────  access_token

6. API callout with Bearer token
```

---

## What you still need in Salesforce

The private key is gone from Salesforce, but you still need **credentials to authenticate to AKV**. The trade-off:

| Approach | What Salesforce holds |
|---|---|
| Certificate in Salesforce (previous) | Private key (RSA 2048/4096) |
| AKV Sign API (this approach) | AKV Service Principal `client_id` + `client_secret` |

The `client_secret` is simpler to rotate and less sensitive than a private key. It can also be stored in a **Protected Custom Setting** in Salesforce, which is encrypted at rest.

---

## Apex implementation

```apex
public class AzureKvSignedJwtCallout {

    // ─── AKV Configuration ────────────────────────────────────────────────────
    // Store these in a Protected Custom Setting (Setup → Custom Settings)
    // Never hardcode secrets in Apex
    private static final String AKV_TENANT_ID     = getProtectedSetting('AKV_TENANT_ID');
    private static final String AKV_CLIENT_ID     = getProtectedSetting('AKV_CLIENT_ID');
    private static final String AKV_CLIENT_SECRET = getProtectedSetting('AKV_CLIENT_SECRET');
    private static final String AKV_VAULT_URL      = 'https://your-vault.vault.azure.net';
    private static final String AKV_KEY_NAME       = 'sf-jwt-key';
    private static final String AKV_KEY_VERSION    = '';   // blank = latest version
    private static final String AKV_API_VERSION    = '7.4';

    // ─── Target OAuth2 Configuration ─────────────────────────────────────────
    private static final String TARGET_TENANT_ID  = 'your-target-tenant-id';
    private static final String TARGET_CLIENT_ID  = 'your-azure-app-client-id';
    private static final String TARGET_SCOPE      = 'https://your-api/.default';
    private static final Integer TOKEN_EXPIRY_SEC = 300;

    private static final String TARGET_TOKEN_ENDPOINT =
        'https://login.microsoftonline.com/' + TARGET_TENANT_ID + '/oauth2/v2.0/token';

    // ─── Public entry point ───────────────────────────────────────────────────
    public static HttpResponse callout(String method, String endpoint, String body) {
        String accessToken = getTargetAccessToken();

        HttpRequest req = new HttpRequest();
        req.setEndpoint(endpoint);
        req.setMethod(method);
        req.setHeader('Authorization', 'Bearer ' + accessToken);
        req.setHeader('Content-Type', 'application/json');
        req.setTimeout(30000);
        if (body != null) req.setBody(body);

        HttpResponse res = new Http().send(req);
        if (res.getStatusCode() >= 400) {
            throw new KvJwtException(
                'Callout failed [' + res.getStatusCode() + ']: ' + res.getBody()
            );
        }
        return res;
    }

    // ─── Step 1: Get target access token via JWT Bearer Flow ─────────────────
    @TestVisible
    private static String getTargetAccessToken() {
        String jwt = buildAkvSignedJwt();

        HttpRequest req = new HttpRequest();
        req.setEndpoint(TARGET_TOKEN_ENDPOINT);
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/x-www-form-urlencoded');
        req.setTimeout(15000);
        req.setBody(
            'grant_type='  + encode('urn:ietf:params:oauth:grant-type:jwt-bearer') +
            '&client_id='  + encode(TARGET_CLIENT_ID) +
            '&scope='      + encode(TARGET_SCOPE) +
            '&assertion='  + encode(jwt)
        );

        HttpResponse res = new Http().send(req);
        if (res.getStatusCode() != 200) {
            throw new KvJwtException(
                'Token exchange failed [' + res.getStatusCode() + ']: ' + res.getBody()
            );
        }

        return (String) ((Map<String, Object>)
            JSON.deserializeUntyped(res.getBody())).get('access_token');
    }

    // ─── Step 2: Build JWT and sign via AKV Sign API ──────────────────────────
    @TestVisible
    private static String buildAkvSignedJwt() {

        // 2a. Get AKV access token (authenticate Salesforce → AKV)
        String akvToken = getAkvAccessToken();

        // 2b. Build JWT header and payload
        Map<String, Object> header = new Map<String, Object>{
            'alg' => 'RS256',
            'typ' => 'JWT'
        };

        Long now = DateTime.now().getTime() / 1000;
        Map<String, Object> payload = new Map<String, Object>{
            'iss' => TARGET_CLIENT_ID,
            'sub' => TARGET_CLIENT_ID,
            'aud' => TARGET_TOKEN_ENDPOINT,
            'iat' => now,
            'exp' => now + TOKEN_EXPIRY_SEC,
            'jti' => generateJti()
        };

        String headerB64  = base64UrlEncode(Blob.valueOf(JSON.serialize(header)));
        String payloadB64 = base64UrlEncode(Blob.valueOf(JSON.serialize(payload)));
        String signingInput = headerB64 + '.' + payloadB64;

        // 2c. SHA-256 hash of the signing input
        // AKV Sign API expects the DIGEST (hash), not the raw data
        // For RS256: provide base64url(SHA-256(signingInput))
        Blob sha256Hash = Crypto.generateDigest('SHA-256', Blob.valueOf(signingInput));
        String digestB64 = base64UrlEncode(sha256Hash);

        // 2d. Call AKV Sign API
        String signature = callAkvSignApi(akvToken, digestB64);

        // 2e. Assemble final JWT
        return signingInput + '.' + signature;
    }

    // ─── Step 3: Get AKV access token (client_credentials) ───────────────────
    @TestVisible
    private static String getAkvAccessToken() {
        HttpRequest req = new HttpRequest();
        req.setEndpoint(
            'https://login.microsoftonline.com/' + AKV_TENANT_ID + '/oauth2/v2.0/token'
        );
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/x-www-form-urlencoded');
        req.setTimeout(10000);
        req.setBody(
            'grant_type=client_credentials' +
            '&client_id='     + encode(AKV_CLIENT_ID) +
            '&client_secret=' + encode(AKV_CLIENT_SECRET) +
            '&scope='         + encode('https://vault.azure.net/.default')
        );

        HttpResponse res = new Http().send(req);
        if (res.getStatusCode() != 200) {
            throw new KvJwtException(
                'AKV auth failed [' + res.getStatusCode() + ']: ' + res.getBody()
            );
        }

        return (String) ((Map<String, Object>)
            JSON.deserializeUntyped(res.getBody())).get('access_token');
    }

    // ─── Step 4: Call AKV Sign API ────────────────────────────────────────────
    @TestVisible
    private static String callAkvSignApi(String akvToken, String digestB64) {
        String keyVersion = String.isBlank(AKV_KEY_VERSION) ? '' : '/' + AKV_KEY_VERSION;
        String endpoint =
            AKV_VAULT_URL + '/keys/' + AKV_KEY_NAME + keyVersion +
            '/sign?api-version=' + AKV_API_VERSION;

        HttpRequest req = new HttpRequest();
        req.setEndpoint(endpoint);
        req.setMethod('POST');
        req.setHeader('Authorization', 'Bearer ' + akvToken);
        req.setHeader('Content-Type', 'application/json');
        req.setTimeout(10000);

        // value = base64url(SHA-256(signingInput))
        // AKV applies RSASSA-PKCS1-v1_5 on the provided digest
        req.setBody(JSON.serialize(new Map<String, String>{
            'alg'   => 'RS256',
            'value' => digestB64
        }));

        HttpResponse res = new Http().send(req);
        if (res.getStatusCode() != 200) {
            throw new KvJwtException(
                'AKV sign failed [' + res.getStatusCode() + ']: ' + res.getBody()
            );
        }

        // AKV returns the signature already base64url-encoded
        Map<String, Object> parsed =
            (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
        return (String) parsed.get('value');
    }

    // ─── Utilities ────────────────────────────────────────────────────────────
    private static String base64UrlEncode(Blob input) {
        return EncodingUtil.base64Encode(input)
            .replace('+', '-')
            .replace('/', '_')
            .replaceAll('=+$', '');
    }

    private static String encode(String value) {
        return EncodingUtil.urlEncode(value, 'UTF-8');
    }

    private static String generateJti() {
        return EncodingUtil.convertToHex(Crypto.generateAesKey(128));
    }

    private static String getProtectedSetting(String key) {
        // Replace with your actual Protected Custom Setting retrieval logic
        // e.g. AKV_Settings__c.getInstance().AKV_TENANT_ID__c
        AKV_Settings__c s = AKV_Settings__c.getOrgDefaults();
        if (key == 'AKV_TENANT_ID')     return s.Tenant_Id__c;
        if (key == 'AKV_CLIENT_ID')     return s.Client_Id__c;
        if (key == 'AKV_CLIENT_SECRET') return s.Client_Secret__c;
        return null;
    }

    public class KvJwtException extends Exception {}
}
```

---

## Comparison: both approaches side by side

| | Certificate in Salesforce | AKV Sign API (this approach) |
|---|---|---|
| Private key location | Salesforce CKM | Azure Key Vault only |
| What Salesforce stores | Private key (RSA) | `client_secret` (for AKV auth) |
| Callouts per token request | 1 (token endpoint) | 3 (AKV auth + AKV sign + token endpoint) |
| Key rotation | Re-export JKS, re-import | Rotate in AKV only, zero Salesforce change |
| Audit trail of signings | None | Full AKV audit log per signing operation |
| Latency | Lower | Higher (~100–200ms extra) |
| Best for | Simplicity, low latency | Compliance, key governance, zero-export policy |

---

## Remote Site Settings required for this approach

```
https://login.microsoftonline.com   → Entra ID (both AKV auth and target token)
https://your-vault.vault.azure.net  → Azure Key Vault Sign API
```
