# Jenkins — Intégration SAML 2.0

**M2 — Intégration SAML | Sprint 2 | Master 1 SSI — ESP-UCAD**
**Responsable : Mamadou Kone NDOUR**

> Guide complet de mise en place de Jenkins comme Service Provider (SP) SAML 2.0 face à Keycloak (IdP). Basé sur le déploiement réel — tous les pièges rencontrés sont documentés avec leur solution exacte.

---

## Architecture

```
Navigateur
    │
    ▼ HTTPS (443)
┌──────────────────────────────┐
│  Nginx — jenkins.entreprise.local  │
└─────────────┬────────────────┘
              │ HTTP interne
              ▼
    ┌──────────────────┐         ┌──────────────────────┐
    │  Jenkins:8080    │◄──SAML─►│  Keycloak:8080       │
    │  (SP)            │         │  idp.entreprise.local │
    └──────────────────┘         └──────────────────────┘
```

> ⚠️ **Deux URLs Keycloak selon le contexte :**
> - `http://keycloak:8080` → communication interne Docker (metadata)
> - `https://idp.entreprise.local` → redirections navigateur

---

## Prérequis

- [ ] Branche `feat/m1-infra` déployée
- [ ] Nextcloud SAML fonctionnel (référence de configuration validée)
- [ ] Réseau Docker `sso-network` existant
- [ ] Entrées DNS locales :
  ```
  127.0.0.1    jenkins.entreprise.local
  ```
- [ ] Plugins Jenkins installés : **SAML Plugin** + **Role-based Authorization Strategy**

---

## Étape 1 — Démarrer Jenkins

```bash
cd saml/jenkins
docker compose up -d
```

---

## Étape 2 — Mettre à jour nginx.conf + redémarrer Nginx

Le bloc Jenkins est déjà dans le `nginx.conf` livré. Redémarrez Nginx depuis `infra/` :

```bash
docker compose up -d --force-recreate nginx
```

Vérifiez :
```bash
curl -k -I https://jenkins.entreprise.local
# Attendu : HTTP/1.1 200 OK avec X-Jenkins: 2.xxx
```

---

## Étape 3 — Installer les plugins Jenkins

Accédez à `https://jenkins.entreprise.local` :

```
Administrer Jenkins → Plugins → Available plugins
```

Installez :
1. **SAML Plugin**
2. **Role-based Authorization Strategy**

Redémarrez Jenkins après installation.

---

## Étape 4 — Configurer l'URL Jenkins

**Administrer Jenkins → Configurer le système → Jenkins URL** :
```
https://jenkins.entreprise.local/
```

---

## Étape 5 — Créer le client SAML dans Keycloak

**`https://idp.entreprise.local/admin` → realm `entreprise` → Clients → Create client**

### Settings

| Champ | Valeur |
|---|---|
| Client type | `SAML` |
| Client ID | `https://jenkins.entreprise.local/securityRealm/finishLogin` |
| Name | `Jenkins` |
| Root URL | `https://jenkins.entreprise.local` |
| Home URL | `https://jenkins.entreprise.local` |
| Valid redirect URIs | `https://jenkins.entreprise.local/*` |
| Master SAML Processing URL | `https://jenkins.entreprise.local/securityRealm/finishLogin` |

### SAML capabilities

| Champ | Valeur |
|---|---|
| Name ID format | `username` |
| Force POST binding | `On` |
| Sign assertions | `On` |
| **Client signature required** | **`Off`** |
| Encrypt assertions | `Off` |

> ⚠️ **Piège — Client signature required**
>
> Jenkins signe ses requêtes SAML avec son propre certificat. Si `Client signature required` est `On` sans le bon certificat importé, Keycloak rejette avec `invalid_signature`. Désactivez via l'API si l'UI échoue :
>
> ```bash
> docker exec sso-keycloak /opt/keycloak/bin/kcadm.sh update clients/<UUID> \
>   -r entreprise -s "attributes.\"saml.client.signature\"=false" \
>   --server http://localhost:8080 --realm master \
>   --user admin --password <KC_ADMIN_PASSWORD>
> ```

### Logout settings

| Champ | Valeur |
|---|---|
| Front channel logout | `Off` |

> ℹ️ Le SLO Jenkins via front channel génère une erreur `403 No valid crumb` (protection CSRF Jenkins). On le désactive pour éviter l'erreur `Logout failed` lors de la déconnexion depuis Nextcloud.

### ⚠️ Retirer le scope `role_list`

Dans **Client scopes**, passez `role_list` en **None** — même raison que Nextcloud.

### Mappers (Dedicated scopes → Add mapper)

| Nom | Type | Propriété | SAML Attribute Name | Option clé |
|---|---|---|---|---|
| uid | User Property | username | `uid` | — |
| email | User Property | email | `email` | — |
| groups | Group list | — | `groups` | **Single Group Attribute: On** |
| roles | Role list | — | `roles` | **Single Role Attribute: On** |

---

## Étape 6 — Configurer SAML dans Jenkins

**Administrer Jenkins → Configurer la sécurité globale → SAML 2.0**

Récupérez le XML metadata Keycloak :
```bash
curl -s http://localhost:8180/realms/entreprise/protocol/saml/descriptor
```

| Champ | Valeur |
|---|---|
| **IdP Metadata** | Sélectionnez **XML** → collez le XML complet |
| Username Attribute | `uid` |
| Display Name Attribute | `uid` |
| Group Attribute | `groups` |
| Email Attribute | `email` |
| Data Binding Method | `HTTP-POST` |
| Logout URL | `https://idp.entreprise.local/realms/entreprise/protocol/openid-connect/logout` |

Cochez **Advanced Configuration** :

| Champ | Valeur |
|---|---|
| SP Entity ID | `https://jenkins.entreprise.local/securityRealm/finishLogin` |

> ⚠️ **Ne pas utiliser l'URL metadata** dans le champ "IdP Metadata URL" — Jenkins ne peut pas joindre `localhost:8180` depuis le conteneur, et `keycloak:8080` génère des URLs de redirection avec un hostname inconnu du navigateur. Utilisez toujours le **mode XML**.

---

## Étape 7 — Configurer les rôles

### Activer Role-based Authorization

**Administrer Jenkins → Configurer la sécurité globale → Authorization → Role-Based Strategy**

### Manage Roles

| Rôle | Permissions |
|---|---|
| `admin` | Global → Administer |
| `dev` | Global → Read + Job (Build, Configure, Discover, Read, Workspace) |
| `ops` | Global → Read + Job (Build, Read, Workspace) |

### Assign Roles

| User/Group | admin | dev | ops |
|---|---|---|---|
| Anonymous | ❌ | ❌ | ❌ |
| Authenticated Users | ❌ | ❌ | ❌ |
| user-admin | ✅ | ❌ | ❌ |
| user-dev | ❌ | ✅ | ❌ |
| user-ops | ❌ | ❌ | ✅ |

---

## Étape 8 — Test

1. Onglet privé → `https://jenkins.entreprise.local`
2. Accepter l'avertissement certificat
3. Jenkins redirige automatiquement vers Keycloak
4. Se connecter avec `user-admin / Test1234!` → dashboard complet
5. Se connecter avec `user-dev / Test1234!` → accès limité (pas d'administration)

---

## Récupérer l'accès Jenkins en cas de blocage

Si la config SAML vous bloque dehors :

```bash
docker exec sso-jenkins bash -c "sed -i 's/<useSecurity>true<\/useSecurity>/<useSecurity>false<\/useSecurity>/' /var/jenkins_home/config.xml"
docker restart sso-jenkins
# Attendre 30 secondes → accéder à https://jenkins.entreprise.local
```

---

## Diagnostic

```bash
docker logs sso-keycloak --tail 30 2>&1 | grep -i "error\|invalid\|saml"
```

| Erreur | Cause | Solution |
|---|---|---|
| `Invalid requester` | Client ID Keycloak ≠ Issuer Jenkins | Aligner : `https://jenkins.entreprise.local/securityRealm/finishLogin` |
| `invalid_signature` | Certificat Jenkins non reconnu | Désactiver `Client signature required` via kcadm.sh |
| `Was not possible to get the Metadata` | URL interne inaccessible | Utiliser mode XML au lieu de l'URL |
| `Access Denied` après login | Utilisateur sans rôle assigné | Vérifier Assign Roles |
| `403 No valid crumb` sur `/samlLogout` | Protection CSRF Jenkins | Désactiver Front channel logout (étape 5) |
| `Oops!` sur `/securityRealm/finishLogin` | Keycloak envoie une réponse SAML inattendue | Vérifier que Client ID et SP Entity ID correspondent exactement |
