# AZURE

Voici les trois types d'objets dans Azure Key Vault, leurs rôles et leurs différences :

**Keys (Clés)** servent à effectuer des opérations cryptographiques — chiffrer, déchiffrer, signer, vérifier. Elles ne quittent jamais le vault. C'est la brique de sécurité la plus bas niveau.

**Secrets** sont de simples paires clé/valeur stockées de façon sécurisée — mots de passe, chaînes de connexion, tokens API. Votre application les lit et les utilise directement.

**Certificates** combinent les deux : un certificat TLS/SSL contient une clé publique/privée et des métadonnées de validité. Key Vault peut gérer leur cycle de vie (création, renouvellement automatique).---

### Résumé comparatif rapide

| | **Keys** | **Secrets** | **Certificates** |
|---|---|---|---|
| Ce que c'est | Clé cryptographique | Valeur texte sensible | TLS/SSL complet |
| L'app reçoit | Jamais la clé (opère via API) | La valeur en clair | Le certificat + clé privée |
| Renouvellement auto | ❌ | ❌ | ✅ |
| Coût extra (HSM) | ✅ Premium | ❌ | ❌ |

---

En fait, pour votre cas d'usage précis, il y a une distinction importante à faire : ce ne sont **pas des Keys**, mais bien des **Certificates** dans Azure Key Vault. Voici pourquoi.

---

## Le point clé : Keys vs Certificates pour votre besoin

Le problème avec les **Keys** de Key Vault, c'est une règle de sécurité fondamentale :

> **La clé privée ne quitte JAMAIS le vault.** On ne peut effectuer des opérations cryptographiques qu'en appelant l'API Key Vault. On ne peut pas exporter la clé privée.

Or, votre besoin est justement d'**exporter** la clé privée pour la stocker dans Salesforce (Certificate and Key Management). Les Keys sont donc incompatibles avec ce besoin.

---

## Ce dont vous avez besoin : un Certificate Key Vault

Un **Certificate** dans Azure Key Vault est différent d'un Key :

| | Key | Certificate |
|---|---|---|
| Clé privée exportable | ❌ jamais | ✅ si la policy le permet |
| Format d'export | public key JWK uniquement | PFX / PEM complet |
| Contient un X.509 | ❌ | ✅ (requis par Salesforce) |
| Renouvellement auto | ❌ | ✅ |

Salesforce Certificate and Key Management attend un fichier **X.509** (`.crt` / `.pem`) ou **PFX** (`.p12`). Seul le type **Certificate** de Key Vault peut fournir cela.

---

## Le flux correct pour votre cas

```
1. Générer ou importer un Certificate dans Azure Key Vault
        (RSA 2048+ bits, self-signed ou signé par une CA)

2. Configurer la policy du certificate : exportable = true

3. Exporter le certificate depuis Key Vault
        → format PFX (contient clé privée + certificat public)

4. Importer dans Salesforce → Setup > Certificate and Key Management
        → Salesforce stocke le certificat (avec clé privée)

5. Dans Salesforce, configurer l'Auth. Provider / Named Credential
        → sélectionner ce certificat pour signer les JWT Bearer assertions

6. Salesforce signe le JWT avec la clé privée
        → envoie le JWT à l'Authorization Server (ex: Entra ID, Auth0...)

7. L'Authorization Server valide avec la clé publique connue
        → retourne un access_token

8. Salesforce utilise l'access_token pour les callouts REST
```

---

## Ce qu'il faut faire concrètement dans Key Vault

```bash
# Créer un certificat self-signed exportable dans Key Vault
az keyvault certificate create \
  --vault-name <nom-vault> \
  --name "salesforce-jwt-cert" \
  --policy '{
    "keyProperties": {
      "exportable": true,
      "keyType": "RSA",
      "keySize": 2048,
      "reuseKey": false
    },
    "secretProperties": {
      "contentType": "application/x-pkcs12"
    },
    "x509CertificateProperties": {
      "subject": "CN=salesforce-jwt",
      "validityInMonths": 12
    }
  }'

# Exporter le PFX complet (clé privée incluse)
az keyvault secret download \
  --vault-name <nom-vault> \
  --name "salesforce-jwt-cert" \
  --encoding base64 \
  --file salesforce-jwt-cert.pfx
```

Le secret (`az keyvault secret download`) — et non `az keyvault certificate download` — est utilisé car c'est lui qui contient le PFX avec la clé privée.

<img width="363" height="832" alt="image" src="https://github.com/user-attachments/assets/d64e5511-1aea-4b4f-9d22-de4aad058271" />


---


<img width="1440" height="1102" alt="image" src="https://github.com/user-attachments/assets/c17213ff-9972-4c88-afb6-c0a0f51372b5" />
