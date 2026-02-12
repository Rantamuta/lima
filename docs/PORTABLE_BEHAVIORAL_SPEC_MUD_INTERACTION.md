# Portable Behavioral Spec: Command-to-Output Interaction Model

This document defines a portable behavioral contract for a text-MUD interaction pipeline. It is intended for implementers who do not have access to original internals.

## 1) Evidence snapshot (short)

- **Evidence source:** Runtime interaction paths and user-visible outcomes for command handling, fallback behavior, and action success/failure.
  - **Confidence:** High for branch outcomes and fallback order.
- **Evidence source:** User-visible interaction patterns for common command shapes (single token, relation form, multi-target phrasing).
  - **Confidence:** High for input classes and recipient classes.
- **Evidence source:** Message rendering/dispatch behavior where one action yields audience-specific phrasing.
  - **Confidence:** Medium-High for perspective transformation and fanout behavior.
- **Evidence source:** Existing repository architecture notes used as context only.
  - **Confidence:** Medium for intent framing; non-authoritative for runtime facts.

## 2) System overview (short)

- **Observed behavior:** The pipeline accepts player input and attempts to interpret it as an in-world action.
- **Observed behavior:** Successful interpretation can mutate world state and then emit audience-specific messages.
- **Observed behavior:** Failed interpretation emits an explicit failure class.
- **Inferred behavior:** The model balances natural language flexibility with deterministic branch handling.
- **Suggested porting strategy:** Preserve branch order and error taxonomy before preserving exact text wording.

**Start point:** one actor input string.

**End point:** exactly one terminal class:
1. success (with delivery),
2. handled failure,
3. unknown after fallback exhaustion.

## 3) Neutral glossary

- **Actor:** issuer of the input.
- **Primary target:** direct object/entity of the action.
- **Secondary target:** indirect object/entity in relation-oriented actions.
- **Bystander:** eligible recipient who is neither actor nor primary target.
- **Resolver:** stage mapping normalized input to intent candidates.
- **Target resolver:** stage binding textual references to concrete entities.
- **Validator:** stage checking feasibility, permissions, and constraints.
- **Executor:** stage applying persistent world mutation.
- **Semantic event:** canonical action representation prior to audience rendering.
- **Renderer:** stage converting one semantic event into audience-specific strings.
- **Dispatcher:** stage delivering rendered strings to eligible recipients.
- **Fallback chain:** ordered secondary interpretation attempts triggered only by unknown intent.
- **Eligible recipient:** recipient approved by visibility/reachability policy.
- **Error envelope:** machine-assertable failure payload (`class`, `code`, `details`).

## 4) Resolution-result taxonomy

- **Success:** intent, targets, and constraints are satisfied.
- **Unknown intent:** no intent mapping exists for the input form.
- **Semantic error:** intent family is recognized but input form is incomplete/malformed.
- **Ambiguous target:** multiple candidates match a required target slot.
- **Invalid context/target:** target exists but relation/context predicate fails.
- **Forbidden/blocked:** action recognized but denied by policy/rules/capacity.

## 5) Input grammar and normalization contract

- **Observed behavior:** Input is processed as tokenized natural-language command text.
- **Inferred behavior:** Deterministic normalization is required for reproducible branching.
- **Suggested porting strategy:** Use the following canonical normalization pipeline, in order:
  1. Trim surrounding whitespace.
  2. Collapse internal repeated whitespace to single separators outside quoted spans.
  3. Apply case-folding to command words and relation keywords; preserve original case for display payloads.
  4. Preserve quoted substrings as atomic tokens.
  5. Preserve explicit numeric selectors (`N.term`) as selector tokens.
  6. Preserve punctuation only when it is syntactically meaningful to token boundaries.

### Normalization invariants
- **Observed behavior:** normalization MUST run after intake and before interpretation.
- **Observed behavior:** normalization MUST NOT mutate world state.
- **Observed behavior:** normalization MUST NOT emit player-visible output.
- **Suggested porting strategy:** stop-word handling SHOULD be explicit and deterministic; if used, document the list and precedence.

## 6) Deterministic target resolution and precedence

- **Observed behavior:** target resolution is context-sensitive and can return ambiguity.
- **Inferred behavior:** deterministic precedence is required for transcript-level parity.

### Required precedence order
- **Suggested porting strategy:** Resolve target candidates in this order unless maintainers choose a different declared order:
  1. actor inventory,
  2. actor equipped/immediately held items,
  3. current locale contents,
  4. relation scope (`from`, `on`, `in`, `with`) constrained set,
  5. global/remote scope only if command class allows it.

### Tie-breakers
- **Suggested porting strategy:** apply tie-breakers in order:
  1. explicit numeric selector (`2.sword`) if present,
  2. exact alias/name match over partial match,
  3. shortest unambiguous match,
  4. ambiguity failure if still unresolved.

### Resolution invariants
- **Observed behavior:** target resolution MUST NOT mutate persistent world state.
- **Observed behavior:** unresolved ambiguity MUST terminate with ambiguity class (not unknown class).
- **Observed behavior:** relation mismatch MUST terminate with invalid-context class.

## 7) Pipeline with stage invariants (ordered)

> **Global invariant (Observed behavior):** pipeline order MUST be intake -> normalization -> interpretation -> target resolution -> validation -> execution -> rendering -> dispatch, with fallback chain only on unknown intent.

1. **Input intake**
   - **Observed behavior:** system receives raw actor input.
   - **Invariants:** MUST run first; MUST NOT mutate state; MUST NOT emit bystander output.

2. **Normalization**
   - **Observed behavior:** parse-ready token form is produced.
   - **Invariants:** MUST run before interpretation; MUST NOT mutate state; MUST NOT emit output.

3. **Interpretation**
   - **Observed behavior:** resolver returns one taxonomy class.
   - **Invariants:** MUST run before target resolution; MUST NOT mutate state; MUST NOT emit unknown output before fallback chain completes.

4. **Target/context resolution**
   - **Observed behavior:** binds targets and relation context.
   - **Invariants:** MUST run after interpretation success candidate; MUST NOT mutate persistent state; MAY emit terminal ambiguity/invalid-context output.

5. **Validation**
   - **Observed behavior:** checks permission, capacity, and rule predicates.
   - **Invariants:** MUST run before execution; MUST NOT emit success output; SHOULD be side-effect free.

6. **Execution**
   - **Observed behavior:** applies persistent mutation when prior stages pass.
   - **Invariants:** MUST run only after validation pass; MAY mutate state; MUST emit failure class if mutation cannot complete.

7. **Rendering**
   - **Observed behavior:** one semantic event is transformed per audience.
   - **Invariants:** MUST run after successful execution; MUST derive all audience strings from one semantic event; MUST NOT reinterpret semantics per audience.

8. **Dispatch**
   - **Observed behavior:** actor/target/bystander sets can receive distinct strings.
   - **Invariants:** MUST run after rendering; MUST deliver only to eligible recipients; MUST NOT mutate state.

9. **Fallback chain**
   - **Observed behavior:** unknown-intent triggers fallback attempts before terminal unknown.
   - **Invariants:** MUST run only for unknown-intent class; MUST complete before terminal unknown output; MUST NOT emit partial unknown output mid-chain.

## 8) Validation ordering and error precedence

- **Inferred behavior:** user-visible reproducibility requires deterministic validation ordering.
- **Suggested porting strategy:** evaluate validation rules in this order:
  1. actor state preconditions,
  2. target existence/reachability,
  3. relation/context predicates,
  4. permission/policy gates,
  5. capacity/load constraints,
  6. action-specific custom guards.

### Error precedence rule
- **Observed behavior:** first failing rule in the declared order MUST determine the emitted failure class and error envelope.
- **Suggested porting strategy:** do not collapse failures from different classes into one generic denial when deterministic class is available.

## 9) Fallback-chain definition

- **Observed behavior:** fallback is attempted before terminal unknown output.
- **Inferred behavior:** deterministic chain ordering is required for stable transcripts.

### Ordered fallback strategies
- **Suggested porting strategy:** evaluate in order:
  1. direct intent mapping,
  2. alternate shorthand expansion,
  3. relation-form reinterpretation,
  4. movement/navigation reinterpretation,
  5. terminal unknown.

### Fallback termination contract
- **Observed behavior:** if any fallback strategy returns non-unknown terminal class, that class MUST be emitted and chain processing MUST halt.
- **Observed behavior:** terminal unknown MUST be emitted only after all configured fallback strategies return unknown.

## 10) Execution atomicity, concurrency, and recipient snapshot timing

### Atomicity contract
- **Inferred behavior:** command processing should appear atomic at actor-facing granularity.
- **Suggested porting strategy:** implement one command transaction per actor input.
- **Observed behavior:** on execution failure, persistent state SHOULD remain unchanged unless a documented legacy exception exists.

### Concurrency contract
- **Suggested porting strategy:** default to single-queue command processing per actor session.
- **Suggested porting strategy:** if global concurrency exists, interleaving MUST NOT expose partially applied state to recipients of one semantic event.

### Recipient eligibility snapshot timing
- **Inferred behavior:** delivery parity depends on when eligibility is sampled.
- **Suggested porting strategy:** sample eligibility once after successful execution and before dispatch; use that fixed recipient set for the full semantic event.

### Delivery ordering contract
- **Suggested porting strategy:** choose and declare one ordering profile:
  - **Profile A (strict):** actor -> primary target -> bystanders.
  - **Profile B (coherent-batch):** all recipients receive one event batch with no required intra-batch order.
- **Observed behavior:** implementation MUST use exactly one declared profile per deployment and test against it.

## 11) Perspective messaging rules

- **Observed behavior:** one semantic action can produce different actor/target/bystander strings.
- **Observed behavior:** perspective transformation includes person/pronoun/reflexive agreement.
- **Inferred behavior:** article/plural shaping should be deterministic for parity-sensitive tests.

### Required constraints
- **Observed behavior:** all audience outputs MUST derive from one canonical semantic event.
- **Observed behavior:** dispatcher MUST apply eligibility filtering before delivery.
- **Observed behavior:** audience outputs MUST NOT come from separate semantic interpretation paths.

### Output compatibility profile
- **Suggested porting strategy:** define command families in one of two modes:
  1. **Semantic-equivalent mode:** intent, class, recipients, ordering, and mutation are asserted.
  2. **Strict-text mode:** exact output text is additionally asserted.
- **Observed behavior:** each command family MUST declare one mode in test configuration.

## 12) Error envelope contract (machine-assertable)

- **Suggested porting strategy:** expose failure payload as:
  - `class`: one taxonomy class,
  - `code`: stable implementation-specific error key,
  - `details`: structured context (target token, relation token, rule identifier).
- **Observed behavior:** error class and code MUST be stable across runs for identical input and world state.

## 13) Acceptance examples (canonical)

> Unless stated otherwise, outputs are semantic expectations with required recipient class and declared ordering profile.

### Case A: `look`
- **Preconditions:** actor is in valid locale.
- **Expected flow:** intake -> normalize -> interpret -> resolve context -> validate -> execute(read-only) -> render -> dispatch.
- **Recipients:** actor only.
- **Expected outputs:** locale description and visible entities summary.
- **Expected failures:** none.
- **Expected state change:** none.

### Case B: `look at <target>` success
- **Preconditions:** target resolvable.
- **Expected flow:** interpret inspect -> resolve target -> validate -> execute inspect -> render -> dispatch.
- **Recipients:** actor required; bystander visibility profile-dependent.
- **Expected outputs:** target detail.
- **Expected failures:** none.
- **Expected state change:** none.

### Case C: `look at <target>` missing target
- **Preconditions:** target phrase unresolved.
- **Expected flow:** target resolution failure -> invalid-context/target class.
- **Recipients:** actor only.
- **Expected outputs:** explicit not-found message.
- **Expected failures:** terminal.
- **Expected state change:** none.

### Case D: `get <item>` success
- **Preconditions:** item reachable and carryable.
- **Expected flow:** resolve -> validate -> execute transfer -> render -> dispatch.
- **Recipients:** actor + eligible bystanders.
- **Expected outputs:** actor success text; bystander third-person event.
- **Expected failures:** none.
- **Expected state change:** item moves to actor inventory.

### Case E: `get <item> from <relation-target>` mismatch
- **Preconditions:** item exists; relation predicate false.
- **Expected flow:** relation check failure.
- **Recipients:** actor only.
- **Expected outputs:** relation-specific failure class.
- **Expected failures:** terminal.
- **Expected state change:** none.

### Case F: `get <item>` blocked
- **Preconditions:** item resolvable; constraints fail.
- **Expected flow:** validation blocked.
- **Recipients:** actor only.
- **Expected outputs:** blocked/capacity denial.
- **Expected failures:** terminal.
- **Expected state change:** none.

### Case G: `help <topic>`
- **Preconditions:** help service available.
- **Expected flow:** resolve utility -> execute help retrieval -> render -> dispatch.
- **Recipients:** actor only.
- **Expected outputs:** topic help or unknown-topic guidance.
- **Expected failures:** unknown topic as help-class output.
- **Expected state change:** optional help-session bookkeeping.

### Case H: unknown input
- **Preconditions:** no direct intent mapping.
- **Expected flow:** unknown -> fallback chain -> terminal unknown.
- **Recipients:** actor only.
- **Expected outputs:** unknown response after chain exhaustion.
- **Expected failures:** terminal unknown class.
- **Expected state change:** none.

### Case I: ambiguous target (`get sword` with multiple matches)
- **Preconditions:** multiple candidates match one slot.
- **Expected flow:** target ambiguity -> terminal ambiguity.
- **Recipients:** actor only.
- **Expected outputs:** disambiguation request/failure.
- **Expected failures:** terminal ambiguity class.
- **Expected state change:** none.

## 14) Porting notes

### Must preserve
- **Observed behavior:** fallback chain runs before terminal unknown.
- **Observed behavior:** failure classes remain distinct and user-visible.
- **Observed behavior:** one semantic event drives multi-audience output.
- **Observed behavior:** relation constraints are validated before mutation.

### Safe to adapt
- **Suggested porting strategy:** parser internals and storage structures.
- **Suggested porting strategy:** styling/punctuation/color where strict-text mode is not required.
- **Suggested porting strategy:** transport protocol details if recipient semantics are preserved.

### Common pitfalls
- **Inferred behavior:** mutating before deterministic validation ordering creates transcript drift.
- **Inferred behavior:** per-audience semantic reinterpretation causes contradiction between recipient views.
- **Inferred behavior:** mixed concurrency without atomic boundaries leaks partial state.

### Minimal vs full-fidelity
- **Suggested porting strategy (minimal):** preserve taxonomy, fallback chain, deterministic validation order, and semantic-equivalent profile.
- **Suggested porting strategy (full-fidelity):** additionally preserve declared target-precedence order, recipient snapshot timing, delivery profile, strict-text families, and stable error envelope codes.

### Non-features / forbidden assumptions
- **Suggested porting strategy:** do not auto-execute every parseable phrase.
- **Suggested porting strategy:** do not collapse unknown/ambiguous/forbidden/invalid-context into one generic failure.
- **Suggested porting strategy:** do not emit success before mutation commits.

## 15) Open questions (requires maintainer confirmation)

1. **Question:** Which exact command families require strict-text compatibility?
   - **Interpretation A (Suggested porting strategy):** strict-text required only for core UX commands.
   - **Interpretation B (Suggested porting strategy):** strict-text required broadly for compatibility.
   - **Differentiating test:** compare semantic vs strict transcript suites by command family.
   - **Impact:** determines localization and wording freedom.

2. **Question:** Are legacy non-atomic execution exceptions permitted?
   - **Interpretation A (Inferred behavior):** no exceptions; all failures rollback.
   - **Interpretation B (Inferred behavior):** narrow documented exceptions exist.
   - **Differentiating test:** inject execution failures mid-action and verify post-state snapshots.
   - **Impact:** determines transaction implementation and recovery policy.

3. **Question:** Which delivery ordering profile is canonical for parity tests?
   - **Interpretation A (Suggested porting strategy):** strict actor->target->bystander ordering.
   - **Interpretation B (Suggested porting strategy):** coherent-batch ordering accepted.
   - **Differentiating test:** instrument recipient delivery ordering across multi-recipient actions.
   - **Impact:** determines transcript determinism requirements.
