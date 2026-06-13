<p align="center">
  <img src="https://img.shields.io/badge/UCAD-École%20Supérieure%20Polytechnique-blue?style=for-the-badge" alt="ESP UCAD">
  <img src="https://img.shields.io/badge/Master%201-SSI%20%7C%20Sécurité%20Web%20%26%20Protocoles-red?style=for-the-badge" alt="M1 SSI">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Identity%20Provider-Keycloak%20%7C%20Authentik-orange?style=flat-square&logo=keycloak" alt="IdP">
  <img src="https://img.shields.io/badge/Protocols-SAML%202.0%20%7C%20OIDC-blue?style=flat-square" alt="Protocoles">
  <img src="https://img.shields.io/badge/Directory-OpenLDAP%20%7C%20Active%20Directory-blueviolet?style=flat-square&logo=openldap" alt="LDAP">
  <img src="https://img.shields.io/badge/Infrastructure-Docker%20Compose%20%7C%20Nginx-lightgrey?style=flat-square&logo=docker" alt="Docker">
</p>

---

# 🔑 Solution SSO d'Entreprise - SAML & OIDC Core Implementation

Ce dépôt héberge l'infrastructure de centralisation des identités et de Fédération d'Accès de notre entreprise. Ce projet s'inscrit dans le cadre du module **Sécurité des protocoles de communication et du Web**.

L'architecture s'appuie sur un Fournisseur d'Identité central (**Identity Provider - IdP**) fédéré à un annuaire **LDAP / Active Directory** agissant comme source unique de vérité. La solution interconnecte de manière sécurisée plusieurs applications métiers via les protocoles de confiance **SAML 2.0** et **OpenID Connect (OIDC)**, tout en durcissant la sécurité par des politiques d'accès contextuel et d'authentification forte (MFA).

---

## 👥 Équipe de Projet & Répartition des Tâches
Ce projet est réalisé sous la direction de notre enseignant **M. Serigne Mouhamadane Diop** à l'École Supérieure Polytechnique (ESP) de Dakar.

### 📋 Matrice des Responsabilités (RACI)

| Membre | Rôle Principal | Sprint 1 (Fondations & SAML) | Sprint 2 (OIDC & Durcissement) |
| :--- | :--- | :--- | :--- |
| **M1: Awa NIANG** | **Infrastructure & IdP** | Déploiement Keycloak/Docker, création du Realm, des utilisateurs et rôles de test. | Configuration Reverse Proxy (Nginx/Traefik) avec TLS 1.3, support réseau global et Fédération d'IdP externes (Google/GitHub). |
| **M2: Mamadou Kone NDOUR** | **Intégration SAML 2.0** | Intégration de Nextcloud en SAML 2.0, configuration des métadonnées, de l'ACS et du mapping d'attributs (`uid`, `email`). | Intégration de Jenkins en SAML via plugin officiel, gestion des rôles par claims et co-rédaction de la procédure d'enrôlement. |
| **M3: Fatou NDOUR** | **Intégration OIDC** | Intégration de GitLab en OpenID Connect, configuration de l'Authorization Code Flow et du mécanisme PKCE. | Intégration de Grafana en OIDC, mapping des rôles utilisateurs, configuration du point de terminaison `userinfo` et des tokens de rafraîchissement. |
| **M4: Mame Thierno DIOP** | **Sécurité, MFA & SLO** | Configuration de l'authentification forte MFA TOTP (Google Authenticator), politique de complexité des mots de passe et implémentation du Single Logout (SLO). | Définition des politiques d'accès conditionnel (filtrage IP, restrictions horaires), audit de sécurité IdP et rédaction du plan de bascule de production. |
| **M5: Mouhamad M SEYDI** | **Documentation & Coordination** | Cartographie des applications du système, rédaction des justifications protocolaires et design des diagrammes de séquence SAML/OIDC. | Gestion des scénarios de démonstration technique, consolidation du guide de mise en œuvre final et tests de validation de bout en bout. |

---

## 🏗️ Architecture Générale du Système

Le flux logique de l'infrastructure est orchestré selon le modèle d'authentification unique suivant :
1. **Utilisateur ➔ Identity Provider (Keycloak / Authentik) :** L'utilisateur initie une demande d'accès standard ou SSO.
2. **IdP ➔ Annuaire LDAP / AD :** L'IdP central interroge l'annuaire (base de données de référence) pour valider l'authentification et en extraire les attributs (groupes, rôles).
3. **IdP ➔ Applications Métiers (SAML / OIDC) :**
   * Pour les plateformes compatibles **SAML** (*Nextcloud, Jenkins*) : Émission d'une **assertion XML signée cryptographiquement**.
   * Pour les plateformes modernes **OIDC** (*GitLab, Grafana*) : Transmission d'un jeton **JWT (ID Token + Access Token)** via un flux sécurisé avec validation **PKCE**.

---

## 🔐 Matrice de Sécurité Appliquee

| Composant | Protocole / Mécanisme appliqué | Objectif de Sécurité |
| :--- | :--- | :--- |
| **Transit Réseau** | Transport exclusivement sous **TLS 1.3** avec des suites de chiffrement fortes. | Protection anti-écoute et prévention des attaques de type Man-In-The-Middle (MITM). |
| **Protection Applicative** | **PKCE (Proof Key for Code Exchange)** obligatoire sur les flux OIDC. | Interception et vol de jetons d'autorisation interdits, même sur les clients vulnérables. |
| **Contrôle d'Accès** | **RBAC (Role-Based Access Control)** imbriqué dans l'annuaire de jetons. | Respect du principe du moindre privilège lors de la propagation des claims. |
| **Double Facteur** | **MFA TOTP** forcé selon des critères contextuels précis (Accès admin ou IP hors ESP). | Atténuation drastique des risques liés au vol ou à la compromission d'identifiants. |
| **Cycle de vie de Session** | **SLO (Single Logout)** synchrone sur l'ensemble des applications liées. | Invalidation globale instantanée des tokens pour éviter les détournements de session résiduelle. |

---

## 🛠️ Déploiement Rapide de l'Infrastructure (Environnement Local)

### 1. Prérequis
* Docker & Docker Compose installés sur votre machine hôte.
* Les ports `8080` (Keycloak), `389`/`636` (LDAP) libres de toute occupation.

### 2. Clonage et Lancement
```bash
git clone [https://github.com/votre-dossier/sso-security-project.git](https://github.com/votre-dossier/sso-security-project.git)
cd sso-security-project
docker-compose up -d
