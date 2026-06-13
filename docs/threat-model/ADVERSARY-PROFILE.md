# Profils d'adversaires — Camelot-AD
## Date : Juin 2025 | Version : 1.0

---

## Contexte

Camelot-AD est un outil de scan AD offensif produisant une cartographie complète
du domaine (comptes, ACL, délégations, GPO, trusts, Kerberos). Les résultats
constituent une cible de valeur extrême pour tout attaquant cherchant à compromettre
un domaine Active Directory.

---

## AP-01 — Opérateur malveillant interne

| Attribut      | Valeur                                                              |
|---------------|---------------------------------------------------------------------|
| Profil        | Pentest interne autorisé, insider malveillant                       |
| Motivation    | Exfiltration de données AD, élargissement de scope non autorisé     |
| Capacités     | Accès légitime à l'outil, connaissance de l'environnement AD        |
| Vecteurs      | Abus des credentials fournis, scan hors périmètre, export des logs  |
| Impact maximal| Cartographie AD complète exfiltrée, comptes privilégiés exposés     |

**Menaces STRIDE associées :** M-001, M-002, M-021, M-024

**Contrôles spécifiques :**
- Référence d'autorisation obligatoire avant chaque scan (CDC §7.1)
- Whitelist de cibles stricte — refus automatique hors périmètre
- Chaque action loggée avec operator_id non répudiable
- Résultats chiffrés, inaccessibles sans clé de session

---

## AP-02 — Attaquant compromettant la machine hôte

| Attribut      | Valeur                                                                    |
|---------------|---------------------------------------------------------------------------|
| Profil        | Attaquant externe ayant compromis le poste de l'opérateur                 |
| Motivation    | Récupérer la cartographie AD sans effort d'énumération                    |
| Capacités     | Accès OS sur la machine hôte, dump mémoire, accès fichiers                |
| Vecteurs      | Vol de credentials AD en mémoire, accès aux résultats de scan au repos    |
| Impact maximal| Cartographie AD complète + credentials Domain Admin potentiellement volés |

**Menaces STRIDE associées :** M-010, M-022, M-035

**Contrôles spécifiques :**
- Credentials AD jamais persistés sur disque (Vault uniquement)
- Zéroïsation mémoire des credentials après usage
- Résultats de scan chiffrés AES-256-GCM dès leur production
- Conteneurisation rootless — surface d'accès OS réduite

---

## AP-03 — Insider passif (accès non autorisé aux résultats)

| Attribut      | Valeur                                                                 |
|---------------|------------------------------------------------------------------------|
| Profil        | Collègue avec accès physique ou réseau à la machine Camelot-AD         |
| Motivation    | Curiosité, avantage informationnel interne, pas d'intention hostile     |
| Capacités     | Accès réseau local, potentiellement accès OS si machine non verrouillée |
| Vecteurs      | Consultation des rapports non protégés, accès API sans auth            |
| Impact maximal| Accès à la cartographie AD complète sans autorisation                  |

**Menaces STRIDE associées :** M-004, M-006, M-016, M-029

**Contrôles spécifiques :**
- Authentification obligatoire sur tous les endpoints API sans exception
- Rapports chiffrés au repos, accès par token de session uniquement
- Logs pseudonymisés — pas de données AD en clair dans les fichiers de log
- Avertissement légal au démarrage avec acquittement obligatoire (CDC §7.1)

---

## AP-04 — Attaquant réseau local (MITM)

| Attribut      | Valeur                                                                      |
|---------------|-----------------------------------------------------------------------------|
| Profil        | Attaquant positionné sur le même segment réseau que le DC                   |
| Motivation    | Intercepter les requêtes LDAP/Kerberos pour cartographier l'AD passivement  |
| Capacités     | ARP spoofing, capture de trafic réseau, outils type Wireshark/Responder     |
| Vecteurs      | Interception des requêtes LDAP en clair, capture des tickets Kerberos       |
| Impact maximal| Cartographie AD passive sans toucher à Camelot-AD, tickets Kerberos capturés|

**Menaces STRIDE associées :** M-019, M-020

**Contrôles spécifiques :**
- LDAPS obligatoire (port 636) — pas de LDAP en clair accepté
- Validation du certificat du DC avant toute requête
- Channel binding LDAP activé si supporté par le DC
- Requêtes Kerberos via canal chiffré uniquement

---

## Matrice de priorité

| Profil | Probabilité POC | Impact | Priorité de mitigation |
|--------|-----------------|--------|------------------------|
| AP-01  | ÉLEVÉE          | CRITIQUE | P1 — immédiat        |
| AP-02  | MOYENNE         | CRITIQUE | P1 — immédiat        |
| AP-03  | ÉLEVÉE          | ÉLEVÉ    | P2 — Phase 0/1       |
| AP-04  | FAIBLE          | ÉLEVÉ    | P2 — Phase 0/1       |

---

## Mapping MITRE ATT&CK

| Profil | Technique ATT&CK                        | ID          |
|--------|-----------------------------------------|-------------|
| AP-01  | Valid Accounts                          | T1078       |
| AP-01  | Data from Local System                  | T1005       |
| AP-02  | OS Credential Dumping                   | T1003       |
| AP-02  | Data Encrypted for Impact               | T1486       |
| AP-03  | Exploitation of Remote Services         | T1210       |
| AP-04  | Network Sniffing                        | T1040       |
| AP-04  | Adversary-in-the-Middle                 | T1557       |
