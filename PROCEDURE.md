Voici la procédure complète de bout en bout.

---

## Étape 1 — Créer le certificat dans Azure Key Vault

```bash
# Créer la policy de certificat dans un fichier JSON
cat > cert-policy.json << 'EOF'
{
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
    "subject": "CN=salesforce-jwt-callout",
    "keyUsage": ["digitalSignature"],
    "validityInMonths": 12
  },
  "issuerParameters": {
    "name": "Self"
  },
  "lifetimeActions": [
    {
      "trigger": { "daysBeforeExpiry": 30 },
      "action": { "actionType": "EmailContacts" }
    }
  ]
}
EOF

# Créer le certificat
az keyvault certificate create \
  --vault-name <nom-vault> \
  --name "sf-jwt-callout" \
  --policy @cert-policy.json
```

Vérifier que le certificat est bien créé :

```bash
az keyvault certificate show \
  --vault-name <nom-vault> \
  --name "sf-jwt-callout" \
  --query "{status: attributes.enabled, expiry: attributes.expires}" \
  --output table
```

---

## Étape 2 — Exporter les deux fichiers nécessaires

Vous avez besoin de **deux exports distincts** :

```bash
# Export 1 : le certificat public (.cer) → pour l'Authorization Server
az keyvault certificate download \
  --vault-name <nom-vault> \
  --name "sf-jwt-callout" \
  --encoding DER \
  --file sf-jwt-callout-public.cer

# Export 2 : le PFX complet (clé privée + certificat) → pour Salesforce
# IMPORTANT : passer par le secret (pas le certificate) pour avoir la clé privée
az keyvault secret download \
  --vault-name <nom-vault> \
  --name "sf-jwt-callout" \
  --encoding base64 \
  --file sf-jwt-callout.pfx
```

> Le fichier `.cer` ne contient que la clé publique — vous pouvez le partager sans risque. Le fichier `.pfx` contient la clé privée — à traiter comme un secret.

Si votre PFX n'a pas de mot de passe et que Salesforce en exige un :

```bash
openssl pkcs12 -in sf-jwt-callout.pfx -out temp.pem -nodes -passin pass:
openssl pkcs12 -export -in temp.pem -out sf-jwt-callout-protected.pfx -passout pass:MonMotDePasse
rm temp.pem
```

---

## Étape 3 — Importer le certificat dans Salesforce

Dans Salesforce :

1. Aller dans **Setup** → rechercher **Certificate and Key Management**
2. Cliquer sur **Import from Keystore**
3. Choisir le fichier `sf-jwt-callout.pfx`
4. Saisir le mot de passe (ou laisser vide si aucun)
5. Cliquer **Save**

Le certificat apparaît maintenant dans la liste avec son nom (`sf-jwt-callout`) et sa date d'expiration. Salesforce stocke la clé privée de façon sécurisée et ne vous permettra plus de l'exporter.

---

## Étape 4 — Configurer le Auth. Provider (si cible OAuth2 externe)

Si l'Authorization Server cible est **Entra ID / Azure AD** ou tout autre serveur OAuth2 :

1. **Setup** → **Auth. Providers** → **New**
2. Choisir le type selon votre cible (ex: `Open ID Connect`)
3. Renseigner :
   - `Consumer Key` : le `client_id` de votre app enregistrée
   - `Consumer Secret` : laisser vide (JWT Bearer n'en a pas besoin)

---

## Étape 5 — Configurer la Named Credential

1. **Setup** → **Named Credentials** → **New**
2. Renseigner :

| Champ | Valeur |
|---|---|
| Label | Nom de votre choix |
| URL | URL de base de l'API cible |
| Identity Type | `Named Principal` |
| Authentication Protocol | `JWT Token Exchange` |
| Token Endpoint URL | URL du token endpoint OAuth2 |
| Issuer | `client_id` de votre application |
| Named Principal Subject | identifiant de l'utilisateur / service account |
| Audiences | URL audience attendue par le serveur OAuth2 |
| Token Valid for | `300` secondes (recommandé) |
| JWT Signing Certificate | **Sélectionner `sf-jwt-callout`** ← votre certificat importé |

3. Cliquer **Save**

---

## Étape 6 — Enregistrer le certificat public côté Authorization Server

Le serveur OAuth2 doit connaître votre clé publique pour valider les JWT signés par Salesforce. Selon la cible :

**Si Entra ID / Azure AD :**
```
Azure Portal
→ App Registrations → votre app
→ Certificates & secrets → Certificates
→ Upload certificate → sélectionner sf-jwt-callout-public.cer
```

**Si un autre système OAuth2 :**
Fournir le fichier `sf-jwt-callout-public.cer` à l'administrateur du serveur.

---

## Étape 7 — Utiliser la Named Credential dans un Apex callout

```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:VotreNamedCredential/api/endpoint');
req.setMethod('GET');
req.setHeader('Accept', 'application/json');

Http http = new Http();
HttpResponse res = http.send(req);
System.debug(res.getBody());
```

Salesforce se charge automatiquement de :
- construire le JWT (header + claims)
- le signer avec la clé privée du certificat importé
- l'envoyer au token endpoint pour obtenir l'`access_token`
- injecter le `Bearer` token dans le header `Authorization` de votre callout

---

## Récapitulatif du flux complet

```
Azure Key Vault
  └─ Certificate "sf-jwt-callout" (RSA 2048, exportable)
        │
        ├─ Export .cer (clé publique)  ──────────────────► Authorization Server
        │                                                   (Entra ID, etc.)
        │                                                   enregistre la clé publique
        │
        └─ Export .pfx (clé privée + cert) ─────────────► Salesforce
                                                           Certificate & Key Management
                                                                │
                                                           Named Credential (JWT Token Exchange)
                                                                │
                                                           Apex callout
                                                           → signe JWT avec clé privée
                                                           → échange contre access_token
                                                           → appel API sécurisé
```
Le diagramme montre les trois étapes du flux :

**Dans Azure Key Vault**, le Certificate `sf-jwt-callout` contient les deux éléments créés ensemble — la clé privée RSA 2048 et le certificat X.509 public. Ils sont indissociables dans le Certificate object.

**La commande `az keyvault secret download --encoding base64`** est le point de fusion : c'est elle qui extrait les deux éléments simultanément depuis le secret sous-jacent du Certificate (et non depuis `az keyvault certificate download` qui ne donne que la clé publique). Le résultat est un fichier **PFX / .p12** — le seul format qui encapsule clé privée + certificat ensemble et que Salesforce sait importer.

**Dans Salesforce**, après import via *Certificate & Key Management*, la clé privée et le certificat sont stockés et protégés par la plateforme. La Named Credential peut ensuite les utiliser pour signer les JWT Bearer assertions à chaque callout OAuth2.

---

<img width="1440" height="698" alt="image" src="https://github.com/user-attachments/assets/8bc29ff0-9d26-4546-806a-955737a4913f" />

