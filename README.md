# my_skills

Collection personnelle de skills Python — des fichiers de référence chargés par
un agent pour cadrer la conception, l'écriture et la relecture de code.

## Contenu

| Skill | Objet |
|---|---|
| [`hexagonal-architecture-python`](hexagonal-architecture-python/SKILL.md) | Architecture hexagonale (Ports & Adapters) : structure de projet, domaine, ports, use cases, adapters, injection, tests |
| [`clean-code-python`](clean-code-python/SKILL.md) | Principes Clean Code appliqués à Python : nommage, fonctions, classes, qualité des tests |
| [`mypy-python`](mypy-python/SKILL.md) | Typage statique en mode strict : annotations, configuration, discipline des `type: ignore`, anti-patterns ; références : types avancés, adoption progressive, FastAPI |
| [`pytest-python`](pytest-python/SKILL.md) | Écriture et organisation des tests : structure, markers, paramétrage, fixtures, doublures ; références : fixtures, mocking, CLI et config, anti-patterns |

## Conventions communes

- **Gestion de paquets** : `uv`
- **Typage** : `mypy` en mode strict, contrats via `typing.Protocol`
- **Tests** : `pytest`
- **Structure** : un dossier par skill, `{nom-du-skill}/SKILL.md` (+ éventuel
  `references/`) ; le nom du dossier est identique au champ `name` du frontmatter
- **Registre** : anglais, dense et opérationnel — du code de référence et des
  règles de décision, pas d'argumentaire

## Format d'un skill

Chaque fichier commence par un frontmatter YAML :

```yaml
---
name: nom-du-skill
description: Ce que le skill couvre et quand l'utiliser.
---
```

La `description` détermine le déclenchement. Ces skills étant appelés
explicitement par un agent dédié, elles sont **resserrées** : elles décrivent le
périmètre plutôt que de ratisser large sur les symptômes.

Le corps vise ~500 lignes maximum : sections courtes, un bloc de code de
référence par notion, puis des règles en liste. Au-delà, le détail part dans
`references/*.md`, listé en tête du `SKILL.md` et chargé seulement au besoin. Ce qu'un agent doit retenir,
ce sont les gabarits et les arbitrages — pas les justifications.

## Installation

Copier (ou lier) les dossiers dans le répertoire de skills de Claude Code :

```bash
cp -r */ ~/.claude/skills/        # skills personnels
# ou, par projet : .claude/skills/
```

## Méthode de construction

Les skills sont écrits section par section, avec validation explicite à chaque
étape, puis relus en deux passes : cohérence globale, et **validation par
exécution** — les extraits sont montés en projet réel et doivent passer
`mypy --strict` et `pytest` avant publication.
