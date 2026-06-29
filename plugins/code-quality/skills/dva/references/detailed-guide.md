# DVA — Guide Detaille

## Outils par Categorie

### Tests Automatises
| Outil | Usage |
|-------|-------|
| pytest / go test | Tests unitaires et d'integration |
| Selenium / Playwright | Tests end-to-end navigateur |
| k6 / Locust | Tests de charge et performance |
| Fuzzing (go-fuzz, AFL) | Decouverte d'inputs inattendus |

### Analyse Statique
| Outil | Usage |
|-------|-------|
| golangci-lint / ESLint | Linting et anti-patterns |
| SonarQube | Vulnerabilites et code smells |
| Semgrep | Patterns de securite custom |
| gosec | Securite specifique Go |

### Chaos Engineering
| Outil | Usage |
|-------|-------|
| Chaos Monkey | Pannes aleatoires en production |
| Gremlin | Simulation de pannes controlees |
| toxiproxy | Simulation de latence/erreurs reseau |

### Monitoring
| Outil | Usage |
|-------|-------|
| Prometheus + Grafana | Metriques temps reel |
| ELK Stack | Centralisation de logs |
| Sentry | Tracking d'erreurs |

## Template Complet de Fiche de Feedback

```markdown
## DVA Feedback — [Nom du composant]

**Date** : YYYY-MM-DD
**Ingenieur** : [nom]
**Antagoniste** : [nom ou "Claude"]

### Perimetre
- Probleme : ...
- Contraintes : ...
- Criteres de succes : ...

### Failles Identifiees

| # | Categorie | Faille | Impact | Preuve | Fix Propose | Severite | Statut |
|---|-----------|--------|--------|--------|-------------|----------|--------|
| 1 | Securite | ... | ... | ... | ... | Critique | Corrige |
| 2 | Performance | ... | ... | ... | ... | Majeur | En cours |
| 3 | Fonctionnel | ... | ... | ... | ... | Mineur | A faire |

### Forces Validees
- [ce qui est bien concu et pourquoi]

### Decision Finale
- [ ] PASS
- [ ] PASS AVEC RESERVES
- [ ] FAIL — blockers restants : ...
```

## Exemples Concrets

### Exemple 1 : Review d'un endpoint API

**Ingenieur** : "Nouvel endpoint POST /api/products qui accepte un JSON, valide les champs, et insere en base."

**Antagoniste** :
| # | Categorie | Faille | Severite |
|---|-----------|--------|----------|
| 1 | Securite | Pas de rate limiting — vulnerable au brute force | Critique |
| 2 | Fonctionnel | Body > 10MB non rejete — OOM possible | Majeur |
| 3 | Performance | INSERT sans batch — N+1 si import bulk | Majeur |
| 4 | Fiabilite | Pas de timeout sur la connexion DB | Mineur |

### Exemple 2 : Review d'architecture

**Ingenieur** : "Migration du monolithe vers 3 microservices avec communication par events."

**Antagoniste** :
| # | Categorie | Faille | Severite |
|---|-----------|--------|----------|
| 1 | Fiabilite | Pas de dead letter queue — events perdus silencieusement | Critique |
| 2 | Performance | Serialisation JSON entre services — overhead sur hot path | Majeur |
| 3 | Maintenabilite | Schema des events non versionne — breaking changes invisibles | Majeur |

## Ressources

### Livres
- *The DevOps Handbook* (Gene Kim et al.) — Integration continue et feedback loops
- *Site Reliability Engineering* (Google) — Robustesse des systemes
- *Secure by Design* (Dan Bergh Johnsson) — Securite par construction

### Methodologies
- Chaos Engineering : https://principlesofchaos.org/
- Red Teaming : adversarial testing structure
- Threat Modeling (STRIDE) : analyse systematique des menaces

### Outils de Securite
- OWASP ZAP : tests d'intrusion automatises
- Trivy : scan de vulnerabilites containers/deps
- Snyk : analyse de dependances
