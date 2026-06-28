# Procédure d'enrôlement d'une nouvelle application SSO

**Livrable M2 — Intégration SAML | Master 1 SSI — ESP-UCAD**
**Auteur : Mamadou Kone NDOUR**

---

## Objectif

Cette procédure décrit les étapes à suivre pour intégrer toute nouvelle application comme **Service Provider (SP) SAML 2.0** dans l'infrastructure SSO d'entreprise déployée avec Keycloak.

Elle est générique et applicable à n'importe quelle application supportant SAML 2.0 (Nextcloud, Jenkins, GitLab, Grafana, Jira, etc.). Les exemples concrets s'appuient sur Nextcloud et Jenkins, déjà intégrés dans ce projet.

---

## Vue d'ensemble du flux SAML

```
┌──────────────┐   1. GET /login          ┌─────────────────┐
│  Navigateur  │ ──────────────────────►  │   Application   │
│              │                          │   (SP)          │
│              │ ◄──────────────────────  │                 │
│              │   2. Redirect + SAMLRequest               │
│              │                          └─────────────────┘
│              │   3. POST SAMLRequest
│              │ ──────────────────────►  ┌─────────────────┐
│              │                          │    Keycloak     │
│              │ ◄──────────────────────  │    (IdP)        │
│              │   4. Login form          │                 │
│              │                          │                 │
│              │   5. Credentials         │                 │
│              │ ──────────────────────►  │                 │
│              │                          │                 │
│              │ ◄──────────────────────  │                 │
│              │   6. SAMLResponse        └─────────────────┘
│              │      (assertion signée)
│              │   7. POST SAMLResponse
│              │ ──────────────────────►  ┌─────────────────┐
│              │                          │   Application   │
│              │ ◄──────────────────────  │   (SP)          │
└──────────────┘   8. Session créée       └─────────────────┘
```

---

## Phase 1 — Préparation de l'infrastructure

### 1.1 Vérifier les prérequis

```bash
# Keycloak accessible
curl -s http://localhost:8180/health/ready
# Attendu : {"status":"UP"}

# Réseau Docker disponible
docker network ls | grep sso-network

# Certificat TLS valide
docker exec sso-nginx openssl x509 -in /etc/nginx/certs/tls.crt -noout -text | grep DNS
```

### 1.2 Choisir le nom de domaine de la nouvelle application

Format recommandé : `<app>.entreprise.local`

Exemples :
- `nextcloud.entreprise.local`
- `jenkins.entreprise.local`
- `grafana.entreprise.local`
- `gitlab.entreprise.local`

### 1.3 Ajouter le domaine dans le fichier hosts

**Windows** (en tant qu'administrateur) :
```
C:\Windows\System32\drivers\etc\hosts
```

Ajouter :
```
127.0.0.1    <app>.entreprise.local
```

### 1.4 Régénérer le certificat TLS avec le nouveau SAN

```bash
# Windows — depuis le dossier infra/
docker run --rm -v "%cd%\certs:/out" alpine/openssl req -x509 -nodes -days 365 ^
  -newkey rsa:2048 -keyout /out/tls.key -out /out/tls.crt ^
  -subj "/C=SN/ST=Dakar/L=Dakar/O=ESP-UCAD/CN=idp.entreprise.local" ^
  -addext "subjectAltName=DNS:idp.entreprise.local,DNS:nextcloud.entreprise.local,DNS:jenkins.entreprise.local,DNS:<app>.entreprise.local"
```

> ⚠️ Listez **tous** les domaines existants + le nouveau dans le SAN — régénérer le certificat invalide l'ancien.

---

## Phase 2 — Déploiement de l'application

### 2.1 Créer le docker-compose.yml de l'application

```yaml
services:
  <app>:
    image: <image>:<version>
    container_name: sso-<app>
    restart: unless-stopped
    environment:
      # variables propres à l'application
    volumes:
      - <app>_data:/data
    networks:
      - sso-network

volumes:
  <app>_data:

networks:
  sso-network:
    external: true
    name: sso-network
```

### 2.2 Ajouter le bloc dans nginx.conf

Dans `infra/nginx.conf`, ajoutez un nouveau bloc `server` :

```nginx
server {
  listen 443 ssl;
  server_name <app>.entreprise.local;
  ssl_certificate     /etc/nginx/certs/tls.crt;
  ssl_certificate_key /etc/nginx/certs/tls.key;
  ssl_protocols       TLSv1.2 TLSv1.3;

  location / {
    proxy_pass         http://<app>:<port_interne>;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto https;
    proxy_read_timeout 3600s;
  }
}
```

### 2.3 Démarrer les services

```bash
# Démarrer l'application
cd saml/<app>
docker compose up -d

# Recharger Nginx
cd infra
docker compose up -d --force-recreate nginx

# Vérifier
curl -k -I https://<app>.entreprise.local
```

---

## Phase 3 — Enrôlement dans Keycloak

### 3.1 Créer le client SAML

**`https://idp.entreprise.local/admin` → realm `entreprise` → Clients → Create client**

| Champ | Valeur |
|---|---|
| Client type | `SAML` |
| Client ID | `https://<app>.entreprise.local/<chemin_metadata>` |
| Root URL | `https://<app>.entreprise.local` |
| Valid redirect URIs | `https://<app>.entreprise.local/*` |

> Le `<chemin_metadata>` dépend de l'application :
> - Nextcloud : `/apps/user_saml/saml/metadata`
> - Jenkins : `/securityRealm/finishLogin`
> - Grafana : `/saml/metadata`
> - GitLab : `/users/auth/saml/metadata`

### 3.2 Configurer les capabilities SAML

| Champ | Valeur recommandée |
|---|---|
| Name ID format | `unspecified` (ou `username` selon l'app) |
| Force POST binding | `On` |
| Sign assertions | `On` |
| Client signature required | `Off` (évite les erreurs de certificat) |
| Encrypt assertions | `Off` |

### 3.3 ⚠️ Retirer le scope `role_list` — OBLIGATOIRE

Dans **Client scopes**, passez `role_list` en **None**.

**Pourquoi :** Sans cette étape, Keycloak génère une balise `<saml:Attribute Name="Role">` par rôle utilisateur. Le parser SAML de la plupart des applications rejette ce format avec l'erreur :
```
Found an Attribute element with duplicated Name
```

### 3.4 Créer les mappers d'attributs

Dans **Dedicated scopes → Mappers** :

| Mapper | Type | SAML Attribute Name | Option clé |
|---|---|---|---|
| uid | User Property (username) | `uid` | — |
| email | User Property (email) | `email` | — |
| displayName | User Property (firstName) | `displayName` | — |
| groups | Group list | `groups` | **Single Group Attribute: On** |

> `Single Group Attribute: On` est obligatoire pour éviter les doublons d'attributs.

### 3.5 Configurer le logout (SLO)

Dans **Logout settings** :

| Champ | Valeur |
|---|---|
| Front channel logout | `On` si l'application supporte le SLO |
| Logout Service Redirect Binding URL | `https://<app>.entreprise.local/<chemin_slo>` |

> Si l'application ne supporte pas le SLO, mettez `Front channel logout` à `Off` pour éviter les erreurs lors de la déconnexion depuis d'autres apps.

---

## Phase 4 — Configuration côté application

Cette phase varie selon l'application. Les éléments communs à configurer sont :

### 4.1 Récupérer les métadonnées Keycloak

```bash
curl -s http://localhost:8180/realms/entreprise/protocol/saml/descriptor
```

Ou depuis le navigateur : `https://idp.entreprise.local/realms/entreprise/protocol/saml/descriptor`

### 4.2 Informations à fournir à l'application

| Information | Valeur |
|---|---|
| IdP Entity ID | `https://idp.entreprise.local/realms/entreprise` |
| SSO URL | `https://idp.entreprise.local/realms/entreprise/protocol/saml` |
| SLO URL | `https://idp.entreprise.local/realms/entreprise/protocol/saml` |
| Certificat public IdP | Valeur de la balise `<X509Certificate>` dans le descriptor |
| SP Entity ID | `https://<app>.entreprise.local/<chemin_metadata>` |
| ACS URL | `https://<app>.entreprise.local/<chemin_acs>` |

### 4.3 Mapping des attributs

| Attribut SAML (Keycloak) | Champ application |
|---|---|
| `uid` | Username / Login |
| `email` | Email |
| `displayName` | Nom affiché |
| `groups` | Groupes / Rôles |

---

## Phase 5 — Validation

### 5.1 Test de connexion

```bash
# 1. Vider les cookies du navigateur
# 2. Onglet privé → https://<app>.entreprise.local
# 3. Cliquer sur "Se connecter avec SSO"
# 4. Se connecter avec user-dev / Test1234!
# 5. Vérifier que le compte est créé automatiquement
```

### 5.2 Vérification des attributs propagés

Une fois connecté, vérifiez que :
- ✅ Le nom d'utilisateur est `user-dev`
- ✅ L'email est `mamadoundour963@gmail.com`
- ✅ Les groupes/rôles sont correctement assignés

### 5.3 Test de déconnexion

```bash
# 1. Se déconnecter de l'application
# 2. Vérifier que la session Keycloak est invalidée
# 3. Tenter d'accéder à une autre application connectée
#    → doit demander une nouvelle authentification
```

### 5.4 Vérifier les logs en cas d'erreur

```bash
# Logs Keycloak
docker logs sso-keycloak --tail 30 2>&1 | grep -i "error\|invalid\|saml"

# Logs application (exemple Nextcloud)
docker exec sso-<app> tail -20 /path/to/app.log
```

---

## Référence rapide — Erreurs fréquentes

| Erreur | Cause | Solution |
|---|---|---|
| `Invalid Request` | Client ID Keycloak ≠ entityID SP | Aligner les deux valeurs exactement |
| `invalid_signature` | Mauvais certificat ou signature non attendue | Désactiver `Client signature required` |
| `Cookie was not present` | Application en HTTP | Forcer HTTPS via reverse proxy |
| `Found an Attribute element with duplicated Name` | Scope `role_list` actif | Retirer `role_list` des Client scopes |
| `Was not possible to get the Metadata` | URL metadata inaccessible depuis le conteneur | Utiliser mode XML direct |
| `Logout failed` | Application rejette la requête SLO | Désactiver `Front channel logout` |

---

## Checklist d'enrôlement

- [ ] Domaine ajouté dans `/etc/hosts` (ou DNS)
- [ ] Certificat TLS régénéré avec le nouveau SAN
- [ ] Bloc Nginx ajouté et rechargé
- [ ] Application démarrée et accessible en HTTPS
- [ ] Client SAML créé dans Keycloak avec le bon Client ID
- [ ] Scope `role_list` retiré
- [ ] Mappers créés (uid, email, displayName, groups)
- [ ] Configuration SAML côté application
- [ ] Test de login SSO réussi avec `user-dev`
- [ ] Test de login SSO réussi avec `user-admin`
- [ ] Provisioning JIT vérifié (compte créé au premier login)
- [ ] Déconnexion testée

---

*Procédure établie sur la base des intégrations réelles de Nextcloud et Jenkins dans le cadre du projet SSO d'Entreprise — Master 1 SSI, ESP-UCAD, 2026.*
