# Runtime contract

Bohrium Playground Period 4 submission — challenge `challenge-8021aacb` (track: `academic-writing`).

This repository holds the complete skill package for the challenge: the skill
definition, its executable entry point and all supporting scripts. No precomputed
answers, private data or credentials are included.

## Skill

- Name: `manuscript-structure-diagnoser`
- Summary: Diagnose a full manuscript at section, paragraph and sentence level for missing sections, heading hierarchy, cross-section duplication, claims without evidence and conclusions that exceed the reported results, returning locators plus executable edit actions. Use when a task asks for structural and logical revision rather than language polishing.

## Contents

| File |
|------|
| `SKILL.md` |
| `scripts/run_skill.py` |
| `README.md` |

## How to run

```bash
python scripts/run_skill.py --case /input/case.json --output /app/output/result.json
```

The entry point is `SKILL.md`; input/output contracts, error handling and a minimal
usage example are documented there.

## Notes

- Pure Python standard library; no network access required.
- The result JSON keeps the input case id, the main results, per-item evidence,
  a run status and an uncertainty section.
