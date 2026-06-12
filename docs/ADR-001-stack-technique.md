# ADR-001 — Choix de la stack technique

## Statut
Accepté — Semaine 1

## Contexte
Choix du langage et des dépendances principales pour le POC.

## Décision
- Langage : Rust édition 2021 (memory-safe, pas de CVE mémoire)
- Runtime async : tokio (la référence, activement maintenue)
- API : axum (construit sur tokio, audité)
- Crypto : ring + ed25519-dalek (auditées, pas d'implémentation maison)
- Logging : tracing (structuré JSON, compatible OpenTelemetry)

## Critères appliqués (CDC §2.2)
- ✅ Activité de commit < 12 mois
- ✅ 0 CVE CVSS ≥ 7.0 (cargo audit)
- ✅ Licences MIT / Apache-2.0
- ✅ Checksums vérifiés via Cargo.lock

## Alternatives rejetées
- Go : écosystème suffisant mais Rust préféré pour la sureté mémoire
- async-std : moins adopté que tokio, écosystème plus restreint
