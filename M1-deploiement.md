# M1 — Guide de Déploiement Infrastructure SSO

**Membre :** M1 — Infrastructure & IdP  
**Branche :** `feat/m1-infra`

---

## Prérequis

| Outil | Version | Vérification |
|---|---|---|
| Docker | 24.x+ | `docker --version` |
| Docker Compose | 2.x+ | `docker compose version` |
| Git | 2.x+ | `git --version` |
| RAM | 4 GB min | — |
| Ports libres | 80, 443, 8180 | — |

---

## 1. Cloner et configurer

```bash
git clone https://github.com/KoneNdour/SSO-Enterprise-SAML-OIDC.git
cd SSO-Enterprise-SAML-OIDC/infra
cp .env.example .env
nano .env  # Remplir les valeurs
```

**Contenu `.env` :**
```env
# PostgreSQL Keycloak
KC_DB_PASSWORD=MotDePassePostgres

# Keycloak Admin
KC_ADMIN_PASSWORD=MotDePasseAdmin
KC_DOMAIN=idp.entreprise.local
KC_REALM=entreprise

# Nextcloud
NEXTCLOUD_ADMIN_USER=admin
NEXTCLOUD_ADMIN_PASSWORD=MotDePasseNextcloud
NEXTCLOUD_TRUSTED_DOMAINS=nextcloud.entreprise.local

# Gitea
GITEA_DB_PASSWORD=MotDePasseGitea
GITEA_DOMAIN=gitea.entreprise.local

# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=MotDePasseGrafana
GRAFANA_DOMAIN=grafana.entreprise.local
GRAFANA_OIDC_CLIENT_ID=grafana
GRAFANA_OIDC_CLIENT_SECRET=<depuis_keycloak>
```

> ⚠️ Ne jamais commiter `.env` — vérifier `.gitignore`

---

## 2. Générer les certificats TLS

```bash
mkdir -p certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/tls.key -out certs/tls.crt \
  -subj "/C=SN/ST=Dakar/O=Entreprise/CN=*.entreprise.local" \
  -addext "subjectAltName=DNS:idp.entreprise.local,\
DNS:nextcloud.entreprise.local,DNS:jenkins.entreprise.local,\
DNS:gitea.entreprise.local,DNS:grafana.entreprise.local"
```

---

## 3. Configurer le fichier hosts

**Windows** — `C:\Windows\System32\drivers\etc\hosts` (en admin) :
```
127.0.0.1   idp.entreprise.local
127.0.0.1   nextcloud.entreprise.local
127.0.0.1   jenkins.entreprise.local
127.0.0.1   gitea.entreprise.local
127.0.0.1   grafana.entreprise.local
```

---

## 4. Démarrer les services

```bash
docker compose up -d
docker compose ps  # Vérifier que tout est "running"

# Attendre ~60s que Keycloak démarre
docker logs -f sso-keycloak | grep "started"
```

---

## 5. Services déployés

| Conteneur | Image | Port | Rôle |
|---|---|---|---|
| sso-postgres | postgres:15-alpine | — | BDD Keycloak |
| sso-keycloak | keycloak:24.0 | 8180 | IdP central |
| sso-nginx | nginx:alpine | 80, 443 | Reverse proxy |
| sso-tunnel | cloudflared | — | Tunnel externe |
| sso-nextcloud | nextcloud:28 | — | SP SAML |
| sso-jenkins | jenkins:lts | — | SP SAML |
| sso-postgres-gitea | postgres:15-alpine | — | BDD Gitea |
| sso-gitea | gitea:latest | — | SP OIDC |
| sso-grafana | grafana:latest | — | SP OIDC |

---

## 6. Configuration initiale Keycloak

```
https://idp.entreprise.local/admin → admin / <KC_ADMIN_PASSWORD>

1. Créer realm "entreprise"
2. Créer rôles : admin, editor, viewer
3. Créer utilisateurs : user-admin, user-dev, user-ops
4. Assigner rôles aux utilisateurs
5. Configurer politique MDP
6. Activer MFA TOTP
```

---

## 7. Commandes utiles

```bash
docker compose down          # Arrêter (données conservées)
docker compose down -v       # ⚠️ Arrêter + supprimer données
docker compose restart nginx # Redémarrer un service
docker exec sso-nginx nginx -s reload  # Recharger Nginx

# Sauvegarder
docker run --rm -v nextcloud_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/nextcloud_$(date +%Y%m%d).tar.gz /data
```

---

## 8. Problèmes courants

| Problème | Solution |
|---|---|
| 502 Bad Gateway | Attendre 60s, `docker logs sso-keycloak` |
| Erreur SSL navigateur | Cliquer "Avancé" → "Continuer" |
| Keycloak ne démarre pas | `docker compose restart keycloak` |
| Port 443 occupé | Vérifier `netstat -tlnp \| grep 443` |
