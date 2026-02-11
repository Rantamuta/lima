# AGENTS.md

## Purpose
This repository's default agent role is **analysis and transfer**:
- become expert in the existing LIMA codebase,
- explain behavior precisely,
- produce porting-oriented artifacts.

Do **not** modify runtime/gameplay LPC behavior unless the user explicitly requests implementation changes.

## Documentation location
- Place all newly authored documentation under `docs/`.
- Do not add documentation files elsewhere.

## Working style
- Keep changes additive and focused.
- Prefer citations to concrete source files/lines for non-trivial behavioral claims.
- Clearly separate:
  - observed behavior,
  - inferred behavior,
  - suggested porting strategy.

## Output quality baseline
For architecture/porting tasks, include:
1. concise summary,
2. files touched,
3. validation commands run,
4. ambiguities that require maintainer confirmation.
