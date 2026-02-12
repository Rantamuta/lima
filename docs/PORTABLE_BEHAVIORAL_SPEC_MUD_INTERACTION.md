# Portable Behavioral Spec: Command-to-Output Interaction Model

This document defines a portable behavioral contract for a text-MUD interaction pipeline. It is written so an implementer can reproduce behavior without access to original internals.

## 1) Evidence snapshot (short)

- **Evidence source:** Runtime interaction paths and user-visible outcomes for command handling, fallback behavior, and action success/failure.
  - **Confidence:** High for branch outcomes and fallback order.
- **Evidence source:** User-visible interaction patterns for common command shapes (single token, preposition form, target + relation).
  - **Confidence:** High for input classes and recipient classes.
- **Evidence source:** Message rendering/dispatch behavior where one action yields different audience-specific phrasing.
  - **Confidence:** Medium-High for perspective transformation and fanout.
- **Evidence source:** Existing repository architecture notes used as context only.
  - **Confidence:** Medium for intent framing; non-authoritative for runtime facts.

## 2) System overview (short)

- **Observed behavior:** The pipeline accepts a player-issued text input and attempts to interpret it as an action within current world context.
- **Observed behavior:** A successful interpretation may mutate world state and then emit recipient-specific output.
- **Observed behavior:** A failed interpretation emits an explicit failure class (for example unknown, ambiguous, forbidden, invalid context).
- **Inferred behavior:** The design emphasizes natural-language feel while preserving deterministic branch handling.
- **Suggested porting strategy:** Preserve branch order, recipient semantics, and failure taxonomy before preserving exact text.

**Start point:** one actor input string.

**End point:** exactly one terminal class:
1. success (with audience delivery),
2. handled failure (with explanatory output),
3. unknown after fallback exhaustion.

**Execution model assumption:** command handling should appear atomic to the actor at command granularity.

## 3) Neutral glossary

- **Actor:** the entity issuing input.
- **Primary target:** direct object/entity of the action.
- **Secondary target:** indirect object/entity (for example relation/object used with the action).
- **Bystander:** nearby eligible recipient that is neither actor nor primary target.
- **Resolver:** stage that maps normalized input into a candidate intent.
- **Target resolver:** stage that binds textual references to concrete world entities.
- **Validator:** stage that checks feasibility/permissions for the bound action.
- **Executor:** stage that applies state mutations for an approved action.
- **Semantic event:** canonical action representation before audience-specific wording.
- **Renderer:** stage that transforms one semantic event into audience-specific messages.
- **Dispatcher:** stage that delivers rendered messages to eligible recipients.
- **Fallback path:** secondary interpretation attempt used only after unknown-intent from primary resolution.
- **Eligible recipient:** recipient approved by external visibility/reachability policy.

## 4) Resolution-result taxonomy

- **Success:** intent, targets, and constraints are satisfied; execution may proceed.
- **Unknown intent:** no intent mapping exists for input; fallback handling applies.
- **Semantic error:** intent class is recognized but input form is incomplete/malformed.
- **Ambiguous target:** more than one candidate target matches required role.
- **Invalid context/target:** target exists but does not satisfy relation/context constraints.
- **Forbidden/blocked:** action is recognized but disallowed by permission/capacity/rules.

## 5) Pipeline with stage invariants (ordered)

> **Global invariant (Observed behavior):** pipeline order MUST be intake -> normalization -> interpretation -> target resolution -> validation -> execution -> rendering -> dispatch, with fallback only on unknown intent.

1. **Input intake**
   - **Observed behavior:** system receives raw actor input.
   - **Observed behavior:** empty or non-actionable input terminates early with no action execution.
   - **Invariants:**
     - MUST run first.
     - MUST NOT mutate world state.
     - MUST NOT emit ambient bystander output.

2. **Normalization**
   - **Observed behavior:** input is transformed into a parse-ready form.
   - **Inferred behavior:** normalization should be deterministic for equivalent textual inputs.
   - **Invariants:**
     - MUST run after intake and before interpretation.
     - MUST NOT mutate world state.
     - MUST NOT emit player-visible output.

3. **Interpretation (primary resolution)**
   - **Observed behavior:** resolver classifies input into success/unknown/semantic error/forbidden-like classes.
   - **Invariants:**
     - MUST run before target resolution.
     - MUST NOT mutate world state.
     - MUST emit output only for terminal error classes that do not require fallback.

4. **Target/context resolution**
   - **Observed behavior:** target phrases are bound against actor context (inventory, locale, relation scope).
   - **Observed behavior:** missing/ambiguous targets produce actor-visible clarification/failure.
   - **Invariants:**
     - MUST run after interpretation success class.
     - MUST NOT mutate persistent world state.
     - MAY emit actor-visible disambiguation/invalid-target output.

5. **Validation/permission checks**
   - **Observed behavior:** feasibility checks evaluate permission, capacity, and rule constraints.
   - **Observed behavior:** validation failure is user-visible and terminal for current command.
   - **Inferred behavior:** validation should remain side-effect free except explicitly documented legacy exceptions.
   - **Invariants:**
     - MUST run before execution.
     - MUST NOT emit success output.
     - SHOULD NOT mutate state; if mutation exists, it should be explicitly documented.

6. **Action execution**
   - **Observed behavior:** execution applies mutations only after prior stages pass.
   - **Observed behavior:** execution can still fail and emit explicit failure output.
   - **Invariants:**
     - MUST run after successful validation.
     - MAY mutate world state.
     - On failure, state MUST remain unchanged unless a legacy non-atomic exception is explicitly documented.

7. **Message construction**
   - **Observed behavior:** one semantic event is rendered into audience-specific text variants.
   - **Observed behavior:** person/pronoun/reflexive forms and noun phrase shaping vary by recipient perspective.
   - **Invariants:**
     - MUST run after execution success.
     - MUST NOT perform additional semantic reinterpretation per audience.
     - MUST derive all audience strings from one canonical semantic event.

8. **Audience-specific delivery**
   - **Observed behavior:** actor/target/bystander outputs may differ while describing one event.
   - **Observed behavior:** some message forms intentionally exclude actor from ambient text.
   - **Invariants:**
     - MUST run after rendering.
     - MUST deliver only to eligible recipients.
     - MUST NOT mutate world state.

9. **Error/fallback handling**
   - **Observed behavior:** fallback is attempted for unknown intent before final unknown output.
   - **Observed behavior:** if fallback succeeds, normal pipeline continues; if fallback fails, unknown output is terminal.
   - **Invariants:**
     - MUST trigger only from unknown-intent class.
     - MUST complete before final unknown output is emitted.
     - MUST NOT emit partial player-visible unknown output before fallback completion.

## 6) Decision matrix

| Condition | Next stage | Recipients | Message ordering | Mutation outcome | Termination rule |
|---|---|---|---|---|---|
| Valid interpretation + valid targets + validation pass | Execute -> render -> dispatch | Actor + eligible others | Actor/target direct outputs precede or match one coherent event boundary for bystanders | Mutate on success | Halt after dispatch |
| Unknown intent from primary interpretation | Fallback interpretation | Actor only until fallback resolves | No final unknown text before fallback complete | None before fallback resolution | Continue |
| Unknown intent after fallback exhaustion | Terminal unknown response | Actor only | Single terminal unknown output | None | Halt |
| Ambiguous target | Terminal ambiguity response | Actor only | Ambiguity response only | None | Halt |
| Invalid target/context relation | Terminal invalid-context response | Actor only | Invalid-context response only | None | Halt |
| Forbidden/blocked action | Terminal blocked response | Actor only | Blocked response only | None | Halt |
| Execution failure after validation pass | Terminal execution-failure response | Actor (plus optional observers if policy requires failure visibility) | Failure output; no success broadcast | No mutation unless documented legacy exception | Halt |

## 7) Perspective messaging rules

- **Observed behavior:** one semantic action can produce actor-, target-, and bystander-specific phrasing.
- **Observed behavior:** perspective rendering includes person shift and pronoun/reflexive agreement.
- **Inferred behavior:** article/plural shaping should remain deterministic per noun phrase.
- **Suggested porting strategy:** define one canonical semantic-event schema and render all audiences from it.

### Required constraints

- **Observed behavior:** all audience outputs MUST be derived from a single canonical semantic event.
- **Observed behavior:** audience outputs MUST NOT be generated from separate semantic interpretation branches.
- **Observed behavior:** dispatcher MUST filter recipients using eligibility policy computed before delivery.
- **Suggested porting strategy:** keep eligibility criteria external to this document, but enforce that only eligible recipients receive output.

### Output precision policy

- **Suggested porting strategy:** default regression should be semantic-equivalent (intent, recipients, ordering, mutation class).
- **Suggested porting strategy:** exact-text matching may be required for compatibility-sensitive commands and should be declared per test suite.

## 8) Acceptance examples (canonical)

> Unless stated otherwise, outputs are semantic expectations (not byte-identical), with required recipient class and ordering.

### Case A: `look`
- **Preconditions:** actor is in a valid locale.
- **Expected flow:** intake -> normalize -> interpret success -> resolve context -> validate -> execute(read-only inspect) -> render -> dispatch.
- **Recipients:** actor only.
- **Expected outputs:** locale description + visible entities/exits summary.
- **Expected failures:** none.
- **Expected state change:** none.

### Case B: `look at <target>` success
- **Preconditions:** target is visible/resolvable.
- **Expected flow:** interpret inspect intent -> resolve target -> validate -> execute inspect -> render actor-focused detail.
- **Recipients:** actor required; optional bystander notification depends on policy.
- **Expected outputs:** target detail to actor.
- **Expected failures:** none.
- **Expected state change:** none.

### Case C: `look at <target>` missing target
- **Preconditions:** target phrase cannot resolve.
- **Expected flow:** resolve target fails -> terminal invalid-target response.
- **Recipients:** actor only.
- **Expected outputs:** explicit not-found/invalid-target message.
- **Expected failures:** terminal for command.
- **Expected state change:** none.

### Case D: `get <item>` success
- **Preconditions:** item is present, reachable, and carryable.
- **Expected flow:** resolve obtain intent -> resolve target -> validate carry/take rules -> execute transfer -> render -> dispatch.
- **Recipients:** actor + eligible bystanders.
- **Expected outputs:** actor receives second-person success; bystanders receive third-person action event.
- **Expected failures:** none.
- **Expected state change:** item relocates to actor inventory.

### Case E: `get <item> from <relation-target>` relation mismatch
- **Preconditions:** item exists but relation predicate is false.
- **Expected flow:** resolve relation -> validation/context check fails -> terminal invalid-context output.
- **Recipients:** actor only.
- **Expected outputs:** relation-specific failure.
- **Expected failures:** terminal.
- **Expected state change:** none.

### Case F: `get <item>` blocked by capacity/rules
- **Preconditions:** item is resolvable but carry/take constraints fail.
- **Expected flow:** validate blocked -> terminal blocked output.
- **Recipients:** actor only.
- **Expected outputs:** capacity/rule denial.
- **Expected failures:** terminal.
- **Expected state change:** none.

### Case G: `help <topic>`
- **Preconditions:** help subsystem available.
- **Expected flow:** resolve utility intent -> execute help retrieval/session -> render -> dispatch to actor.
- **Recipients:** actor only.
- **Expected outputs:** topic help, or unknown-topic/help-menu guidance.
- **Expected failures:** unknown topic handled within help response class.
- **Expected state change:** optional per-session help bookkeeping only.

### Case H: unknown input
- **Preconditions:** no primary intent mapping.
- **Expected flow:** primary unknown -> fallback attempt -> fallback failure -> terminal unknown.
- **Recipients:** actor only.
- **Expected outputs:** unknown/nonsense response after fallback exhaustion.
- **Expected failures:** terminal unknown class.
- **Expected state change:** none.

### Case I: ambiguous target (`get sword` with multiple matches)
- **Preconditions:** multiple valid candidates for one target phrase.
- **Expected flow:** target resolution yields ambiguity -> terminal ambiguity response.
- **Recipients:** actor only.
- **Expected outputs:** disambiguation request or ambiguity failure.
- **Expected failures:** terminal ambiguity class.
- **Expected state change:** none.

## 9) Porting notes

### Must preserve
- **Observed behavior:** fallback attempt precedes final unknown output.
- **Observed behavior:** success and failure classes are distinct and user-visible.
- **Observed behavior:** one action event fans out into perspective-specific outputs.
- **Observed behavior:** relation-sensitive actions enforce relation predicates before mutation.

### Safe to adapt
- **Suggested porting strategy:** internal parser architecture and data structures.
- **Suggested porting strategy:** exact punctuation/style/color details unless compatibility tests require strict text.
- **Suggested porting strategy:** transport/format layer choices, if recipient semantics remain intact.

### Common pitfalls
- **Inferred behavior:** mutating state before final validation can create silent divergence.
- **Inferred behavior:** generating audience text from separate semantic paths can cause inconsistent narratives.
- **Inferred behavior:** collapsing unknown/ambiguous/blocked into one error class removes useful feedback and breaks transcript parity.

### Minimal version vs full-fidelity
- **Suggested porting strategy (minimal):** preserve taxonomy + fallback order + actor-only success/failure with deterministic mutation rules.
- **Suggested porting strategy (full-fidelity):** preserve full multi-audience rendering, perspective grammar, relation predicates, and ambiguity-specific dialog.

### Non-features / forbidden assumptions
- **Suggested porting strategy:** do not auto-execute every parseable phrase.
- **Suggested porting strategy:** do not assume one output string serves all audiences.
- **Suggested porting strategy:** do not treat fallback messages as always safe to echo verbatim.

## 10) Open questions (requires maintainer confirmation)

1. **Question:** Are validation stages strictly side-effect free for all action families?
   - **Interpretation A (Inferred behavior):** validation is pure; all persistent mutation occurs in execution.
   - **Interpretation B (Inferred behavior):** limited legacy checks may mutate transient or persistent state.
   - **Differentiating test:** run repeated validation-only probes under fixed state; compare state snapshots before/after.
   - **Impact:** determines whether validator must be implemented as pure function in the target.

2. **Question:** What exact ordering is guaranteed for actor/target/bystander delivery?
   - **Interpretation A (Inferred behavior):** actor/target direct deliveries occur before ambient bystander delivery.
   - **Interpretation B (Inferred behavior):** ordering is implementation-defined if event coherence is preserved.
   - **Differentiating test:** instrument timestamped recipient deliveries for one multi-recipient action.
   - **Impact:** affects deterministic transcript assertions and race handling.

3. **Question:** Which commands require byte-identical text compatibility vs semantic equivalence?
   - **Interpretation A (Suggested porting strategy):** semantic equivalence is sufficient for most commands.
   - **Interpretation B (Suggested porting strategy):** a core compatibility subset needs exact text matching.
   - **Differentiating test:** run the same transcript suite in semantic mode and strict-text mode; compare pass/fail deltas.
   - **Impact:** defines regression policy strictness and localization freedom.
