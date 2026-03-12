# LIMA Messaging System Deep Dive (for Porting)

This document explains LIMA's perspective-aware messaging model in detail for porting teams.
It is intentionally over-explained and includes both architecture and practical gotchas.

---

## 1) Concise summary

LIMA treats a player action as a **single semantic event** and renders that event into different text for different recipients (actor, target, bystanders).

That is why one template can produce:

- actor: `You wave.`
- target: `Foo waves to you.`
- others: `Foo waves to Bar.`

The system does this through three layers:

1. **Template language** (`$N`, `$t`, `$v`, `$p`, `$o`, etc.)
2. **Perspective renderer** (`compose_message`)
3. **Scope broadcasters** (`simple_action`, `targetted_action`, etc.)

---

## 2) What problem it solves

### Observed behavior
- LIMA avoids hand-writing separate strings per audience in most action code.
- The sender provides one tokenized template, and recipients get role-correct language.

### Inferred behavior
- This reduces duplication, grammar bugs, and perspective drift between actor/target/room messages.

### Suggested porting strategy
- Keep this "single canonical event -> multi-view render" model as the center of your implementation.

---

## 3) Mental model: event -> render -> dispatch

Think in three phases:

1. **Event formation**
   - Action code decides *what happened* and who participated.
   - Example participants: actor `who[0]`, target `who[1]`.

2. **View rendering**
   - For each recipient perspective, render one string from the same event template.

3. **Audience dispatch**
   - Send each rendered message to the right scope (self, target, room, carried entities, etc.), while avoiding duplicates.

This strict separation is the reason the system scales from simple emotes to elaborate multi-target social text.

---

## 4) The template token language (core grammar)

LIMA token parsing matches placeholders of this family:

- `$N`, `$n` (person reference)
- `$T`, `$t` (target-oriented person reference)
- `$V`, `$v` (verb inflection)
- `$P`, `$p` (possessive)
- `$R`, `$r` (reflexive)
- `$O`, `$o` (object/string slot)

Tokens can include numeric and letter suffixes (for participant indexing and pronoun behavior).

### Practical reading of common tokens

- `$N` often means "actor as name/pronoun depending on perspective and mention history".
- `$t` defaults toward the target participant slot.
- `$vverb` inflects to recipient perspective (`wave` vs `waves`).
- `$p` gives possessive (`your` vs `his/her` vs named possessive).
- `$o` inserts object/string argument with article/list shaping logic.

### Important subtlety
Uppercase token heads force capitalization of the resolved fragment.

---

## 5) Pronouns and grammar intelligence

The renderer is not a dumb search-replace. It tracks state while composing:

1. **Mention tracking**
   - First mention tends toward proper-name-like rendering.
   - Later mentions shift to pronouns where appropriate.

2. **Reflexive correction**
   - If subject and object resolve to the same participant, it emits reflexives (`yourself`, `himself`, etc.) instead of awkward repeated names.

3. **Verb agreement**
   - `$v` inflects differently depending on whether the recipient is the subject participant.

4. **Possessive nuance**
   - Supports plain possessive pronoun and named possessive forms.

5. **Object phrase shaping**
   - Handles single object, list of objects, and article/the-prefix behavior.

### Why this matters
These are exactly the details that make social output feel "human" rather than templated.

---

## 6) Broadcast scope functions (the operational API)

LIMA exposes multiple message-sending helpers with different audience contracts.

### `simple_action(template, ...)`
- Sends actor-specific render to actor.
- Sends "others" render to surrounding recipients.
- Use for "I did X and room saw X".

### `my_action(template, ...)`
- Actor only.
- Use for private feedback (`You ...`).

### `other_action(template, ...)`
- Others only, excludes actor.
- Use when actor already got explicit direct text elsewhere.

### `targetted_action(template, target, ...)`
- Actor gets actor render.
- Target gets target render.
- Everyone else gets bystander render.
- This is the canonical function for `wave to bar` style interactions.

### `targetted_other_action(template, target, ...)`
- Target + bystanders only, no actor message.
- Useful when actor already got custom output and you still want social broadcast.

### Lower-level pair
- `action(...)` computes all perspective strings first.
- `inform(...)` sends them and de-duplicates recipients.

---

## 7) Souls/emotes: data-driven social layer

LIMA's soul system (`SOUL_D`) stores emote rules and message templates, then runs them through the same renderer.

- Rule examples: empty rule, `LIV`, `STR`, and combinations.
- A rule determines participant layout and arguments passed to template rendering.
- A single soul can have multiple templates, including audience-specific template splits.

### Multi-part soul messages
`addemote` supports message splitting with `&&`:

- first part for actor,
- second for target,
- third for others,
- and so on.

This is one mechanism behind highly expressive social output.

---

## 8) Why your remembered `wave` examples work

For social behavior, the typical data pattern is effectively:

- template for bare wave: `$N $vwave.`
- template for targeted wave: `$N $vwave to $t.`

From that one targeted template, render outcomes naturally become:

- actor sees "You wave to Bar."
- Bar sees "Foo waves to you."
- room sees "Foo waves to Bar."

The same holds for complex lines (`$N puts $o on $r`, etc.) because reflexive, possessive, and perspective logic are built into rendering.

---

## 9) High-value gotchas (important for ports)

These are the pain points that tend to bite reimplementations.

### Gotcha 1: Participant indexing discipline
If your participant array ordering is inconsistent, token suffixes (`$N1`, `$t10`, `$v1run`) produce wrong recipients or grammar.

**Mitigation:** define and enforce participant slot contracts per action family.

### Gotcha 2: `$t` default target behavior
`$t/$T` default toward target slot semantics; this is convenient but easy to misuse in non-target actions.

**Mitigation:** require explicit indexing in templates where ambiguity is possible.

### Gotcha 3: Duplicate delivery
Recipients may appear in multiple collections (inventory/environment/target scopes).

**Mitigation:** de-dup by recipient identity before send.

### Gotcha 4: Grammar edge cases around reflexives
Without subject/object identity checks, you get awkward output (`Foo waves to Foo`).

**Mitigation:** preserve reflexive rewrite logic in renderer core.

### Gotcha 5: Object phrases and articles
Hardcoding "a/an/the" outside the renderer often causes mismatches for lists/plurals.

**Mitigation:** centralize article/plural shaping with object-slot rendering.

### Gotcha 6: Mixed command families
Not all text goes through exactly the same path (e.g., custom speech/history channels may take special routes).

**Mitigation:** explicitly catalog "standard action pipeline" vs "special channel pipeline" in your port.

### Gotcha 7: Silent fallback failures
If unknown/invalid tokens silently pass, debugging authored templates becomes painful.

**Mitigation:** include strict validation mode for token templates and surface diagnostics.

---

## 10) How to do it better in a modern implementation

If you are re-creating this for Rantamuta (or similar), you can preserve behavior while improving maintainability.

### Suggested improvements

1. **Typed semantic event object**
   - Explicit fields: actor, targets, objects, tense/aspect options, channels.

2. **Template compiler phase**
   - Parse token templates once into AST/opcodes; render many times efficiently.

3. **Static lint for templates**
   - Validate unknown token types, out-of-range participant indices, and impossible combinations before runtime.

4. **Deterministic recipient pipeline**
   - Compute recipient sets once per event; keep ordering deterministic and documented.

5. **Message test harness**
   - Snapshot tests for actor/target/others across canonical scenarios.
   - Include edge tests for reflexive, duplicate recipients, and multi-object lists.

6. **Compatibility levels**
   - `semantic-equivalent` mode (default) for similar behavior.
   - `strict-text` mode only for commands where exact legacy wording matters.

7. **Observability hooks**
   - Log event + rendered outputs + recipients during development to catch perspective bugs quickly.

---

## 11) Minimal contract to preserve "LIMA-like feel"

If you only preserve a subset, preserve this:

1. One canonical template per semantic event.
2. Per-recipient render with pronoun/verb/reflexive logic.
3. Distinct scope broadcasters (self-only, others-only, targetted).
4. Deterministic recipient de-duplication.

That gives you most of the recognizable behavior users care about.

---

## 12) Ambiguities / maintainer confirmation points

1. Exact recipient ordering expectations in all edge scopes (especially nested containers and mixed channels).
2. Exact parity requirements for wording vs semantic equivalence in each command family.
3. Which command paths intentionally bypass standard action broadcasting.

These should be decided early to avoid divergent assumptions during porting.

---

## 13) Quick implementation sketch for Codex

Use an interface roughly like:

- `render(event, recipient) -> text`
- `render_all(event, participants) -> {recipient->text, others_text}`
- `dispatch(scope_policy, rendered_bundle)`

Where `scope_policy` corresponds to:

- simple
- my
- other
- targetted
- targetted_other

and templates are authored in the token language described above.

This keeps behavior portable and implementation details replaceable.

