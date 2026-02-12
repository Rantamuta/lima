# Portable Behavior Spec Prompt (Strict / RFC-Style)

Use this prompt when you need a high-rigor, implementation-agnostic behavioral specification suitable for faithful porting to a different codebase.

## Prompt

You are producing a **portable behavioral specification** of a MUD command/interaction system for reimplementation in another codebase.

### Mission
Write a document that explains behavior in clear, implementation-agnostic language so a downstream porter can recreate equivalent behavior **without access to source internals**.

### Hard constraints
1. Do not rely on source object/class/function/file names in the specification body.
2. Describe **behavioral contracts and flow**, not code organization.
3. Use normative language:
   - **MUST** for required behavior,
   - **SHOULD** for recommended behavior,
   - **MAY** for optional behavior.
4. A competent engineer must be able to implement from your prose alone.
5. If evidence is incomplete, do not guess silently.

### Required output sections

#### 1) Scope and boundaries
- Define system boundary: what enters, what leaves.
- Define external assumptions (session model, world state availability, audience model).
- Define non-goals.

#### 2) Conceptual component model (role names only)
- Input intake stage
- Input normalization stage
- Intent resolution stage
- Context/target resolution stage
- Authorization/feasibility stage
- Execution stage
- Message realization stage
- Audience dispatch stage
- Error/fallback stage

For each stage, define:
- inputs,
- outputs,
- invariants,
- failure classes.

#### 3) End-to-end pipeline
Describe the exact lifecycle from raw player text to all emitted outputs, including branch points and ordering guarantees.

#### 4) Decision tables (mandatory)
Provide explicit tables for:
- interpretation success/failure classes,
- unknown-intent handling,
- ambiguous-target handling,
- invalid-context handling,
- authorization failure,
- fallback path invocation.

Each row must include:
- condition,
- next stage,
- actor-visible output,
- non-actor output,
- state mutation rule,
- termination/continuation rule.

#### 5) Perspective-aware messaging contract (mandatory)
Specify how one semantic action produces viewpoint-specific outputs:
- actor,
- primary target,
- bystanders,
- system/observer channels.

Include explicit rules for:
- person/pronoun transformation,
- reflexive/self-target forms,
- singular/plural agreement,
- recipient suppression/filtering,
- consistency across channels,
- ordering if multiple messages emit.

#### 6) Canonical acceptance transcripts (mandatory)
Provide 8–10 canonical scenarios (for example: look, look-at, take, take-from, help, unknown input, ambiguous input, invalid target).

For each scenario include:
- preconditions,
- normalized input form,
- expected stage-by-stage path,
- exact expected outputs per audience,
- expected state deltas,
- failure outputs (if applicable).

#### 7) Porting guidance
For each major stage:
- what is essential to preserve,
- what can vary safely,
- minimum-viable implementation choice,
- full-fidelity implementation choice,
- typical failure modes during porting.

#### 8) Ambiguities and confirmation needs
If behavior cannot be established confidently:
- mark **REQUIRES MAINTAINER CONFIRMATION**,
- provide two plausible interpretations,
- describe implementation impact of each.

### Mandatory labeling discipline
Every substantive claim must be labeled as exactly one of:
- **Observed behavior**
- **Inferred behavior**
- **Suggested porting strategy**

### Quality gates (must pass)
- [ ] No implementation-specific symbol names in normative sections.
- [ ] All major branches and failure modes covered.
- [ ] Perspective-message transformation rules are explicit and testable.
- [ ] Acceptance transcripts are concrete enough for regression testing.
- [ ] Ambiguities are explicit; no silent assumptions.

### Deliverable footer
End with:
1. Open ambiguities list,
2. Regression test checklist derived from transcripts,
3. “Minimum viable first milestone” for implementers.
