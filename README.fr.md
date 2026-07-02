# kirozen-skills

*[English version](README.md).*

Marketplace personnel de plugins [Claude Code](https://claude.ai/code). Tout est en Markdown + config — aucune étape de build.

Le marketplace regroupe trois plugins. Les skills sont auto-découvertes après installation et invoquées **avec namespace** : `/<plugin>:<skill>`.

## Plugins

### `code-quality`

Skills de code propre & de revue, basées sur *Clean Code* / *Clean Architecture* de Robert C. Martin.

| Skill | Ce qu'elle fait |
|-------|-----------------|
| `clean-code` | Garder le code simple et lisible pendant l'écriture/refactoring (nommage, petites fonctions, SOLID). |
| `clean-architect` | Structurer couches/modules/frontières et juger le sens des dépendances. |
| `code-reviewer` | Revoir un diff/PR concret pour correction, sécurité, performance et qualité avant merge. |
| `dva` | Stress-tester de façon adversariale une solution/design *proposé* (red-team Ingénieur/Antagoniste) avant de s'y engager. |
| `debt-analyzer` | Transformer la dette technique en backlog priorisé et étayé (impact ÷ effort) — *quoi corriger en premier*. |
| `techdebt` | Scan exhaustif, orchestré par agents, qui *énumère* duplication concrète & code mort avec preuves fichier/ligne. |
| `debugger` | Trouver et *prouver* la cause racine d'un bug avant de proposer un correctif. |

### `sdd`

Développement piloté par la spec, adossé à SQLite. La spec vit dans une base de données ; `SPEC.md` en est une vue générée en lecture seule. La boucle complète est pilotée par la CLI `sdd`. Pour la référence des commandes CLI et le modèle de spec (blocs durables vs éphémères, citations), voir le dépôt moteur [`Kirozen/sdd`](https://github.com/Kirozen/sdd).

| Skill | Ce qu'elle fait |
|-------|-----------------|
| `sdd-grill` | Affûter une idée floue en goal + contraintes d'une feature. |
| `sdd-spec` | Seul mutateur de la spec — invariants, interfaces, tâches. |
| `sdd-research` | Rassembler du savoir externe dans le journal de recherche durable, chaque trouvaille citée. |
| `sdd-review` | Revue senior adversariale de la spec avant tout code ; finit sur une gate go/no-go. |
| `sdd-build` | Plan-puis-exécution contre les tâches de la spec ; auto-invoque backprop en cas d'échec. |
| `sdd-backprop` | Bug → spec : tracer la cause racine et persister un nouvel invariant pour capter la récurrence. |
| `sdd-deepen` | Passe optionnelle d'amélioration du design — réduire les interfaces, cacher les décisions. |
| `sdd-drift` | Détecteur en lecture seule de divergence code↔spec ; rapporte les violations par sévérité, n'écrit rien. Distinct de la commande CLI `sdd check` (SPEC.md == spec.db). |

Le binaire de la CLI `sdd` n'est **pas** vendorisé ici : un hook `SessionStart` le provisionne automatiquement depuis les [releases GitHub de `Kirozen/sdd`](https://github.com/Kirozen/sdd) (vérifié par SHA256). Le tag de release du binaire est épinglé indépendamment de la version du plugin dans `plugins/sdd/scripts/binary-version`, pour que les mises à jour de skills seules puissent faire évoluer le plugin sans exiger une release de binaire correspondante.

### `gopls-daemon`

Configure le serveur de langage Go (`gopls`) en mode daemon partagé pour l'intégration LSP de Claude Code. Pure config `.lsp.json` — aucune skill.

## Utilisation

Les skills se déclenchent de **deux façons** :

- **Automatiquement** — Claude lit la `description` de chaque skill et invoque la bonne quand votre demande correspond. Vous ne la nommez pas ; décrivez juste la tâche (« relis mon diff », « scan the codebase for duplication », « challenge this design »). Écrire les déclencheurs dans la description est le mécanisme *primaire*.
- **Explicitement** — l'invoquer par son nom. Une fois installées, les skills sont **namespacées** : `/code-quality:code-reviewer`, `/sdd:sdd-spec`. En dogfooding dans ce repo (symlinks), elles sont sans namespace : `/code-reviewer`.

### Choisir entre skills qui se recouvrent

Deux paires dans `code-quality` sont volontairement proches — choisissez par *intention* :

| Vous voulez… | Utilisez | Pas |
|--------------|----------|-----|
| Revoir des changements concrets (un diff/PR) avant merge | `code-reviewer` | `dva` |
| Éprouver une approche/architecture *proposée* avant de l'écrire | `dva` | `code-reviewer` |
| Décider *quelle dette corriger en premier* (backlog priorisé) | `debt-analyzer` | `techdebt` |
| *Énumérer* la duplication / le code mort concrets du repo | `techdebt` | `debt-analyzer` |

Un flux typique les enchaîne : `clean-architect` (design) → `dva` (stress-test du design) → `clean-code` (écriture) → `code-reviewer` (revue du diff) → `debugger` (si ça casse). Pour les bilans de santé, `techdebt` énumère, puis `debt-analyzer` priorise les trouvailles.

## Installation

```bash
/plugin marketplace add Kirozen/skills      # ou : /plugin marketplace add .  (clone local)
/plugin install code-quality@kirozen-skills
/plugin install sdd@kirozen-skills
/plugin install gopls-daemon@kirozen-skills
```

Lancez `/reload-plugins` après installation.

## Développement local

```bash
claude plugin validate .                      # valide les manifestes marketplace + plugins
claude --plugin-dir ./plugins/code-quality    # charge un plugin sans l'installer
```

Pour le dogfooding dans ce repo, chaque skill est symlinkée dans `.claude/skills/`, elle se charge donc comme une skill de projet ordinaire (`/<skill>`, sans namespace) sans installer le plugin.

Les conventions, la structure et les règles pour ajouter des skills/plugins vivent dans [`CLAUDE.md`](./CLAUDE.md).
