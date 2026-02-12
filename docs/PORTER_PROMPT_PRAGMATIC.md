# Portable Behavior Spec Prompt (Pragmatic / Agent-Friendly)

Use this prompt when you want a fast but disciplined implementation-agnostic spec that another agent can execute with minimal overhead.

## Prompt

Create a plain-language behavioral spec for porting this MUD interaction model to another project where the implementer has no access to original internals.

### Main goal
Explain **what happens** and **in what order** from player input to final output, so the behavior can be rebuilt from the document alone.

### Keep it portable
- Prefer role names (parser, resolver, executor, renderer, dispatcher) over source-specific names.
- Avoid engine/repo jargon unless you define it in a neutral glossary.
- Focus on behavior and decisions, not code files.
- Make branch behavior explicit.

### Labeling + normative discipline
Mark each non-trivial statement as one of:
- **Observed behavior**
- **Inferred behavior**
- **Suggested porting strategy**

Rules:
- Observed may use MUST/SHOULD/MAY.
- Inferred and Suggested should avoid MUST.

### Required sections

1. **Evidence snapshot (short)**
   - Sources used (transcripts, visible strings/docs, structural hints).
   - Confidence notes.

2. **System overview (short)**
   - What problem this pipeline solves.
   - Start/end points and key assumptions.

3. **Neutral glossary**
   - Define core terms (actor, target, bystander, semantic event, etc.).

4. **Pipeline in ordered steps**
   - Input intake
   - Normalization
   - Interpretation
   - Target/context resolution
   - Validation/permission checks
   - Action execution
   - Message construction
   - Audience-specific delivery
   - Error/fallback handling

   For each step, briefly note ordering and whether it may mutate state.

5. **Decision matrix**
   Include conditions and outcomes for:
   - valid interpretation,
   - unknown command/verb,
   - ambiguous target,
   - invalid target/context,
   - blocked action,
   - fallback path.

   Include: recipients, message ordering, and mutation outcome.

6. **Perspective messaging rules**
   - Single semantic action -> actor/target/bystander outputs.
   - Pronoun/reflexive behavior expectations.
   - Visibility/filtering rules.
   - State whether output matching is exact text or semantic-equivalent.

7. **Acceptance examples (8–10 cases)**
   Use common commands (look, look-at, take, take-from, help, unknown input, ambiguous input, invalid target).

   For each case provide:
   - preconditions,
   - expected flow steps,
   - expected recipients,
   - expected outputs by audience,
   - expected failures (if any),
   - expected state change.

8. **Porting notes**
   - “Must preserve” behaviors.
   - “Safe to adapt” behaviors.
   - Common pitfalls.
   - Minimal version vs full-fidelity version.
   - Non-features/forbidden assumptions to avoid feature creep.

9. **Open questions**
   - Any uncertain behavior,
   - two plausible interpretations each,
   - one test that would differentiate interpretations,
   - what needs maintainer confirmation.

### Done criteria
A good result lets another engineer implement a behaviorally equivalent system without reading the original source.
