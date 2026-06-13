# Matrice des risques — Camelot-AD
## Date : Juin 2025 | Version : 1.0
## Référence : THREAT-MODEL.md + ADVERSARY-PROFILE.md

---

## Méthode

Risque brut = Probabilité (1-3) × Impact (1-3)
Risque résiduel = risque après application des mitigations
Seuil d'acceptabilité : risque résiduel = 2 (Moyen) pour merger en production

---

## Risques critiques (traitement immédiat requis)

| ID    | Menace                                  | Prob. | Impact | Risque brut | Mitigation principale                  | Risque résiduel | Statut     |
|-------|-----------------------------------------|-------|--------|-------------|----------------------------------------|-----------------|------------|
| M-010 | Fuite credentials AD en mémoire         | 2     | 3      | 6           | Vault + zéroïsation mémoire            | 2               | À implémenter Phase 1 |
| M-014 | Altération logs audit                   | 2     | 3      | 6           | Hash chain append-only                 | 1               | À implémenter Phase 1 |
| M-019 | MITM sur LDAP                           | 1     | 3      | 3           | LDAPS obligatoire + validation cert    | 1               | À implémenter Phase 1 |
| M-022 | Résultats LDAP bruts non chiffrés       | 2     | 3      | 6           | AES-256-GCM dès production             | 1               | À implémenter Phase 1 |
| M-024 | Credentials AD utilisés hors scope      | 1     | 3      | 3           | Compte lecture seule + scope restreint | 1               | À implémenter Phase 1 |
| M-034 | Rapport falsifié post-génération        | 1     | 3      | 3           | Signature Ed25519 embarquée            | 1               | À implémenter Phase 2 |

---

## Risques élevés (traitement requis avant release POC)

| ID    | Menace                                  | Prob. | Impact | Risque brut | Mitigation principale                  | Risque résiduel | Statut     |
|-------|-----------------------------------------|-------|--------|-------------|----------------------------------------|-----------------|------------|
| M-001 | Token opérateur volé                    | 2     | 3      | 6           | JWT Ed25519 + rotation par session     | 2               | À implémenter Phase 1 |
| M-002 | Paramètres requête hors périmètre       | 2     | 2      | 4           | Validation stricte + whitelist cibles  | 1               | À implémenter Phase 1 |
| M-005 | Flood API gateway                       | 2     | 2      | 4           | Rate limiting par IP et par token      | 1               | À implémenter Phase 1 |
| M-006 | Bypass auth endpoints admin             | 1     | 3      | 3           | Auth obligatoire tous endpoints        | 1               | À implémenter Phase 1 |
| M-008 | Faux négatifs résultats scan            | 1     | 3      | 3           | Hash résultats bruts avant scoring     | 1               | À implémenter Phase 1 |
| M-012 | core-engine droits excessifs sur hôte   | 2     | 3      | 6           | seccomp + namespaces + least-privilege | 2               | À implémenter Phase 0 |
| M-021 | Scan sans référence d'autorisation      | 2     | 2      | 4           | Autorisation obligatoire avant scan    | 1               | À implémenter Phase 1 |
| M-023 | Requêtes LDAP impactant le DC           | 2     | 2      | 4           | Paging LDAP + throttling + dry-run     | 1               | À implémenter Phase 1 |
| M-029 | Événements Kerberos exposés en clair    | 2     | 3      | 6           | Pseudonymisation SPN et comptes        | 2               | À implémenter Phase 2 |
| M-035 | Rapport transmis sans chiffrement       | 2     | 3      | 6           | AES-256-GCM + TLS 1.3                  | 1               | À implémenter Phase 2 |

---

## Risques moyens (backlog priorisé)

| ID    | Menace                                  | Prob. | Impact | Risque brut | Mitigation principale                  | Risque résiduel | Statut     |
|-------|-----------------------------------------|-------|--------|-------------|----------------------------------------|-----------------|------------|
| M-003 | Logs non intègres — répudiation         | 1     | 2      | 2           | HMAC-SHA256 sur chaque entrée          | 1               | À implémenter Phase 1 |
| M-004 | Stack trace exposée en réponse API      | 3     | 2      | 6           | Erreurs génériques côté client         | 1               | À implémenter Phase 1 |
| M-009 | Session non tracée dans les logs        | 2     | 2      | 4           | session_id + operator_id obligatoires  | 1               | À implémenter Phase 1 |
| M-011 | Boucle infinie sur domaine AD complexe  | 2     | 2      | 4           | Timeout + limite profondeur récursion  | 1               | À implémenter Phase 1 |
| M-017 | Flood logs saturant le stockage         | 1     | 2      | 2           | Quota log par session + alertes        | 1               | À implémenter Phase 2 |
| M-025 | Score AD manipulé                       | 1     | 3      | 3           | Hash avant/après corrélation           | 1               | À implémenter Phase 1 |
| M-027 | NVD API indisponible                    | 3     | 2      | 6           | Cache local CVE TTL 24h                | 2               | À implémenter Phase 1 |
| M-030 | Parser nom boucle infinie               | 2     | 2      | 4           | Limite taille + timeout + fuzzing      | 1               | À implémenter Phase 2 |
| M-031 | Feed MISP/OTX compromis                 | 1     | 2      | 2           | Vérification signature feeds           | 1               | À implémenter Phase 2 |
| M-033 | Feed STIX/TAXII indisponible            | 2     | 2      | 4           | Cache local IOC + dégradation gracieuse| 1               | À implémenter Phase 2 |
| M-036 | Rapport non signé                       | 1     | 2      | 2           | Signature Ed25519 obligatoire          | 1               | À implémenter Phase 2 |

---

## Risques acceptés (surveillance uniquement)

| ID    | Menace                                  | Justification                                              |
|-------|-----------------------------------------|------------------------------------------------------------|
| M-007 | Module compromis usurpant core-engine   | mTLS réduit le risque résiduel à négligeable en POC local  |
| M-013 | Injection faux événements audit         | Canal dédié mTLS + seul core-engine autorisé en écriture   |
| M-015 | Logs présents mais non signés           | Couvert par M-014 (hash chain) + M-003 (HMAC)              |
| M-018 | Accès direct sled DB hors audit-logger  | Droits OS restreints au process audit-logger uniquement     |
| M-020 | Données LDAP falsifiées                 | Validation schéma stricte + LDAPS réduit à négligeable      |
| M-026 | Données AD vers API externes            | IOC matching local uniquement — aucune donnée AD transmise  |
| M-028 | Logs AD forgés                          | Validation source + signature avant ingestion              |
| M-032 | Données AD vers threat-intel externe    | Couvert par M-026                                          |

---

## Indicateurs de suivi (KRI — Key Risk Indicators)

| Indicateur                                    | Seuil d'alerte         | Fréquence de contrôle |
|-----------------------------------------------|------------------------|-----------------------|
| Nombre de CVE CVSS = 7.0 non traitées         | > 0                    | À chaque commit       |
| Nombre de scans sans référence d'autorisation | > 0                    | Temps réel            |
| Taille des logs d'audit par session            | > 500 Mo               | Quotidien             |
| Échecs d'authentification consécutifs          | > 5 en moins de 60s    | Temps réel            |
| Intégrité hash chain audit-logger              | Toute rupture          | À chaque lecture      |
| Dépendances sans checksum vérifié              | > 0                    | À chaque build        |

---

## Risques résiduels > 2 nécessitant une décision

| ID    | Menace                          | Risque résiduel | Décision requise                                      |
|-------|---------------------------------|-----------------|-------------------------------------------------------|
| M-001 | Token opérateur volé            | 2               | Acceptable en POC local — réévaluer à l'ouverture distante |
| M-012 | core-engine droits excessifs    | 2               | Implémentation seccomp obligatoire avant Phase 1      |
| M-029 | Événements Kerberos en clair    | 2               | Pseudonymisation obligatoire avant tout log Kerberos  |

