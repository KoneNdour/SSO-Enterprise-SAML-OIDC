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
| **M5: Mouhamad Moustapha SEYDI** | **Documentation & Coordination** | Cartographie des applications du système, rédaction des justifications protocolaires et design des diagrammes de séquence SAML/OIDC. | Gestion des scénarios de démonstration technique, consolidation du guide de mise en œuvre final et tests de validation de bout en bout. |

---

## ❓ Problématique Centrale

Dans un écosystème d'entreprise où les collaborateurs doivent interagir quotidiennement avec une multitude d'applications hétérogènes (*outils collaboratifs, serveurs d'intégration continue, plateformes de supervision*), la gestion cloisonnée des identités pose un triple défi de sécurité et d'ergonomie :
1. **Multiplication et faiblesse des secrets :** L'obligation de mémoriser plusieurs paires d'identifiants pousse les utilisateurs à adopter des mots de passe faibles ou identiques, augmentant drastiquement la surface d'attaque face au phishing et au credential stuffing.
2. **Complexité de la gouvernance (Cycle de vie) :** Pour les administrateurs, l'absence de gestion centralisée complique l'enrôlement des nouveaux arrivants et, de manière plus critique, la révocabilité immédiate de l'ensemble des accès lorsqu'un collaborateur quitte l'organisation.
3. **Incompatibilité des protocoles :** Les infrastructures modernes doivent faire cohabiter de manière unifiée des outils historiques basés sur des architectures d'annuaires traditionnelles et des applications Cloud-native.

**La problématique de ce projet est donc la suivante :** *Comment concevoir et déployer une infrastructure fédérée de gestion des identités et des accès (IAM) capable de centraliser l'authentification de manière transparente pour l'utilisateur, tout en garantissant l'interopérabilité sécurisée entre des protocoles de confiance distincts (SAML 2.0 et OIDC) et un référentiel d'identité unique (LDAP) ?*

---

## 🏗️ Architecture Générale du Système

<p align="center">
  <img src="images/architecture_sso.png" alt="Schéma d'Architecture Fonctionnelle SSO" width="85%">
</p>

Le flux logique de l'infrastructure est orchestré selon le modèle d'authentification unique schématisé ci-dessus :
1. **Utilisateur ➔ Identity Provider (Keycloak / Authentik) :** L'utilisateur initie une demande d'accès standard ou SSO sur la mire d'authentification unique centralisée.
2. **IdP ➔ Annuaire LDAP / AD :** L'IdP central interroge l'annuaire (base de données de référence) pour valider l'authentification et en extraire de façon transparente les attributs utilisateur (groupes, rôles, alias).
3. **IdP ➔ Applications Métiers (SAML / OIDC) :**
   * Pour les plateformes compatibles **SAML 2.0** (*Nextcloud, Jenkins*) : Émission d'une **assertion XML signée cryptographiquement** contenant les mappers d'attributs requis.
   * Pour les plateformes modernes **OIDC** (*GitLab, Grafana*) : Transmission d'un jeton structuré **JWT (ID Token + Access Token)** via un flux de communication sécurisé soutenu par la validation **PKCE**.

---

## 🔬 Choix Technologiques & Justifications

### 1. Pourquoi Keycloak plutôt qu'Authentik ?
Bien qu'Authentik soit une solution moderne et performante, **Keycloak (développé par Red Hat)** a été retenu comme Fournisseur d'Identité (IdP) central pour notre infrastructure d'entreprise en raison de critères stricts de maturité et d'architecture :

* **Conformité et Certification Standard :** Keycloak est officiellement certifié par la *OpenID Foundation*. Sa gestion des spécifications strictes de SAML 2.0 et d'OIDC (notamment les extensions comme FAPI ou le support natif de PKCE) offre une robustesse éprouvée face aux audits de sécurité.
* **Fédération LDAP/AD Avancée :** Les mécanismes de synchronisation de Keycloak (User Federation) avec les annuaires OpenLDAP ou Active Directory sont extrêmement granulaires (importation sélective, synchronisation bidirectionnelle ou à la volée, mapping dynamique des rôles via les groupes LDAP).
* **Maturité et Écosystème Éprouvé :** Présent sur le marché depuis plus de 10 ans, Keycloak bénéficie d'une communauté massive et de retours d'expérience industriels critiques, indispensables pour garantir la pérennité d'un noyau de sécurité d'entreprise (IAM) par rapport à des solutions plus récentes.

### 2. Justification des Applications et Protocoles Choisis

L'infrastructure valide la coexistence de deux mondes protocolaires majeurs pour répondre aux besoins spécifiques de chaque outil métier :

#### A. Le Choix de SAML 2.0 (Sécurité par Échange d'Assertions XML)
* **Nextcloud & Jenkins :** Ces plateformes représentent les outils de production et de CI/CD traditionnels de l'entreprise. SAML 2.0 a été privilégié ici car il s'agit d'un standard industriel historique conçu spécifiquement pour le SSO de serveurs d'entreprise (*Enterprise-grade*). 
* **Avantage de Sécurité :** L'authentification repose sur des échanges d'assertions XML signées numériquement et optionnellement chiffrées (via des paires de clés privées/publiques), ce qui évite d'exposer ou de faire transiter les jetons d'accès directement sur le navigateur de l'utilisateur.

#### B. Le Choix d'OpenID Connect - OIDC (Sécurité par Jetons Éphémères JWT)
* **GitLab & Grafana :** Ces solutions incarnent la stack Cloud-native et DevSecOps moderne. OIDC, qui est une surcouche d'identité bâtie sur **OAuth 2.0**, est nativement conçu pour les applications web légères, les Single Page Applications (SPA) et les architectures microservices.
* **Avantage de Sécurité :** OIDC permet l'utilisation du flux d'autorisation avec **PKCE (Proof Key for Code Exchange)**. Ce mécanisme dynamique empêche l'interception du code d'autorisation par un attaquant sur le réseau ou le navigateur. De plus, l'utilisation de jetons structurés **JWT (JSON Web Tokens)** signés via l'algorithme `RS256` permet aux applications de valider localement et instantanément l'identité de l'utilisateur sans surcharger l'IdP de requêtes de vérification continuelles.

---

## 🔐 Matrice de Sécurité Appliquée

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
