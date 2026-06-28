# Nextcloud — Intégration SAML 2.0

**M2 — Intégration SAML | Sprint 1 | Master 1 SSI — ESP-UCAD**
**Responsable : Mamadou Kone NDOUR**

> Guide complet de mise en place de Nextcloud comme Service Provider (SP) SAML 2.0 face à Keycloak (IdP). Basé sur le déploiement réel — tous les pièges rencontrés sont documentés avec leur solution exacte.

---

## Architecture

```
Navigateur
    │
    ▼ HTTPS (443)
┌─────────────────────────────┐
│  Nginx — nextcloud.entreprise.local  │
└─────────────┬───────────────┘
              │ HTTP interne
              ▼
    ┌──────────────────┐         ┌──────────────────────┐
    │  Nextcloud:80    │◄──SAML─►│  Keycloak:8080       │
    │  (SP)            │         │  idp.entreprise.local │
    └──────────────────┘         └──────────────────────┘
```

**URLs d'accès :**
- Nextcloud : `https://nextcloud.entreprise.local`
- Keycloak admin : `https://idp.entreprise.local/admin`

---

## Prérequis

- [ ] Branche `feat/m1-infra` déployée (Keycloak + Postgres + Nginx)
- [ ] Realm `entreprise` créé dans Keycloak avec utilisateurs `user-dev`, `user-ops`, `user-admin`
- [ ] Réseau Docker `sso-network` existant (`docker network ls | grep sso`)
- [ ] Entrées DNS locales dans `C:\Windows\System32\drivers\etc\hosts` :
  ```
  127.0.0.1    idp.entreprise.local
  127.0.0.1    nextcloud.entreprise.local
  ```
- [ ] Certificat TLS avec SAN couvrant tous les domaines (voir étape 1)

---

## Étape 1 — Certificat TLS (si pas encore fait)

Le certificat doit couvrir tous les domaines via SAN. Générez-le depuis un conteneur jetable :

```bash
# Windows — depuis le dossier infra/
docker run --rm -v "%cd%\certs:/out" alpine/openssl req -x509 -nodes -days 365 ^
  -newkey rsa:2048 -keyout /out/tls.key -out /out/tls.crt ^
  -subj "/C=SN/ST=Dakar/L=Dakar/O=ESP-UCAD/CN=idp.entreprise.local" ^
  -addext "subjectAltName=DNS:idp.entreprise.local,DNS:nextcloud.entreprise.local,DNS:jenkins.entreprise.local,DNS:localhost"
```

Vérifiez :
```bash
docker exec sso-nginx openssl x509 -in /etc/nginx/certs/tls.crt -noout -text | grep DNS
# Attendu : DNS:idp.entreprise.local, DNS:nextcloud.entreprise.local, ...
```

---

## Étape 2 — Démarrer Nextcloud

```bash
cd saml/nextcloud
docker compose up -d
```

Vérifiez que Nextcloud rejoint le bon réseau :
```bash
docker network inspect sso-network | grep nextcloud
```

---

## Étape 3 — Mettre à jour nginx.conf

Ajoutez le bloc Nextcloud dans `infra/nginx.conf` (déjà inclus dans le fichier livré) et redémarrez Nginx :

```bash
cd infra
docker compose up -d --force-recreate nginx
```

Vérifiez :
```bash
curl -k -I https://nextcloud.entreprise.local
# Attendu : HTTP/1.1 302 avec Set-Cookie contenant "Secure"
```

---

## Étape 4 — Forcer Nextcloud en HTTPS

> ⚠️ **Point critique** : Nextcloud DOIT être servi en HTTPS. En HTTP, les cookies de session ont `SameSite=Lax` sans `Secure`, ce qui bloque le POST cross-origin depuis Keycloak → erreur `Cookie was not present` en boucle infinie.

```bash
docker exec -u www-data sso-nextcloud php occ config:system:set overwriteprotocol --value="https"
docker exec -u www-data sso-nextcloud php occ config:system:set overwrite.cli.url --value="https://nextcloud.entreprise.local"
docker exec -u www-data sso-nextcloud php occ config:system:set overwritehost --value="nextcloud.entreprise.local"
docker exec -u www-data sso-nextcloud php occ config:system:set trusted_domains 1 --value="nextcloud.entreprise.local"
docker exec -u www-data sso-nextcloud php occ config:system:set cookie_samesite --value="None"
docker restart sso-nextcloud
```

---

## Étape 5 — Installer le plugin SAML

Connectez-vous en admin sur `https://nextcloud.entreprise.local` (acceptez l'avertissement certificat auto-signé) :

```
Paramètres → Applications → "SSO & SAML authentication" → Installer
```

---

## Étape 6 — Créer le client SAML dans Keycloak

**`https://idp.entreprise.local/admin` → realm `entreprise` → Clients → Create client**

### Settings

| Champ | Valeur |
|---|---|
| Client type | `SAML` |
| Client ID | `https://nextcloud.entreprise.local/apps/user_saml/saml/metadata` |
| Name | `Nextcloud-saml` |
| Root URL | `https://nextcloud.entreprise.local` |
| Home URL | `https://nextcloud.entreprise.local` |
| Valid redirect URIs | `https://nextcloud.entreprise.local/*` |
| Master SAML Processing URL | `https://nextcloud.entreprise.local/apps/user_saml/saml/acs` |

### SAML capabilities

| Champ | Valeur |
|---|---|
| Name ID format | `unspecified` |
| Force POST binding | `On` |
| Sign assertions | `On` |
| Client signature required | `Off` |

### Logout settings

| Champ | Valeur |
|---|---|
| Front channel logout | `On` |
| Logout Service Redirect Binding URL | `https://nextcloud.entreprise.local/apps/user_saml/saml/sls` |

### ⚠️ Piège — Retirer le scope `role_list`

Dans **Client scopes**, passez `role_list` en **None**.

Sans ça, Keycloak génère une balise `<saml:Attribute Name="Role">` par rôle (5 à 7 fois). Le parser Nextcloud rejette avec :
```
Found an Attribute element with duplicated Name
```

### Mappers (Dedicated scopes → Add mapper)

| Nom | Type | Propriété | SAML Attribute Name | Option |
|---|---|---|---|---|
| uid | User Property | username | `uid` | — |
| email | User Property | email | `email` | — |
| displayName | User Property | firstName+lastName | `displayName` | — |
| groups | Group list | — | `groups` | Single Group Attribute: **On** |

---

## Étape 7 — Configurer SAML dans Nextcloud

**Paramètres → Administration → SSO & SAML**

Récupérez le certificat Keycloak :
```bash
curl -s http://localhost:8180/realms/entreprise/protocol/saml/descriptor | grep X509Certificate
```

| Champ | Valeur |
|---|---|
| Attribute to map the UID to | `uid` |
| Identifier of the IdP entity | `https://idp.entreprise.local/realms/entreprise` |
| URL Target of the IdP (SSO) | `https://idp.entreprise.local/realms/entreprise/protocol/saml` |
| URL of IdP's SLO | `https://idp.entreprise.local/realms/entreprise/protocol/saml` |
| Public X.509 certificate | *(valeur de la balise `X509Certificate` du descriptor)* |
| Display name mapping | `displayName` |
| Email mapping | `email` |
| Groups mapping | `groups` |

---

## Étape 8 — Test

1. Vider les cookies pour `nextcloud.entreprise.local`
2. Onglet privé → `https://nextcloud.entreprise.local`
3. Accepter l'avertissement certificat
4. Cliquer **Connexion avec SSO**
5. Se connecter avec `user-dev / Test1234!`

✅ Le compte est créé automatiquement (provisioning JIT) — aucune création manuelle nécessaire.

---

## Diagnostic

```bash
docker exec sso-nextcloud tail -20 /var/www/html/data/nextcloud.log
```

| Erreur | Cause | Solution |
|---|---|---|
| `Cookie was not present` | HTTP au lieu de HTTPS | Appliquer étape 4 |
| `Invalid Request` (Keycloak) | Client ID ≠ entityID Nextcloud | Vérifier `curl -k https://nextcloud.entreprise.local/apps/user_saml/saml/metadata?idp=1` |
| `Found an Attribute element with duplicated Name` | Scope `role_list` actif | Retirer `role_list` (étape 6) |
| `Location: http://localhost/login` | `overwriteprotocol` non configuré | Appliquer étape 4 |
