# Portable Behavioral Spec: Command-to-Output Interaction Model

This document is an implementation-agnostic behavioral spec for porting a classic text-MUD interaction model into a new codebase where original internals are unavailable.

## 1) Evidence snapshot (short)

- **Evidence source:** Runtime-facing behavior in core interaction paths (command parsing outcomes, fallback behavior, action success/failure messaging, audience-specific message fanout).
  - **Confidence:** High for command outcomes and fallback behavior.
- **Evidence source:** User-visible command descriptions and verb usage patterns (single-word, preposition-based, multi-target actions).
  - **Confidence:** High for accepted input forms in common actions.
- **Evidence source:** Message composition behavior (pronoun/person shift, recipient-specific rendering, pluralization/article shaping).
  - **Confidence:** Medium-High for deterministic audience-specific transformation.
- **Evidence source:** Existing architecture/porting notes already written in this repo.
  - **Confidence:** Medium for intent-level interpretation; treated as supporting context, not primary runtime proof.

## 2) System overview (short)

- **Observed behavior:** The pipeline takes a player text input and attempts to interpret it as an action request in world context.
- **Observed behavior:** If interpretation succeeds, the system either performs state changes or emits explanatory failure output.
- **Observed behavior:** A single successful semantic action can produce different output strings for actor, target, and bystanders.
- **Inferred behavior:** The design goal is to preserve natural-language feel while still supporting deterministic fallback behavior.
- **Suggested porting strategy:** Preserve behavior ordering and recipient semantics first; preserve exact wording only where required by tests.

**Start point:** one player-issued input string.

**End point:** one of:
1. successful action + audience-specific output,
2. graceful failure + explanatory output,
3. unknown-input response after fallback path(s) are exhausted.

**Key assumptions:** world state is queryable at command time; recipient visibility can be determined; command processing is effectively single-request atomic at user-facing granularity.

## 3) Neutral glossary

- **Actor:** the entity that issued the input.
- **Primary target:** the direct object/entity acted on.
- **Secondary target:** an indirect object/entity used by or related to the action.
- **Bystander:** any nearby recipient not actor/primary target.
- **Resolver:** component that maps input text into a semantic action candidate.
- **Validator:** component that checks whether the candidate action is allowed/possible now.
- **Executor:** component that performs state mutation for a valid action.
- **Semantic event:** normalized action meaning before recipient-specific phrasing.
- **Renderer:** component that turns a semantic event into audience-specific strings.
- **Dispatcher:** component that delivers rendered strings to recipients.
- **Fallback path:** alternate interpretation attempt when primary resolution fails.

## 4) Pipeline in ordered steps

1. **Input intake**
   - **Observed behavior:** Receive raw input string from actor.
   - **Observed behavior:** Empty/invalid-at-source input does not proceed as a meaningful action.
   - **Mutation:** none.

2. **Normalization**
   - **Observed behavior:** Input is normalized enough for resolution (for example, token boundaries and candidate form preparation).
   - **Inferred behavior:** Normalization should not mutate world state.
   - **Mutation:** none.

3. **Interpretation (primary resolution)**
   - **Observed behavior:** Resolver attempts to match normalized input to an action pattern.
   - **Observed behavior:** Primary resolution can return: success, unknown-intent, semantic error, or forbidden/unavailable action.
   - **Mutation:** none.

4. **Target/context resolution**
   - **Observed behavior:** Candidate targets are resolved in current context (inventory, location, relation context like "from" or "with").
   - **Observed behavior:** Ambiguous or missing targets produce user-visible clarification/failure.
   - **Mutation:** none (except transient computation state).

5. **Validation/permission checks**
   - **Observed behavior:** System checks whether actor can perform the resolved action now.
   - **Observed behavior:** Validation failure yields a direct denial/error message.
   - **Inferred behavior:** Validation should be side-effect free whenever feasible.
   - **Mutation:** should be none.

6. **Action execution**
   - **Observed behavior:** On valid action, executor performs state changes (for example moving an item to actor inventory).
   - **Observed behavior:** Execution may still fail and then emits specific failure output.
   - **Mutation:** yes, on success (and occasionally partial changes in legacy edge cases).

7. **Message construction**
   - **Observed behavior:** A single semantic event is transformed into audience-specific strings.
   - **Observed behavior:** Rendering applies person/pronoun differences and noun phrase shaping (article/plural handling).
   - **Mutation:** none.

8. **Audience-specific delivery**
   - **Observed behavior:** Actor, target, and bystanders may receive different strings for the same event.
   - **Observed behavior:** Some variants intentionally suppress delivery to actor while still notifying others.
   - **Mutation:** none (delivery only).

9. **Error/fallback handling**
   - **Observed behavior:** If primary resolution yields unknown-intent, the system attempts a fallback interpretation path.
   - **Observed behavior:** If fallback also fails, the system emits generic unknown/nonsense messaging.
   - **Observed behavior:** Certain generic fallback errors are intentionally suppressed to avoid noisy output.
   - **Mutation:** none.

## 5) Decision matrix

| Condition | Next step | Recipients | Message ordering | Mutation outcome |
|---|---|---|---|---|
| **Observed behavior:** Primary interpretation succeeds and validation passes | Execute action -> render -> dispatch | actor + relevant non-actor recipients | actor-facing result typically emitted with or before environmental broadcast | state mutated on success |
| **Observed behavior:** Primary interpretation returns semantic error (malformed/invalid request) | Emit direct error; stop | actor only | single immediate error | no mutation |
| **Observed behavior:** Primary interpretation returns forbidden/unavailable | Emit denial; stop | actor only | single immediate denial | no mutation |
| **Observed behavior:** Primary interpretation returns unknown-intent | Run fallback interpretation attempt | none yet | no player-visible output before fallback completes | no mutation |
| **Observed behavior:** Fallback succeeds | Execute fallback action -> render -> dispatch | actor + relevant non-actor recipients | same as successful primary path | state mutated on success |
| **Observed behavior:** Fallback fails with generic navigation-style miss | Suppress overly-generic fallback text, continue to final unknown | actor only (final unknown) | no generic fallback spam before final unknown | no mutation |
| **Observed behavior:** Target is ambiguous | Ask for clarification / fail with disambiguation-style output | actor only | immediate | no mutation |
| **Observed behavior:** Target absent/invalid in context | Emit not-found/inapplicable output | actor only | immediate | no mutation |
| **Observed behavior:** Execution attempt fails (capacity/rule failure) | Emit specific execution failure | actor only (+ optional non-actor nonevent) | actor failure first; no success broadcast | no committed success mutation |

## 6) Perspective messaging rules

- **Observed behavior:** One semantic action fans out into multiple rendered views for actor, target, and bystanders.
- **Observed behavior:** Renderer applies perspective shifts so actor sees second-person framing while others see third-person framing.
- **Observed behavior:** Renderer supports reflexive/self-target output shaping.
- **Observed behavior:** Renderer handles article/plural agreement to produce grammatical aggregate references.
- **Observed behavior:** Dispatcher can exclude actor from a broadcast variant while still delivering to others.
- **Inferred behavior:** Delivery set should respect visibility/reachability constraints in the current context.
- **Suggested porting strategy:** Use one canonical semantic-event structure and derive all audience strings from that single event object.

**Output precision policy for this spec:** semantic-equivalent matching is required; byte-identical punctuation/whitespace is optional unless a test explicitly pins exact text.

## 7) Acceptance examples (9 canonical cases)

### Case A: `look`
- **Preconditions:** actor exists in a valid location.
- **Expected flow:** intake -> normalize -> resolve as context-inspection action -> validate -> execute (read-only) -> render actor view.
- **Recipients:** actor (primary), optional bystanders none.
- **Expected outputs:** actor sees location/room description.
- **Failures:** if actor context missing, system should recover to a safe context then provide inspect output.
- **State change:** none.

### Case B: `look at <target>` where target exists
- **Preconditions:** named target resolvable in scope.
- **Expected flow:** resolve target -> validate inspectability -> execute inspection -> render.
- **Recipients:** actor gets description; bystanders may receive lightweight observation message.
- **Expected outputs:** actor sees target long description or item description.
- **Failures:** none.
- **State change:** none.

### Case C: `look at <target>` where target missing
- **Preconditions:** target not in scope.
- **Expected flow:** resolve fails at target/context stage.
- **Recipients:** actor only.
- **Expected outputs:** explicit not-found message.
- **Failures:** handled as normal user-facing failure.
- **State change:** none.

### Case D: `get <item>` success
- **Preconditions:** item is obtainable and movable to actor.
- **Expected flow:** resolve obtain action -> validate -> execute move -> render success.
- **Recipients:** actor success line; bystanders see actor-takes-item style event.
- **Expected outputs:** actor gets taken confirmation; non-actors get third-person action line.
- **Failures:** none.
- **State change:** item relocates to actor inventory.

### Case E: `get <item> from <container/relation>` with relation mismatch
- **Preconditions:** item exists but not at requested relation.
- **Expected flow:** relation check fails during validation/execution precondition.
- **Recipients:** actor only.
- **Expected outputs:** relation-specific failure (e.g., item is not in/on requested place).
- **Failures:** handled locally; no broadcast success.
- **State change:** none.

### Case F: `get <item>` blocked by capacity/rules
- **Preconditions:** item present but actor cannot carry/take now.
- **Expected flow:** resolve -> validate/execute failure.
- **Recipients:** actor only.
- **Expected outputs:** load/rule failure explanation.
- **Failures:** explicit and immediate.
- **State change:** none.

### Case G: `help <topic>`
- **Preconditions:** help subsystem available.
- **Expected flow:** resolve as utility command -> execute help session start -> deliver paged/help output.
- **Recipients:** actor only.
- **Expected outputs:** topic help or menu/help prompt flow.
- **Failures:** unknown topic handled by help subsystem messaging.
- **State change:** at most session/help-read bookkeeping.

### Case H: Unknown intent input (non-action nonsense)
- **Preconditions:** input does not map to known intent.
- **Expected flow:** primary resolve unknown -> fallback attempt -> fallback fails -> final unknown response.
- **Recipients:** actor only.
- **Expected outputs:** nonsense/unknown-command style message.
- **Failures:** this is terminal failure path.
- **State change:** none.

### Case I: Ambiguous target (`get sword` with multiple matches)
- **Preconditions:** multiple candidate targets satisfy same phrase.
- **Expected flow:** resolve returns ambiguity class; no execution.
- **Recipients:** actor only.
- **Expected outputs:** disambiguation request or ambiguity failure.
- **Failures:** expected normal branch.
- **State change:** none.

## 8) Porting notes

### Must preserve
- **Observed behavior:** Unknown-intent path attempts fallback before final failure.
- **Observed behavior:** Validation failures are user-visible and do not silently succeed.
- **Observed behavior:** Successful actions can produce different actor/target/bystander strings from one semantic event.
- **Observed behavior:** Relation-sensitive actions (from/on/with) enforce context constraints.

### Safe to adapt
- **Suggested porting strategy:** Exact phrasing, punctuation, and text styling.
- **Suggested porting strategy:** Internal parser architecture, as long as externally visible behavior matches this spec.
- **Suggested porting strategy:** Error-text wording, if semantic class remains equivalent.

### Common pitfalls
- **Inferred behavior:** Executing mutation before all validations complete can create divergence.
- **Inferred behavior:** Broadcasting success text before confirming state change causes false positives.
- **Inferred behavior:** Treating all unknowns identically can lose useful distinction between malformed, forbidden, and unresolved inputs.

### Minimal version vs full-fidelity
- **Suggested porting strategy (minimal):** implement deterministic resolve/validate/execute pipeline + actor-only messaging + fallback unknown handling.
- **Suggested porting strategy (full-fidelity):** add multi-audience perspective rendering, article/plural grammar shaping, relation-aware validation, and ambiguity-specific handling.

### Non-features / forbidden assumptions
- **Suggested porting strategy:** Do not assume every parseable phrase should auto-execute; unresolved/forbidden branches must fail clearly.
- **Suggested porting strategy:** Do not assume one output string fits all recipients.
- **Suggested porting strategy:** Do not assume fallback errors should always be shown verbatim.

## 9) Open questions

1. **Question:** Should validation be strictly side-effect free in every action family?
   - **Plausible interpretation A (Inferred behavior):** validation is pure; all mutation occurs only in execute.
   - **Plausible interpretation B (Inferred behavior):** some legacy actions may perform minor side effects during feasibility checks.
   - **Differentiating test:** run repeated dry-style checks with identical world state and verify zero state delta before explicit execute.
   - **Maintainer confirmation needed:** yes.

2. **Question:** What is the exact audience ordering guarantee when both target and bystanders are present?
   - **Plausible interpretation A (Inferred behavior):** actor/target direct messages precede ambient bystander broadcast.
   - **Plausible interpretation B (Inferred behavior):** ordering is implementation-defined as long as all recipients receive one coherent event.
   - **Differentiating test:** instrument delivery timestamps across recipient categories for one multi-target event.
   - **Maintainer confirmation needed:** yes.

3. **Question:** Should semantic-equivalent output matching remain default for all regression tests?
   - **Plausible interpretation A (Suggested porting strategy):** semantic equivalence is sufficient except where exact text is contractually important.
   - **Plausible interpretation B (Suggested porting strategy):** key core actions require byte-identical output for compatibility expectations.
   - **Differentiating test:** run transcript tests in semantic mode and strict-text mode; compare compatibility outcomes.
   - **Maintainer confirmation needed:** yes.
