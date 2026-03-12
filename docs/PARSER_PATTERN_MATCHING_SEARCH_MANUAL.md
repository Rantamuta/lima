# LIMA Parser Pattern Matching Manual (Search Verb Focus)

## Purpose

Explain, technically and concretely, how LIMA-style rule patterns match word/token input for verbs, using `search` as the worked example.

This manual is implementation-oriented and focuses on what can be directly observed in the current codebase.

---

## 1) Quick answer

For `search`, matching is driven by three things:

1. A verb declares allowed grammar patterns via `add_rules(...)`.
2. The parser classifies typed slots (`OBJ`, `STR`, etc.) and literal connector words (`for`, `in`, `with`).
3. On a successful rule match, it dispatches to a `do_search_<rule-shape>` handler with typed arguments in rule order.

The declared patterns are:

- `"OBJ"`
- `"OBJ with OBJ"`
- `"for STR"`
- `"for STR in OBJ"`
- `"OBJ for STR"`
- `"OBJ with OBJ for STR"`
- `"OBJ for STR with OBJ"`
- `"for STR in OBJ with OBJ"`
- `""` (bare `search`)

---

## 2) Where patterns are declared

`lib/cmds/verbs/search.c` declares these rules in `create()` with one `add_rules(...)` call. The parser registration itself is handled by `VERB_OB` (`add_rules` -> `parse_add_rule`).

- `search` rule declaration: `lib/cmds/verbs/search.c`.
- registration path: `lib/std/verb_ob.c` (`add_rules` uses `parse_add_rule`).

---

## 3) Token-pattern model

### Observed behavior

Rules are mixed **literal words** and **typed placeholders**:

- Literal words: `for`, `in`, `with`
- Typed placeholders:
  - `OBJ` = object slot
  - `STR` = free string slot

The parser treats literal words as anchors and typed slots as capture points.

### Inferred behavior

This works as a grammar matcher over token sequences. For example:

- `search for coins in chest`
  - literal `for`
  - capture `STR=coins`
  - literal `in`
  - capture `OBJ=chest`

---

## 4) Exact rule -> handler mapping for `search`

The `search` verb file defines one `do_...` function per rule shape. This lets you see exactly how matched tokens are forwarded.

| Rule | Handler | Signature |
|---|---|---|
| `"OBJ"` | `do_search_obj` | `(object ob)` |
| `"OBJ with OBJ"` | `do_search_obj_with_obj` | `(object ob1, object ob2)` |
| `"for STR"` | `do_search_for_str` | `(string str)` |
| `"for STR in OBJ"` | `do_search_for_str_in_obj` | `(string str, object ob)` |
| `"OBJ for STR"` | `do_search_obj_for_str` | `(object ob, string str)` |
| `"OBJ with OBJ for STR"` | `do_search_obj_with_obj_for_str` | `(object ob, object with, string str)` |
| `"OBJ for STR with OBJ"` | `do_search_obj_for_str_with_obj` | `(object ob1, string str, object ob2)` |
| `"for STR in OBJ with OBJ"` | `do_search_for_str_in_obj_with_obj` | `(string str, object ob1, object ob2)` |
| `""` | `do_search` | `()` |

---

## 5) Why these similar forms do not collide

The key distinction is that literals are part of the pattern:

- `OBJ with OBJ for STR`
- `OBJ for STR with OBJ`

Both contain `OBJ`, `STR`, `OBJ`, but the literal connectors are ordered differently (`with ... for` vs `for ... with`), so they map to different handlers and argument order.

This is the core mechanism that lets many near-duplicate natural-language forms coexist safely.

---

## 6) Validation phase after pattern match

Pattern match is only phase 1. After a candidate rule is chosen, parser calls object-level `direct_*` / `indirect_*` checks corresponding to that rule shape.

For `search`, these checks exist in `lib/std/object/vsupport.c`, for example:

- `direct_search_obj`
- `direct_search_for_str`
- `direct_search_obj_for_str`
- `indirect_search_obj_with_obj`
- `indirect_search_for_str_in_obj_with_obj`
- etc.

These checks can:

- allow (`1`),
- deny (`0`), or
- return explicit error text.

Example: several `...with_obj...` checks enforce that the tool object is held by the actor and return a concrete failure string if not.

---

## 7) Execution phase for `search`

The `do_search_*` handlers mostly normalize to one API call:

- `ob->do_search(withObj, searchString)` or
- `environment(this_body())->do_search(...)`.

So many syntactic variants converge into the same semantic operation while preserving different parse routes.

---

## 8) Worked examples (token flow)

### Example A: `search chest`

- candidate rule: `OBJ`
- parser binds `OBJ=chest`
- check function path: `direct_search_obj(chest)`
- execute: `do_search_obj(chest)` -> `chest->do_search(0)`

### Example B: `search chest with crowbar`

- candidate rule: `OBJ with OBJ`
- binds: `ob1=chest`, `ob2=crowbar`
- checks include `indirect_search_obj_with_obj(...)` guard for held tool
- execute: `do_search_obj_with_obj(chest, crowbar)`

### Example C: `search for coins in chest`

- candidate rule: `for STR in OBJ`
- binds: `str="coins"`, `ob=chest`
- execute: `do_search_for_str_in_obj("coins", chest)`

### Example D: `search chest for coins with crowbar`

- candidate rule: `OBJ for STR with OBJ`
- binds: `ob1=chest`, `str="coins"`, `ob2=crowbar`
- execute: `do_search_obj_for_str_with_obj(chest, "coins", crowbar)`

---

## 9) How this generalizes to other verbs

The same parser contract appears in other verbs:

1. declare rule strings in `add_rules(...)`,
2. implement `do_<verb>_<shape>` handlers,
3. rely on `direct_/indirect_` rule checks on involved objects.

`VERB_OB` is the common integration point into parser registration (`parse_add_rule`).

---

## 10) Gotchas and porting advice

### Gotcha 1: similar patterns with swapped connectors

`OBJ with OBJ for STR` vs `OBJ for STR with OBJ` are semantically close but distinct parse signatures.

**Port advice:** preserve both if your UX expects both phrasings.

### Gotcha 2: check-function naming explosion

Each grammar shape can imply multiple `direct_`/`indirect_` checks.

**Port advice:** generate these handlers mechanically from rule specs where possible.

### Gotcha 3: tool possession checks in indirect phase

Failure ownership often lives in indirect checks (e.g., "with" tool not held).

**Port advice:** model validation after match, before execution, with role-aware ownership checks.

### Gotcha 4: parser precedence details are driver-level

Exact internal tie-break precedence when multiple patterns could theoretically fit is implemented in parser efuns not shown in this repo.

**Port advice:** enforce unambiguous grammars and include disambiguation tests for near-overlapping patterns.

---

## 11) Ambiguities requiring maintainer confirmation

1. Exact precedence/tie-break policy between multiple candidate rules with equivalent typed fit (driver parser internals).
2. Full formal semantics of token classes beyond observed usage (`OBJ`, `OBS`, `WRD`, `STR`) in this repository.
3. Whether any verbs depend on parser behavior not represented by explicit `add_rules(...)` declarations.

---

## 12) Source anchors

- `search` rules + handlers: `lib/cmds/verbs/search.c`
- rule registration wrapper: `lib/std/verb_ob.c`
- `search` check functions + default behavior: `lib/std/object/vsupport.c`

