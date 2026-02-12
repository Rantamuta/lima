# Portable Behavior Spec Prompt (Strict / RFC-Style)

Use this prompt when you need a high-rigor, implementation-agnostic behavioral specification suitable for faithful porting to a different codebase.

## Prompt

You are producing a **portable behavioral specification** of a MUD command/interaction system for reimplementation in another codebase.

### Mission
Write a document that explains behavior in clear, implementation-agnostic language so a downstream porter can recreate equivalent behavior **without access to source internals**.

### Hard constraints
1. Do not rely on source object/class/function/file names in the specification body.
2. Treat implementation-specific references broadly: avoid engine-specific nouns, repository layout terms, and legacy jargon as normative anchors unless first normalized in a neutral glossary.
3. Describe **behavioral contracts and flow**, not code organization.
4. Use normative language:
   - **MUST** for required behavior,
   - **SHOULD** for recommended behavior,
   - **MAY** for optional behavior.
5. A competent engineer must be able to implement from your prose alone.
6. If evidence is incomplete, do not guess silently.

### Epistemic and normative discipline (mandatory)
Every substantive claim must be labeled as exactly one of:
- **Observed behavior**
- **Inferred behavior**
- **Suggested porting strategy**

Normative keyword restrictions:
- **Observed behavior** MAY use MUST/SHOULD/MAY.
- **Inferred behavior** MUST NOT use MUST (SHOULD/MAY allowed).
- **Suggested porting strategy** MUST NOT use MUST (SHOULD/MAY allowed).

### Required output sections

#### 0) Evidence inventory (required before the spec body)
Provide an evidence table with:
- evidence source type (runtime transcript, user-visible strings/help text, structural hints),
- evidence description,
- confidence level (High/Medium/Low),
- scope covered.

#### 1) Scope and boundaries
- Define system boundary: what enters, what leaves.
- Define external assumptions (session model, world state availability, audience model).
- Define non-goals.
- Declare boundary choices (transport/markup/localization/encoding) when relevant.

#### 2) Neutral glossary
Define neutral terms used throughout (for example: actor, primary target, indirect target, bystander, inventory, container, locale, session, semantic event).

#### 3) Conceptual component model (role names only)
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
- failure classes,
- ordering constraints,
- mutation policy (pure/check-only vs mutating),
- idempotence expectations,
- output emission allowance.

#### 4) End-to-end pipeline
Describe the exact lifecycle from raw player text to all emitted outputs, including branch points and ordering guarantees.

#### 5) Decision tables (mandatory)
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
- message count by audience,
- message ordering rule,
- state mutation guarantee,
- termination/continuation rule.

Tables must avoid implicit fallthrough and cover all defined failure classes.

#### 6) Perspective-aware messaging contract (mandatory)
Specify one canonical **semantic event** representation. All viewpoint outputs MUST derive from that event.

Define deterministic transformation rules for:
- person/pronoun shift,
- reflexive/self-target forms,
- singular/plural agreement,
- article/determiner policy,
- recipient suppression/filtering,
- consistency across channels,
- ordering if multiple messages emit.

State output precision policy explicitly:
- byte-identical required, or
- semantic-token equivalence with formatting tolerance.

#### 7) Canonical acceptance transcripts (mandatory)
Provide 8–10 canonical scenarios (for example: look, look-at, take, take-from, help, unknown input, ambiguous input, invalid target).

For each scenario include:
- preconditions,
- normalized input form,
- expected stage-by-stage path,
- expected recipients,
- expected outputs (per stated precision policy),
- expected state deltas,
- failure outputs (if applicable).

All transcripts must be consistent with decision tables and stage invariants.

#### 8) Porting guidance
For each major stage:
- what is essential to preserve,
- what can vary safely,
- minimum-viable implementation choice,
- full-fidelity implementation choice,
- typical failure modes during porting.

Include explicit “negative space”:
- forbidden interpretations,
- non-features that must not be added by default.

#### 9) Ambiguities and confirmation needs
If behavior cannot be established confidently:
- mark **REQUIRES MAINTAINER CONFIRMATION**,
- provide two plausible interpretations,
- describe implementation impact of each,
- provide at least one acceptance test that would distinguish them.

### Quality gates (must pass)
- [ ] No implementation-specific symbols or unnormalized legacy jargon in normative sections.
- [ ] No contradiction between evidence labels and normative claims.
- [ ] Inferred/Suggested claims do not use MUST.
- [ ] All major branches and failure modes covered.
- [ ] Stage ordering and mutation guarantees are explicit.
- [ ] Perspective-message transformation rules are explicit and testable.
- [ ] Decision tables include audience message count/order and mutation guarantees.
- [ ] Acceptance transcripts align with decision tables and stage invariants.
- [ ] Ambiguities are explicit and include differentiating tests.

### Deliverable footer
End with:
1. Open ambiguities list,
2. Regression test checklist derived from transcripts,
3. “Minimum viable first milestone” for implementers.
