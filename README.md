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

<img width="1440" height="1102" alt="image" src="https://github.com/user-attachments/assets/c17213ff-9972-4c88-afb6-c0a0f51372b5" />
