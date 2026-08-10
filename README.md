# my_skills

Collection personnelle de skills Python — des fichiers de référence chargés par
un agent pour cadrer la conception, l'écriture et la relecture de code.

## Contenu

| Skill | Objet |
|---|---|
| [`hexagonal_architecture_python_SKILL.md`](hexagonal_architecture_python_SKILL.md) | Architecture hexagonale (Ports & Adapters) : structure de projet, domaine, ports, use cases, adapters, injection, tests |
| [`clean_code_SKILL.md`](clean_code_SKILL.md) | Principes Clean Code appliqués à Python : nommage, fonctions, classes, qualité des tests |
| [`mypy_SKILL.md`](mypy_SKILL.md) | Typage statique en mode strict : configuration, annotations, cas difficiles |
| [`pytest_SKILL.md`](pytest_SKILL.md) | Écriture et organisation des tests : fixtures, paramétrage, structure de suite |

## Conventions communes

- **Gestion de paquets** : `uv`
- **Typage** : `mypy` en mode strict, contrats via `typing.Protocol`
- **Tests** : `pytest`
- **Nommage des fichiers** : `{nom}_SKILL.md`, fichiers plats à la racine
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
référence par notion, puis des règles en liste. Ce qu'un agent doit retenir,
ce sont les gabarits et les arbitrages — pas les justifications.

## Méthode de construction

Les skills sont écrits section par section, avec validation explicite à chaque
étape, puis relus en deux passes : cohérence globale, et **validation par
exécution** — les extraits sont montés en projet réel et doivent passer
`mypy --strict` et `pytest` avant publication.
