# Threat Model — Camelot-AD
## Méthode : STRIDE
## Date : Juin 2025 | Version : 1.0

---

## Contexte

Camelot-AD est un outil de scan Active Directory à double usage offensif/défensif,
conçu à l'image de PingCastle et Purple Knight. Il produit un score global de sécurité
AD et des rapports détaillés par catégorie de risque (Kerberos, ACL, délégations,
GPO, trusts, comptes à risque).

Périmètre POC : opérateurs locaux, API REST interne, Linux conteneurisé.
Post-POC prévu : accès distant multi-opérateurs.

---

## Composants analysés

| ID  | Composant       | Rôle                                              | Criticité |
|-----|-----------------|---------------------------------------------------|-----------|
| C01 | api-gateway     | Point d'entrée REST, auth, rate limiting          | CRITIQUE  |
| C02 | core-engine     | Orchestration des scans, gestion des sessions     | CRITIQUE  |
| C03 | audit-logger    | Journalisation intègre, valeur probante           | CRITIQUE  |
| C04 | scanner-net     | Requêtes LDAP/Kerberos/SMB vers l'AD              | ÉLEVÉ     |
| C05 | vuln-correlator | Corrélation CVE, scoring CVSS, score AD           | ÉLEVÉ     |
| C06 | log-analyzer    | Parsing des événements AD (4624, 4768, 4769...)   | MOYEN     |
| C07 | threat-intel    | IOC externes, feeds MISP/OTX                      | MOYEN     |
| C08 | report-gen      | Génération rapports PDF/HTML signés               | MOYEN     |

---

## STRIDE par composant

---

### C01 — api-gateway

| ID    | Catégorie STRIDE      | Menace                                                                 | Impact   | Probabilité |
|-------|-----------------------|------------------------------------------------------------------------|----------|-------------|
| M-001 | Spoofing              | Un attaquant usurpe l'identité d'un opérateur légitime via token volé  | CRITIQUE | MOYENNE     |
| M-002 | Tampering             | Modification des paramètres de requête pour cibler hors périmètre      | ÉLEVÉ    | MOYENNE     |
| M-003 | Repudiation           | Un opérateur nie avoir lancé un scan — logs non intègres               | ÉLEVÉ    | FAIBLE      |
| M-004 | Information Disclosure| Réponses d'API exposant des détails internes (stack trace, chemins)    | MOYEN    | ÉLEVÉE      |
| M-005 | Denial of Service     | Flood de requêtes saturant l'API et bloquant les scans légitimes       | ÉLEVÉ    | MOYENNE     |
| M-006 | Elevation of Privilege| Bypass de l'auth pour accéder à des endpoints admin non exposés        | CRITIQUE | FAIBLE      |

**Mitigations C01 :**
- M-001 ? Tokens JWT signés Ed25519, rotation par session, révocation immédiate
- M-002 ? Validation stricte des paramètres + whitelist de cibles AD
- M-003 ? Chaque requête loggée avec HMAC-SHA256 dans audit-logger
- M-004 ? Pas de stack trace en production, erreurs génériques côté client
- M-005 ? Rate limiting par IP + par token (axum middleware)
- M-006 ? Authentification obligatoire sur tous les endpoints sans exception

---

### C02 — core-engine

| ID    | Catégorie STRIDE      | Menace                                                                        | Impact   | Probabilité |
|-------|-----------------------|-------------------------------------------------------------------------------|----------|-------------|
| M-007 | Spoofing              | Un module compromis se fait passer pour core-engine auprès des autres modules | CRITIQUE | FAIBLE      |
| M-008 | Tampering             | Altération des résultats de scan avant scoring (faux négatifs volontaires)    | CRITIQUE | FAIBLE      |
| M-009 | Repudiation           | Impossibilité de prouver quel opérateur a déclenché quel scan                 | ÉLEVÉ    | MOYENNE     |
| M-010 | Information Disclosure| Fuite des credentials AD stockés en mémoire lors d'un crash                  | CRITIQUE | MOYENNE     |
| M-011 | Denial of Service     | Scan récursif ou boucle infinie sur un domaine AD complexe                    | ÉLEVÉ    | MOYENNE     |
| M-012 | Elevation of Privilege| core-engine s'exécute avec trop de droits sur le système hôte                 | CRITIQUE | MOYENNE     |

**Mitigations C02 :**
- M-007 ? mTLS entre tous les composants, certificats par module
- M-008 ? Hash des résultats bruts avant scoring, vérification à la lecture
- M-009 ? session_id + operator_id obligatoires dans chaque action loggée
- M-010 ? Credentials AD jamais persistés, injectés via Vault, zéroïsés après usage
- M-011 ? Timeout par scan + limite de profondeur de récursion configurable
- M-012 ? Exécution en contexte least-privilege, seccomp + namespaces Linux

---

### C03 — audit-logger

| ID    | Catégorie STRIDE      | Menace                                                                     | Impact   | Probabilité |
|-------|-----------------------|----------------------------------------------------------------------------|----------|-------------|
| M-013 | Spoofing              | Un composant non autorisé injecte de faux événements dans les logs         | CRITIQUE | FAIBLE      |
| M-014 | Tampering             | Modification ou suppression d'entrées de log après coup                    | CRITIQUE | MOYENNE     |
| M-015 | Repudiation           | Logs présents mais non signés — valeur probante nulle en cas de litige     | CRITIQUE | FAIBLE      |
| M-016 | Information Disclosure| Les logs contiennent des credentials AD ou des données personnelles en clair| ÉLEVÉ   | MOYENNE     |
| M-017 | Denial of Service     | Flood de logs saturant le stockage et écrasant les entrées critiques       | ÉLEVÉ    | FAIBLE      |
| M-018 | Elevation of Privilege| Accès en écriture directe à sled DB sans passer par audit-logger           | CRITIQUE | FAIBLE      |

**Mitigations C03 :**
- M-013 ? Seul core-engine peut écrire dans audit-logger via canal mTLS dédié
- M-014 ? Hash chain : chaque entrée contient le hash de la précédente (append-only)
- M-015 ? Signature Ed25519 sur chaque entrée critique + horodatage RFC 3339
- M-016 ? IP hashées (BLAKE3), credentials jamais loggés, données AD pseudonymisées
- M-017 ? Quota de log par session + alertes si volume anormal
- M-018 ? sled DB accessible uniquement par le process audit-logger, droits OS restreints

---

### C04 — scanner-net (AD)

| ID    | Catégorie STRIDE      | Menace                                                                          | Impact   | Probabilité |
|-------|-----------------------|---------------------------------------------------------------------------------|----------|-------------|
| M-019 | Spoofing              | Réponses LDAP forgées par un attaquant man-in-the-middle sur le réseau local    | CRITIQUE | FAIBLE      |
| M-020 | Tampering             | Injection de données AD falsifiées pour biaiser le score de sécurité            | CRITIQUE | FAIBLE      |
| M-021 | Repudiation           | Scan lancé sans référence d'autorisation enregistrée                            | ÉLEVÉ    | MOYENNE     |
| M-022 | Information Disclosure| Résultats bruts LDAP (comptes, hashes, ACL) accessibles sans chiffrement        | CRITIQUE | MOYENNE     |
| M-023 | Denial of Service     | Requêtes LDAP trop larges provoquant un impact sur le DC en production          | ÉLEVÉ    | MOYENNE     |
| M-024 | Elevation of Privilege| Utilisation des credentials AD pour des actions hors scope du scan              | CRITIQUE | FAIBLE      |

**Mitigations C04 :**
- M-019 ? LDAPS obligatoire (port 636), validation du certificat DC
- M-020 ? Validation des données LDAP reçues avant parsing (schéma strict)
- M-021 ? Référence d'autorisation obligatoire avant tout scan, loggée dans audit-logger
- M-022 ? Chiffrement AES-256-GCM des résultats bruts dès production
- M-023 ? Paging LDAP (PageSize configurable), throttling entre requêtes, mode dry-run
- M-024 ? Credentials AD en lecture seule obligatoire, scope LDAP restreint au minimum

---

### C05 — vuln-correlator

| ID    | Catégorie STRIDE      | Menace                                                                     | Impact  | Probabilité |
|-------|-----------------------|----------------------------------------------------------------------------|---------|-------------|
| M-025 | Tampering             | Manipulation du score AD pour masquer des vulnérabilités critiques         | CRITIQUE| FAIBLE      |
| M-026 | Information Disclosure| API NVD/MITRE interrogée avec des données AD identifiantes                 | MOYEN   | FAIBLE      |
| M-027 | Denial of Service     | Rate limiting NVD API dépassé, corrélation CVE impossible                  | MOYEN   | ÉLEVÉE      |

**Mitigations C05 :**
- M-025 ? Score calculé de manière déterministe, résultats hashés avant/après corrélation
- M-026 ? Seuls les identifiants techniques (CVE ID, type de finding) envoyés aux API externes
- M-027 ? Cache local des CVE (TTL 24h), fallback sur base offline en cas d'indisponibilité

---

### C06 — log-analyzer

| ID    | Catégorie STRIDE      | Menace                                                                        | Impact  | Probabilité |
|-------|-----------------------|-------------------------------------------------------------------------------|---------|-------------|
| M-028 | Tampering             | Injection de logs AD forgés pour déclencher de fausses alertes                | ÉLEVÉ   | FAIBLE      |
| M-029 | Information Disclosure| Logs AD contenant des événements Kerberos (4768/4769) exposés en clair        | CRITIQUE| MOYENNE     |
| M-030 | Denial of Service     | Fichier de log malformé provoquant une boucle infinie dans le parser nom      | ÉLEVÉ   | MOYENNE     |

**Mitigations C06 :**
- M-028 ? Validation du format et de la source avant ingestion (signature de la source)
- M-029 ? Pseudonymisation des SPN et des comptes dans les logs analysés
- M-030 ? Parser nom avec limite de taille d'entrée + timeout, fuzzing obligatoire (T06 cargo-fuzz)

---

### C07 — threat-intel

| ID    | Catégorie STRIDE      | Menace                                                                     | Impact  | Probabilité |
|-------|-----------------------|----------------------------------------------------------------------------|---------|-------------|
| M-031 | Spoofing              | Feed MISP/OTX compromis injectant de faux IOC                              | ÉLEVÉ   | FAIBLE      |
| M-032 | Information Disclosure| Données AD envoyées involontairement vers des services externes             | CRITIQUE| FAIBLE      |
| M-033 | Denial of Service     | Feed STIX/TAXII indisponible bloquant l'analyse threat intel               | MOYEN   | MOYENNE     |

**Mitigations C07 :**
- M-031 ? Vérification de la signature des feeds (TLP, source authentifiée)
- M-032 ? Aucune donnée AD transmise vers l'extérieur, IOC matching en local uniquement
- M-033 ? Cache local des IOC (TTL configurable), dégradation gracieuse si feed indisponible

---

### C08 — report-gen

| ID    | Catégorie STRIDE      | Menace                                                                     | Impact  | Probabilité |
|-------|-----------------------|----------------------------------------------------------------------------|---------|-------------|
| M-034 | Tampering             | Modification du rapport PDF après génération (faux résultats)              | CRITIQUE| FAIBLE      |
| M-035 | Information Disclosure| Rapport contenant des données AD sensibles transmis sans chiffrement       | CRITIQUE| MOYENNE     |
| M-036 | Repudiation           | Rapport non signé — contestable en cas d'audit ou de litige                | ÉLEVÉ   | FAIBLE      |

**Mitigations C08 :**
- M-034 ? Hash SHA-256 du rapport généré, signature Ed25519 embarquée dans le PDF
- M-035 ? Rapports chiffrés AES-256-GCM au repos, TLS 1.3 si transmis
- M-036 ? Signature Ed25519 obligatoire + horodatage qualifié sur chaque rapport

---

## Synthèse des menaces critiques

| ID    | Composant       | Menace résumée                          | Mitigation clé                        |
|-------|-----------------|-----------------------------------------|---------------------------------------|
| M-001 | api-gateway     | Token opérateur volé                    | JWT Ed25519 + rotation par session    |
| M-008 | core-engine     | Faux négatifs dans les résultats de scan| Hash des résultats bruts              |
| M-010 | core-engine     | Fuite credentials AD en mémoire         | Vault + zéroïsation après usage       |
| M-014 | audit-logger    | Altération des logs d'audit             | Hash chain append-only                |
| M-019 | scanner-net     | MITM sur LDAP                           | LDAPS obligatoire + validation cert   |
| M-022 | scanner-net     | Résultats LDAP bruts non chiffrés       | AES-256-GCM dès production            |
| M-024 | scanner-net     | Credentials AD utilisés hors scope      | Compte lecture seule + scope restreint|
| M-034 | report-gen      | Rapport falsifié post-génération        | Signature Ed25519 embarquée           |
