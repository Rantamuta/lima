# Portable Behavior Spec Prompt (Pragmatic / Agent-Friendly)

Use this prompt when you want a fast but disciplined implementation-agnostic spec that another agent can execute with minimal overhead.

## Prompt

Create a plain-language behavioral spec for porting this MUD interaction model to another project where the implementer has no access to original internals.

### Main goal
Explain **what happens** and **in what order** from player input to final output, so the behavior can be rebuilt from the document alone.

### Keep it portable
- Prefer role names (parser, resolver, executor, renderer, dispatcher) over source-specific names.
- Focus on behavior and decisions, not code files.
- Make branch behavior explicit.

### Required sections

1. **System overview (short)**
   - What problem this pipeline solves.
   - Start/end points.

2. **Pipeline in ordered steps**
   - Input intake
   - Normalization
   - Interpretation
   - Target/context resolution
   - Validation/permission checks
   - Action execution
   - Message construction
   - Audience-specific delivery
   - Error/fallback handling

3. **Decision matrix**
   Include conditions and outcomes for:
   - valid interpretation,
   - unknown command/verb,
   - ambiguous target,
   - invalid target/context,
   - blocked action,
   - fallback path.

4. **Perspective messaging rules**
   - Single action intent -> actor/target/bystander outputs.
   - Pronoun/reflexive behavior expectations.
   - Visibility/filtering rules.

5. **Acceptance examples (8–10 cases)**
   Use common commands (look, look-at, take, take-from, help, unknown input, ambiguous input, invalid target).

   For each case provide:
   - preconditions,
   - expected flow steps,
   - expected outputs by audience,
   - expected failures (if any).

6. **Porting notes**
   - “Must preserve” behaviors.
   - “Safe to adapt” behaviors.
   - Common pitfalls.
   - Minimal version vs full-fidelity version.

7. **Open questions**
   - Any uncertain behavior,
   - two plausible interpretations each,
   - what needs maintainer confirmation.

### Style requirements
- Write in plain language.
- Use concise bullets/tables where helpful.
- Keep each rule testable.

### Classification requirement
Mark each non-trivial statement as one of:
- **Observed behavior**
- **Inferred behavior**
- **Suggested porting strategy**

### Done criteria
A good result lets another engineer implement a behaviorally equivalent system without reading the original source.
