# Agent Map: Command & Verb System (LIMA)

This document is a practical map for humans/agents who want to reuse or adapt LIMA's command + verb stack (for example, in **Rantamuta**).

## 1) What to cannibalize first

If your goal is command/verb handling, start with these files in order:

1. `lib/std/body/cmd.c` — runtime entrypoint for player command parsing (`do_game_command`).
2. `lib/daemons/verb_d.c` — bootstraps all verb objects under `lib/cmds/verbs/`.
3. `lib/std/verb_ob.c` — shared behavior for verb objects (`add_rules`, `can_*`, `do_*` patterns).
4. `lib/secure/daemons/cmd_d.c` — command discovery + argument parsing for shell/player command objects.
5. Example verbs (`lib/cmds/verbs/look.c`, `lib/cmds/verbs/get.c`) and one non-verb command (`lib/cmds/player/help.c`).

## 2) High-level split: "verbs" vs "commands"

LIMA uses **two layers** that are easy to confuse:

- **Verbs** (`lib/cmds/verbs/*.c`): natural-language actions handled through parser grammar (`look at lamp`, `get sword from chest`, etc.).
- **Commands** (`lib/cmds/player/*.c`, `lib/trans/cmds/*.c`, etc.): shell-style command objects (`help`, `ls`, `cd`, tooling, admin commands).

This split is why both `verb_d.c` and `cmd_d.c` exist.

## 3) Runtime flow (player input)

### Verb/natural-language path

1. User enters input.
2. `do_game_command()` in `lib/std/body/cmd.c` gathers parseable objects and calls `parse_sentence(...)`.
3. Parser resolves grammar/rules and dispatches into verb object handlers (`can_*`, `do_*`).
4. If parse returns user-facing text, it is written directly.

Fallback behavior in `do_game_command()`:
- If parser cannot resolve input as a verb, it attempts `go <input>` to treat bare words as exits.
- It filters generic parser errors (`"You can't go ..."`, `"There is no ..."`) from that fallback path.

### Command-object path (shell-style commands)

When a body/module wants command-object dispatch (or NPC scripted command execution), it goes through `CMD_D`:

1. Resolve command by name and search path (`find_cmd_in_path` / `find_cmd`).
2. Validate command object (`call_main` exists via `CMD` inheritance contract).
3. Parse command args according to optional per-directory `Cmd_rules` prototypes.
4. Execute command object (`call_main` / command `main` flow).

## 4) Key internals worth porting

## 4.1 `lib/std/body/cmd.c` (player command loop)

- `do_game_command(string str, int debug)` is the practical entrypoint.
- Handles:
  - environment safety (move to `VOID_ROOM` if player is misplaced),
  - parser invocation,
  - parser result handling (`0`, `1`, `-1`, `-2`, or string),
  - exit fallback (`go <input>`).

If porting to another game, this is your minimal orchestration skeleton.

## 4.2 `lib/daemons/verb_d.c` (verb loading)

- On creation, scans `CMD_DIR_VERBS/*.c` and reloads each verb object.
- Each verb object self-registers its own grammar/rules.

This is a clean pattern for modular grammar extension: one file per verb.

## 4.3 `lib/std/verb_ob.c` (shared verb behavior)

- Shared helper behavior for verbs (including defaults for multi-object handling helpers such as `do_verb_obs`).
- Provides useful default message behavior for empty grammar cases (`"You can't just <verb> ..."`).

When reusing verb files, ensure your target engine provides equivalents to these helpers.

## 4.4 `lib/secure/daemons/cmd_d.c` (command discovery + arg parsing)

This daemon is the command-system heavy lifter:

- Caches command directories and files (`cache_dir`, `cache`).
- Parses optional `Cmd_rules` files into argument prototypes (`parse_verb_defs`).
- Resolves commands from search path (`find_cmd_in_path`) or absolute names (`find_cmd`).
- Performs smart argument parsing and expansion in `smart_arg_parsing(...)`:
  - file/path expansion,
  - object/user coercion,
  - optional/plural argument semantics.

If your port goal includes shell-like commands with typed args, this is the most reusable subsystem.

## 5) How verbs are authored (pattern)

A canonical verb object (`look.c`, `get.c`) typically:

1. `inherit VERB_OB;` (and optional helpers like `M_GRAMMAR`).
2. In `create()`, call `add_rules(...)` with grammar variants + aliases.
3. Implement:
   - `can_<verb>_<rule>()` for permission/validation messages,
   - `do_<verb>_<rule>()` for action execution.

Examples:

- `look.c` demonstrates rich grammar (`"at OBJ"`, `"for OBS"`, `"at OBS with OBJ"`) and aliasing (`examine`, `gaze`, `stare`).
- `get.c` shows inventory/container rules and plural/object handling with helper dispatch.

## 6) Proposed "convergence" process (you + me)

To build a lasting doc agents can navigate quickly, I suggest this iterative process:

### Phase A — Inventory and contracts (short)

- Produce a compact table: **component**, **file**, **responsibility**, **inputs/outputs**.
- Mark hard contracts to preserve when porting (`do_game_command`, `add_rules`, `can_*`/`do_*`, command `call_main`).

### Phase B — Sequence diagrams (medium)

- Add 2 concrete flows:
  1. `look at sword`
  2. `help parser`
- For each step, list the exact file/function involved.

### Phase C — Portability notes (high value)

- For each component, classify:
  - **portable as-is**,
  - **portable with adapter**,
  - **LIMA-specific coupling**.
- Add a "minimum viable transplant" checklist for Rantamuta.

### Phase D — Living examples

- Keep 3 annotated exemplar files:
  - one simple verb,
  - one complex verb,
  - one command object.
- This gives agents concrete templates to clone.

## 7) Side quest map: perspective-aware messaging system

LIMA’s dynamic message perspective system lives primarily in:

- `lib/std/modules/m_messages.c` (core formatting + dispatch)
- `lib/std/living/messages.c` (living/body convenience wrappers)
- `lib/daemons/messages_d.c` (default message set storage/loading)
- `lib/daemons/soul_d.c` (social/emote integration using same tokenized grammar)

### 7.1 Why one message string becomes many perspectives

The core idea is: coder writes one tokenized template, then `compose_message()` renders distinct strings for each recipient perspective (`you` vs third person), including pronouns, possessives, reflexives, articles, and verb conjugation.

Typical usage from verbs/modules:

- `simple_action(msg, obs...)`
  - sends actor view + others view
- `targetted_action(msg, target, obs...)`
  - sends actor view + target view + bystander view
- `my_action(msg, ...)` / `other_action(msg, ...)`
  - actor-only or others-only variants

### 7.2 Token language used in templates

`compose_message()` parses `$` tokens and maps them by recipient:

- `$N` / `$n` : participant name/pronoun forms
- `$T` / `$t` : target participant shorthand (defaults to participant index 1)
- `$v` : verb stem conjugated per viewer (`drink` -> `drink`/`drinks`)
- `$p` : possessive (`your`/`his`)
- `$r` : reflexive (`yourself`/`himself`)
- `$o` : object arguments supplied to action call

Numeric suffixes select participant/object index (`$n1`, `$p1`, `$o2`, etc.).

### 7.3 Dispatch model (who receives what)

Action helpers generate perspective-specific strings and then deliver via messaging APIs:

- actor (`tell(this_object(), us, ...)`)
- target if present (`tell(target, them, ...)`)
- actor inventory/listeners (`tell_from_outside(...)`)
- room/environment bystanders (`tell_environment(...)`)

This is why one template can correctly produce:

- actor: `You drink from the flask.`
- room: `Beek drinks from the flask.`

without writing two separate strings.

### 7.4 Custom/default message packs

`M_MESSAGES` supports per-object message classes (`add_msg`, `set_msgs`, `query_msg`) with fallback to `MESSAGES_D` defaults via `def_message_type` (e.g. `living-default`).

`MESSAGES_D` loads file-backed message groups from `/data/messages/*`, supports random selection when multiple lines exist, and validates combat message schemas.

### 7.5 Social/emote layer uses same perspective engine

`SOUL_D` inherits `M_MESSAGES` and composes emote outputs using parsed emote rules and the same action/token mechanics. That means socials and verbs share the same grammar-aware rendering model rather than having separate hard-coded output logic.

## 8) Immediate next deliverables I can do

Pick any of these and I can generate it in the next pass:

1. **Command/Verb Call Graph** (markdown with function-level edges).
2. **Rantamuta Porting Checklist** (ordered, risk-tagged).
3. **Annotated examples** for `look.c`, `get.c`, and `help.c` with "what to keep vs replace".
4. **Glossary** of parser tokens/rules used by LIMA verbs (`OBJ`, `OBS`, `STR`, `WRD`, etc.).

---

If you want, we can treat this file as the canonical seed and iterate on it directly until it becomes your permanent onboarding map.

## 9) High-confidence pass (porting-oriented, citation-first)

This section is intentionally **normative** and separates claims by confidence class:

- **Observed**: directly visible in repository code.
- **Inferred**: likely true from callsites/structure, but not fully local to this repo.
- **Suggested porting strategy**: implementation guidance for Rantamuta.

Primary evidence roots for this pass:
- Player input and parser fallback logic in `do_game_command()`. *(Observed; source: `lib/std/body/cmd.c:L35-L140`)*
- Wizard shell command dispatch in `execute_command()`. *(Observed; source: `lib/trans/obj/wish.c:L161-L333`)*
- Command resolution and argument parsing in `CMD_D`. *(Observed; source: `lib/secure/daemons/cmd_d.c:L66-L140`, `lib/secure/daemons/cmd_d.c:L214-L330`)*
- Verb bootstrap and verb base contracts in `VERB_D` + `VERB_OB`. *(Observed; source: `lib/daemons/verb_d.c:L48-L54`, `lib/std/verb_ob.c:L16-L27`, `lib/std/verb_ob.c:L109-L145`)*
- Messaging composition and delivery in `M_MESSAGES`. *(Observed; source: `lib/std/modules/m_messages.c:L213-L416`, `lib/std/modules/m_messages.c:L545-L642`)*

## 10) Phase A — Inventory + contracts

| component | file | key functions | inputs | outputs | invariants | portability class |
|---|---|---|---|---|---|---|
| Player command entry | `lib/std/body/cmd.c` | `do_game_command()` | raw input string + optional debug | parser return handling + writes + fallback attempt | missing environment is healed by move to `VOID_ROOM`; parser return codes handled explicitly (`0/1/-1/-2/string`) | **Portable with adapter** (depends on parser/object model) |
| Verb bootstrap | `lib/daemons/verb_d.c` | `create()`, `reload_verb()` | files in `CMD_DIR_VERBS` | verb objects loaded/reloaded | each `*.c` verb file is loaded at daemon create | **Portable as-is conceptually** |
| Verb base contract | `lib/std/verb_ob.c` | `add_rules()`, `can_verb_rule()`, `do_verb_obj()`, `do_verb_obs()` | verb grammar rules + objects | parser rule registration + default checks + action dispatch | default checks include vision/ghost/condition flags | **Portable with adapter** |
| Command lookup + typed args | `lib/secure/daemons/cmd_d.c` | `find_cmd_in_path()`, `find_cmd()`, `smart_arg_parsing()` | command token + path + argv + optional `Cmd_rules` | command object and parsed args or error/int status | command object must expose `call_main` from `CMD`; optional `Cmd_rules` prototype controls coercion/expansion | **Portable with adapter** |
| Command execution contract | `lib/obj/secure/cmd.c` | `call_main()` | parsed command args/stdin/pipeline metadata | calls command `main()` then returns output | caller must be shell object; `main()` is command-specific | **Portable with adapter** |
| Shell orchestration | `lib/trans/obj/wish.c` | `execute_command()` | original user input | shell-builtins -> CMD_D commands -> parser verbs -> channels | command-path is attempted before verb parser fallback | **LIMA-specific coupling** |
| Message composition + fanout | `lib/std/modules/m_messages.c` | `compose_message()`, `simple_action()`, `targetted_action()` | tokenized template + participants + object args | per-recipient rendered strings + delivery | one template renders perspective-aware grammar and pronouns | **Portable with adapter** |

Contract evidence:
- `do_game_command()` branch behavior and fallback. *(Observed; source: `lib/std/body/cmd.c:L35-L140`)*
- command-before-verb ordering in shell. *(Observed; source: `lib/trans/obj/wish.c:L244-L325`)*
- command object `call_main` contract and caller restriction. *(Observed; source: `lib/obj/secure/cmd.c:L28-L53`)*

## 11) Phase B — concrete traces

### Trace #1: `look at sword`

1. User input enters shell command pipeline via `execute_command(original_input)`. *(Observed; source: `lib/trans/obj/wish.c:L161-L245`)*
2. Shell tries command-object dispatch first with `CMD_D->smart_arg_parsing(argv, path, implode_info)`. *(Observed; source: `lib/trans/obj/wish.c:L244-L277`)*
3. If no command resolves, shell asks body parser via `this_body()->do_game_command(original_input)`. *(Observed; source: `lib/trans/obj/wish.c:L306-L311`)*
4. `do_game_command()` builds parser object set and calls `parse_sentence(str, debug, objs)`. *(Observed; source: `lib/std/body/cmd.c:L57-L65`)*
5. Verb `look` file has rules including `"at OBJ"` and handlers like `do_look_at_obj(object ob, string name)`, which emits description and may run `targetted_other_action`. *(Observed; source: `lib/cmds/verbs/look.c:L24-L27`, `lib/cmds/verbs/look.c:L46-L60`)*
6. Message fanout is perspective-aware via `targetted_other_action()` / `simple_action()` in `M_MESSAGES`. *(Observed; source: `lib/std/modules/m_messages.c:L626-L642`, `lib/std/modules/m_messages.c:L545-L564`)*

Failure modes in this trace:
- parser returns `-1` => nonsense message; `-2` => ability failure text. *(Observed; source: `lib/std/body/cmd.c:L93-L99`)*
- unresolved as verb may get `go <input>` fallback and filtered generic errors. *(Observed; source: `lib/std/body/cmd.c:L120-L139`)*

### Trace #2: `help parser`

1. Shell tokenizes/expands then executes command lookup via `CMD_D->smart_arg_parsing(...)`. *(Observed; source: `lib/trans/obj/wish.c:L176-L277`)*
2. `CMD_D` resolves command object using `find_cmd()` / `find_cmd_in_path()`. *(Observed; source: `lib/secure/daemons/cmd_d.c:L214-L263`, `lib/secure/daemons/cmd_d.c:L265-L330`)*
3. Shell invokes command object through `cmd_info[0]->call_main(...)`. *(Observed; source: `lib/trans/obj/wish.c:L252-L255`)*
4. Base `CMD` object enforces caller legitimacy and calls command `main(...)`. *(Observed; source: `lib/obj/secure/cmd.c:L34-L53`)*
5. `help` command `main(arg)` instantiates `HELPSYS` and calls `begin_help(arg)`. *(Observed; source: `lib/cmds/player/help.c:L28-L36`)*

Failure modes in this trace:
- command exists but uncallable => shell prints “Found command is uncallable.” *(Observed; source: `lib/trans/obj/wish.c:L288-L293`)*
- no command/verb recognized => shell reports unknown verb. *(Observed; source: `lib/trans/obj/wish.c:L321-L331`)*

## 12) Phase C — Rantamuta porting checklist (ordered, risk-tagged)

### 12.1 Minimum viable transplant

1. [LOW] Port tokenized action rendering (`compose_message`-style) before porting verbs, so message templates remain stable while behavior ports incrementally. *(Suggested; observed dependency: `lib/std/modules/m_messages.c:L213-L416`)*
2. [MED] Port verb bootstrap (scan/load verb modules) and rule registration API equivalent to `parse_add_rule`/`parse_add_synonym`. *(Suggested; observed shape: `lib/daemons/verb_d.c:L48-L54`, `lib/std/verb_ob.c:L16-L27`)*
3. [MED] Port body-level command entry with return-code handling and fallback semantics (`0/1/-1/-2/string`, plus `go` retry). *(Suggested; observed: `lib/std/body/cmd.c:L71-L140`)*
4. [HIGH] Port only a narrow command object path: find command by search path + invoke `call_main` equivalent. Defer full `Cmd_rules` coercion until stable. *(Suggested; observed interfaces: `lib/secure/daemons/cmd_d.c:L214-L263`, `lib/obj/secure/cmd.c:L28-L53`)*
5. [MED] Port `look`, `get`, `help` as acceptance exemplars (one descriptive verb, one inventory verb, one command object). *(Suggested; observed examples: `lib/cmds/verbs/look.c:L24-L120`, `lib/cmds/verbs/get.c:L187-L223`, `lib/cmds/player/help.c:L28-L36`)*

### 12.2 Full-fidelity transplant

1. [HIGH] Port `CMD_D` prototype parser (`Cmd_rules`) including plural/optional/type coercion paths. Failure risk: command incompatibility and false-positive parsing. *(Suggested; observed complexity: `lib/secure/daemons/cmd_d.c:L66-L140`, `lib/secure/daemons/cmd_d.c:L318-L340`)*
2. [HIGH] Port shell orchestration order (shell builtins -> command objects -> body parser -> channels) to preserve user expectations. *(Suggested; observed order: `lib/trans/obj/wish.c:L219-L325`)*
3. [MED] Port perspective message tokens and index semantics including reflexive/possessive edge cases (`$n1`, `$p1`, `$r`, `$o2`). *(Suggested; observed token handling: `lib/std/modules/m_messages.c:L231-L410`)*
4. [MED] Port emote/soul integration onto same message engine to avoid dual rendering stacks. *(Suggested; observed coupling: `lib/daemons/soul_d.c:L6-L9`, `lib/daemons/soul_d.c:L131-L202`)*
5. [HIGH] Reproduce security/caller contracts for command invocation to avoid command spoofing. *(Suggested; observed check: `lib/obj/secure/cmd.c:L34-L36`)*

## 13) Phase D — annotated examples (keep vs replace)

### 13.1 `lib/cmds/verbs/look.c`

**Role in system (Observed)**
- Registers many grammar rules and aliases for “look/examine/gaze/stare” interactions. *(source: `lib/cmds/verbs/look.c:L24-L27`, `lib/cmds/verbs/look.c:L184-L216`)*
- Uses `this_body()->force_look(1)` for room self-look and object-specific handlers for targeted lookups. *(source: `lib/cmds/verbs/look.c:L41-L60`)*

**Contracts it depends on (Observed)**
- `VERB_OB` + parser rule system (`add_rules`). *(source: `lib/cmds/verbs/look.c:L16-L27`, `lib/std/verb_ob.c:L16-L27`)*
- Object description API (`get_item_desc`, `long`, `is_living`). *(source: `lib/cmds/verbs/look.c:L46-L58`)*
- Messaging helpers (`targetted_other_action`). *(source: `lib/cmds/verbs/look.c:L54-L55`, `lib/std/modules/m_messages.c:L626-L642`)*

**Keep as-is (Suggested)**
1. Rule granularity and aliasing breadth.
2. Object-first dispatch style (`do_look_at_obj`, `do_look_for_obj`).
3. Use of messaging templates rather than hard-coded actor/room strings.

**Replace/adapt for Rantamuta (Suggested)**
1. Parser token API (`add_rules`, `parse_add_rule` backend) must map to your parser.
2. Environment/object API names (`get_item_desc`, `query_prep`, `look_in`) likely differ.
3. Messaging dispatch hooks (`targetted_other_action`) must map to your event bus.

### 13.2 `lib/cmds/verbs/get.c`

**Role in system (Observed)**
- Implements item acquisition rules (`OBS`, `OBS from OBJ`, relation forms, money forms) with aliases (`take`, `carry`, `pick up`). *(source: `lib/cmds/verbs/get.c:L216-L223`)*
- Central move/take logic in `get_one()` handles relation checks, `MOVE_*` outcomes, and action emission. *(source: `lib/cmds/verbs/get.c:L24-L78`)*

**Contracts it depends on (Observed)**
- Move constants and object movement semantics (`MOVE_OK`, `MOVE_NO_ROOM`, `move`). *(source: `lib/cmds/verbs/get.c:L58-L78`)*
- Relation API from containers (`query_relation`, `query_relation_aliases`). *(source: `lib/cmds/verbs/get.c:L31-L47`)*
- Body and messaging APIs (`this_body()`, `other_action`). *(source: `lib/cmds/verbs/get.c:L61-L64`)*

**Keep as-is (Suggested)**
1. Single internal function (`get_one`) for consistency.
2. Explicit handling of move error classes.
3. Rule aliases and relation-phrase support.

**Replace/adapt for Rantamuta (Suggested)**
1. Move/equipment constants and transfer semantics.
2. Container relation APIs and relation aliases.
3. Money denomination integration (`MONEY_D`) and present lookup semantics.

### 13.3 `lib/cmds/player/help.c`

**Role in system (Observed)**
- Thin command object that delegates to help subsystem (`new (HELPSYS)->begin_help(arg)`). *(source: `lib/cmds/player/help.c:L28-L33`)*

**Contracts it depends on (Observed)**
- `CMD` inheritance contract for `main(...)` execution via command system. *(source: `lib/cmds/player/help.c:L24-L33`, `lib/obj/secure/cmd.c:L28-L53`)*
- Shell/command resolver path to reach command object. *(source: `lib/trans/obj/wish.c:L244-L255`, `lib/secure/daemons/cmd_d.c:L214-L263`)*

**Keep as-is (Suggested)**
1. Thin-controller approach (command delegates to subsystem).
2. No parser coupling in command body.
3. Menu entry wrapper pattern if your UI uses menu hooks.

**Replace/adapt for Rantamuta (Suggested)**
1. `HELPSYS` object path + lifecycle model.
2. Command base class API (`CMD`, `main` signature).
3. Menu entry wiring if your shell does not support `player_menu_entry()`.

## 14) Glossary — parser tokens/rules used by LIMA verbs

> Confidence note: token semantics below are **Observed** from verb rule usage and handler signatures; parser-internal grammar implementation is external to this file set and may add additional constraints. *(Observed usage: `lib/cmds/verbs/look.c:L24-L40`, `lib/cmds/verbs/get.c:L216-L223`; requires parser internals for full formal grammar.)*

- `OBJ`
  - **Observed usage**: single object token in rules like `"at OBJ"`, `"OBJ with OBJ"`; handlers receive object parameters. *(source: `lib/cmds/verbs/look.c:L24-L27`, `lib/cmds/verbs/get.c:L106-L109`)*
  - **Inferred semantics**: resolves exactly one object candidate.

- `OBS`
  - **Observed usage**: multi-object token in rules like `"for OBS"`, `"OBS from OBJ"`; handlers receive collection (`string *info`/object list) and iterate via `handle_obs`. *(source: `lib/cmds/verbs/look.c:L24-L27`, `lib/cmds/verbs/look.c:L74-L77`, `lib/cmds/verbs/get.c:L86-L99`, `lib/std/verb_ob.c:L127-L139`)*
  - **Inferred semantics**: resolves one-or-more object matches.

- `STR`
  - **Observed usage**: freeform string token in rules like `"STR"`, `"at STR"`; handlers receive string argument. *(source: `lib/cmds/verbs/look.c:L24-L36`, `lib/cmds/verbs/get.c:L121-L156`)*
  - **Inferred semantics**: textual phrase not resolved as object.

- `WRD`
  - **Observed usage**: relation/preposition token in rules like `"WRD OBJ"`, `"OBS from WRD OBJ"`; handlers receive relation string then object(s). *(source: `lib/cmds/verbs/look.c:L24-L27`, `lib/cmds/verbs/get.c:L101-L104`, `lib/cmds/verbs/get.c:L111-L114`)*
  - **Inferred semantics**: single lexical word (often preposition/relation).

- Token index suffixes in message templates (`$n1`, `$p1`, `$o2`)
  - **Observed usage**: numeric indices select participant/object slots during composition. *(source: `lib/std/modules/m_messages.c:L239-L259`, `lib/std/modules/m_messages.c:L262-L267`)*
  - **Observed edge case**: `$t/$T` defaults to participant index 1. *(source: `lib/std/modules/m_messages.c:L257-L258`)*

## 15) Open ambiguities (requires maintainer confirmation)

1. Full formal semantics of parser tokens (`OBJ/OBS/STR/WRD`) are not defined in the sampled verb files; only usage is visible.
   - Plausible interpretation A: token behavior is entirely parser-efun-defined and stable across verbs.
   - Plausible interpretation B: mudlib-side parser setup/mixins may further constrain resolution behavior per environment.
   - **Requires maintainer confirmation**.

2. Exact parser dispatch order between parser internals and verb object handlers (beyond observed outcomes) is not fully visible in this repo segment.
   - Plausible interpretation A: parser resolves rule and directly calls matching `can_`/`do_` chain.
   - Plausible interpretation B: parser uses intermediate compatibility layers before invoking verb handlers.
   - **Requires maintainer confirmation**.

## 16) Acceptance transcript appendix (canonical inputs)

Purpose: quick regression-style checks for future agent porting passes.

Legend:
- **Actor output** = what the initiator should see.
- **Room output** = what bystanders in environment should see.
- **System/shell output** = shell/framework-level fallback or errors.

| Input | Function path (observed) | Actor output (expected) | Room output (expected) | System/shell output (expected) |
|---|---|---|---|---|
| `look` | `wish::execute_command` -> `body::do_game_command` -> parser -> `look::do_look` -> `force_look(1)` | Room description via look subsystem. | None required by `do_look` itself. | None. |
| `look at sword` | `wish::execute_command` -> `body::do_game_command` -> parser -> `look::do_look_at_obj` | Item description from `get_item_desc(name)` or `long()`. | For non-living target: none required. For living target: target/others get `"$N $vlook at $t."` via `targetted_other_action`. | None. |
| `look at nosuchthing` | `wish::execute_command` -> `body::do_game_command` -> parser -> `look::do_look_at_str` | `"You do not see that here."` if room declines lookup. | None required. | None. |
| `get sword` | `wish::execute_command` -> `body::do_game_command` -> parser -> `get::do_get_obj` -> `get_one` | On success: `"<Short>: Taken."`; on failure, explicit move/get error text. | On success: room gets `"$N $vtake a $o."` through `other_action`. | None. |
| `get sword from chest` | `wish::execute_command` -> `body::do_game_command` -> parser -> `get::do_get_obj_from_obj` -> `get_one` | Same success/failure class as `get sword`. | Same as `get sword` success path (`other_action`). | None. |
| `get sword from in chest` (relation form) | `wish::execute_command` -> `body::do_game_command` -> parser -> `get::do_get_obj_from_wrd_obj` -> `get_one(rel)` | If relation mismatch: `"It's not <rel> there."`; otherwise same as `get sword`. | Same as `get sword` on success. | None. |
| `help parser` | `wish::execute_command` -> `CMD_D::smart_arg_parsing` -> command `call_main` -> `help::main` -> `HELPSYS->begin_help(arg)` | Help UI/help text flow begins for `parser` topic. | None required. | None. |
| `frobnicate` | `wish::execute_command` -> command path miss -> parser miss -> shell unknown-verb fallback | None guaranteed from verb handlers. | None. | `"I don't know the verb 'frobnicate'."` |

### 16.1 Source anchors for transcript rows

- Shell orchestration and unknown verb fallback: `execute_command()` command-first, parser-second, then error text. *(Observed; source: `lib/trans/obj/wish.c:L244-L325`)*
- Body parser return and fallback behavior: `do_game_command()`. *(Observed; source: `lib/std/body/cmd.c:L64-L140`)*
- `look` command handlers and strings. *(Observed; source: `lib/cmds/verbs/look.c:L46-L70`)*
- `get` command handlers and strings. *(Observed; source: `lib/cmds/verbs/get.c:L29-L112`)*
- Help command delegation to help subsystem. *(Observed; source: `lib/cmds/player/help.c:L28-L31`)*
- Message fanout semantics for actor/room/target paths. *(Observed; source: `lib/std/modules/m_messages.c:L545-L642`)*
