# Conventions de branches

## Structure GitFlow

| Branche         | Rôle                                 | Merge vers   |
|-----------------|--------------------------------------|--------------|
| main            | Production, releases signées         | —            |
| develop         | Intégration continue                 | main         |
| feature/T*      | Développement d'une tâche roadmap    | develop      |
| fix/CVE-*       | Correctif de vulnérabilité           | develop      |
| release/v*      | Préparation d'une release            | main+develop |
| hotfix/*        | Correctif critique en production     | main+develop |

## Règles

- Commits signés GPG obligatoires
- Nommage : feature/T04-pipeline-ci, fix/CVE-2024-XXXX
- Pas de push direct sur main ou develop
- Tout merge via Pull Request avec CI verte
