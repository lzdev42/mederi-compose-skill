# mederi-compose-skill

Customized Compose Multiplatform (Android, iOS, Desktop, `wasmJs`) agent skill, adapted from [`Meet-Miyani/compose-skill`](https://github.com/Meet-Miyani/compose-skill).

## Key Customizations

- Auto-activated for Compose Multiplatform UI, layout, state, and data-binding tasks (`name: mederi-compose-skill`).
- Full `commonMain` + `wasmJs` target focus (removed Android-only Hilt / `dependency-injection.md`; updated Room/DataStore/Paging/Navigation/Resources/Coroutines for `wasmJs`).
- Enforces Data-Layer SSOT + reactive ViewModel derivation, transparent container hierarchy, semantic theme inheritance, constraint-based centering, one-file-per-component boundaries, typed models, and honest `-` error display.

## Installation

```bash
git clone https://github.com/lzdev42/mederi-compose-skill.git ~/.gemini/config/skills/mederi-compose-skill
```

## Validation

```bash
./scripts/validate.sh
```
