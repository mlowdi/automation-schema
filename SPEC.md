# Automation Disclosure and Operator Attestation for AT Protocol Accounts

**Status:** Draft / exploratory  
**Namespace:** `one.mlf.actor.automation.*`  
**Author:** Martin Fällman (with Copilot/GPT-5.4)  

## 1. Abstract

This document proposes a paired disclosure model for automated AT Protocol accounts.

The proposal introduces two custom record types:

- `one.mlf.actor.automation.declaration`
- `one.mlf.actor.automation.operates`

The first is published by an automated account and describes the nature of its automation, optionally naming an operator DID. The second is published by an operator account and attests that it claims to operate a specific automated account.

Taken together, these records provide a simple bilateral validation model:

1. the automated account claims an operator
2. the operator account claims the automated account
3. clients and services may treat the relationship as valid only if both claims agree

This is intended to complement, not replace, the existing coarse `bot` self-label used by Bluesky.

## 2. Background

Bluesky currently supports a global `bot` self-label to mark accounts as automated. That label is useful as a coarse disclosure bit, but it does not distinguish between different kinds of automation such as scheduled broadcasters, trackers, or conversational agents.

AT Protocol already provides the appropriate extension mechanism for richer metadata: custom lexicons published under a domain-controlled reverse-DNS namespace. This proposal therefore does **not** attempt to extend or reinterpret Bluesky’s global `bot` label. Instead, it defines a sidecar disclosure model that can coexist with current Bluesky behavior.

## 3. Goals

This proposal has four goals.

First, it should let an automated account describe **what kind of automation it is** in a structured form that is both human-readable and machine-usable.

Second, it should let an operator account make a reciprocal claim that it **operates a specific automated account**.

Third, it should make abuse-resistant validation cheap. A bot’s operator claim should not be trusted merely because the bot says so; it should be possible to verify that the named operator account also claims the bot.

Fourth, it should remain small and durable. The design should avoid overfitting to current discourse around AI systems, model internals, or social relationships that are likely to be contentious, unstable, or impossible to standardize.

## 4. Non-goals

This proposal does not attempt to standardize:

- enforcement policy
- moderation policy
- legal or organizational accountability
- the internals of an automated system
- a complete ontology of human/agent relationships

This proposal is also not intended to replace the existing global `bot` self-label. The `bot` label remains the coarse compatibility layer for clients and services that only understand current Bluesky behavior.

## 5. Naming and Namespace

This proposal uses the namespace:

- `one.mlf.actor.automation.declaration`
- `one.mlf.actor.automation.operates`

Grouping both records under the shared `.actor.automation.*` prefix is intentional: they are separate records, but they represent one conceptual feature family.

The leaf name `declaration` is used for the automated account’s singleton disclosure record. The leaf name `operates` is used for the operator-side attestation collection.

The naming explicitly avoids overloading the existing term `bot`, and also avoids adjective-style schema names such as `automated`, which are less clear as record types.

## 6. Identity Model

All identity linkage in this proposal is DID-based.

Handles are human-readable aliases and may change; DIDs are the correct stable pointer for long-lived operator relationships. Any field that names another account in this proposal therefore uses `format: "did"`.

## 7. Record 1: `one.mlf.actor.automation.declaration`

### 7.1 Purpose

This record is published by an automated account.

It declares structured automation metadata for that account, complementing the existing global `bot` self-label with richer disclosure that can be used by clients, discovery tools, and moderation systems.

### 7.2 Cardinality

This is a singleton account-level record.

It uses `key: "literal:self"` because it is intended to behave like a profile-style declaration: one current disclosure per repository.

### 7.3 Semantics

A declaration record may contain:

- a machine-readable automation class
- an optional operator DID
- an optional human-readable purpose string
- optional behavioral hints such as interaction mode and supervision level

The `operator` field, when present, names the DID expected to publish a reciprocal `one.mlf.actor.automation.operates` record for validation.

### 7.4 Draft schema

```json
{
  "lexicon": 1,
  "id": "one.mlf.actor.automation.declaration",
  "defs": {
    "main": {
      "type": "record",
      "key": "literal:self",
      "description": "Structured automation disclosure for an account. Intended to complement the coarse global bot self-label with richer metadata for clients, discovery, moderation, and cross-checking against operator attestations.",
      "record": {
        "type": "object",
        "required": [
          "class"
        ],
        "properties": {
          "class": {
            "type": "string",
            "description": "Primary category of automation for this account.",
            "knownValues": [
              "broadcast",
              "tracker",
              "notifier",
              "conversational",
              "assistant",
              "creative",
              "hybrid"
            ],
            "maxLength": 64
          },
          "operator": {
            "type": "string",
            "format": "did",
            "description": "Optional DID of the account claimed to operate this automated account."
          },
          "purpose": {
            "type": "string",
            "description": "Optional plain-language description of the account's purpose and expected behavior.",
            "maxLength": 1000,
            "maxGraphemes": 300
          },
          "interactionMode": {
            "type": "string",
            "description": "How this automated account interacts with other users.",
            "knownValues": [
              "none",
              "broadcast-only",
              "mention-reply",
              "thread-reply",
              "proactive"
            ],
            "maxLength": 64
          },
          "humanSupervision": {
            "type": "string",
            "description": "Whether and how a human reviews or supervises the account's behavior.",
            "knownValues": [
              "none",
              "reviewed",
              "mixed"
            ],
            "maxLength": 64
          },
          "createdAt": {
            "type": "string",
            "format": "datetime",
            "description": "RFC 3339 timestamp indicating when this automation declaration was first created."
          },
          "updatedAt": {
            "type": "string",
            "format": "datetime",
            "description": "RFC 3339 timestamp indicating when this automation declaration was last updated."
          }
        }
      }
    }
  }
}
```

## 8. Record 2: `one.mlf.actor.automation.operates`

### 8.1 Purpose

This record is published by an operator account.

It is an operator-side attestation that the authoring account claims to operate a specific automated account. Its main function is to provide the reciprocal half of a bilateral validation model.

### 8.2 Why this is a collection, not an array

An early design option was a singleton operator record containing an array of operated DIDs.

That design is workable but inferior.

A per-bot record collection is cleaner because it:

- avoids rewriting one large array whenever a single relationship changes
- supports direct fetches for a specific `(operator DID, bot DID)` pair
- makes revocation trivial by deleting one record
- supports clearer indexing and audit history
- scales better for operators who run many automated accounts

### 8.3 Cardinality and rkey convention

This is a multi-record collection.

The collection uses `key: "any"`.

By convention, the record key **SHOULD** equal the `subject` DID named in the record body. This is not something Lexicon can enforce directly, so it is a specification convention rather than a schema constraint. The convention enables direct lookups of the form:

```text
at://<operator-did>/one.mlf.actor.automation.operates/<bot-did>
```

### 8.4 Why there is no `role` field

This proposal intentionally omits a `role` field.

A role vocabulary appears attractive at first, but quickly forces unstable social or organizational semantics into schema. That creates exactly the kind of brittle ontology that is difficult to evolve once published.

For the purposes of bilateral attestation, the only statement that matters is:

> “this account claims operator control of that automated account”

If more nuance is needed, the record includes an optional human-readable `description` field instead of a controlled `role` vocabulary.

### 8.5 Draft schema

```json
{
  "lexicon": 1,
  "id": "one.mlf.actor.automation.operates",
  "defs": {
    "main": {
      "type": "record",
      "key": "any",
      "description": "Operator-side attestation that the authoring account claims to operate a specific automated account. Intended to be cross-checked against the subject account's automation declaration.",
      "record": {
        "type": "object",
        "required": [
          "subject"
        ],
        "properties": {
          "subject": {
            "type": "string",
            "format": "did",
            "description": "DID of the automated account this account claims to operate."
          },
          "description": {
            "type": "string",
            "description": "Optional human-readable description of the operated account, such as its purpose, brand relationship, or expected behavior.",
            "maxLength": 1000,
            "maxGraphemes": 300
          },
          "createdAt": {
            "type": "string",
            "format": "datetime",
            "description": "Optional RFC 3339 timestamp indicating when this operator attestation was first created."
          },
          "updatedAt": {
            "type": "string",
            "format": "datetime",
            "description": "Optional RFC 3339 timestamp indicating when this operator attestation was last updated."
          }
        }
      }
    }
  }
}
```

## 9. Bilateral Validation Model

The core design decision in this proposal is that operator linkage is **not** trusted as a unilateral claim.

A bot saying “my operator is DID X” is not sufficient.

Instead, a consumer may validate operator linkage using the following algorithm:

1. fetch the automated account’s `one.mlf.actor.automation.declaration` record
2. read its `operator` field
3. fetch `at://<operator-did>/one.mlf.actor.automation.operates/<bot-did>`
4. confirm that the reciprocal record exists and that its `subject` matches the bot DID
5. only then treat the operator linkage as valid

This is intentionally simple and cheap. It requires at most one extra direct fetch after reading the bot’s declaration.

## 10. Abuse Resistance

This bilateral model is designed to mitigate a straightforward abuse case.

A malicious swarm of automated accounts could otherwise claim that some target DID is their operator, then behave abusively and attempt to transfer reputational or policy consequences onto that target.

Under this proposal, that attack fails by default. A bot’s operator claim should not be treated as valid unless the named operator account also publishes the matching reciprocal attestation.

This is the primary rationale for separating bot-side disclosure from operator-side attestation instead of relying on a single `operator` field alone.

## 11. Operator Chains

This proposal allows, but does not require, operator chains.

An automated account may name another account as its operator, and that operator account may itself publish an automation declaration. This makes it possible to model cases such as:

- an automated account operated by an organizational service account
- a service account operated by another automation layer
- more complex delegation or orchestration patterns

This specification does **not** attempt to standardize chain-resolution policy.

Consumers **MAY** impose their own policy, including a requirement that an operator chain eventually terminates at an account that does **not** publish `one.mlf.actor.automation.declaration`.

The important point is that chain termination is verifier policy, not schema law.

## 12. Discovery and UI

These records are intended to support straightforward UI patterns.

Examples include:

- “this account is automated” disclosure cards
- “this account claims operator X”
- “bots operated by this account”
- richer bot badges derived from `class`
- moderation and discovery filters based on automation category or behavior

The operator-side `description` field is especially useful for a list view. For example, an organization account could publish multiple `operates` records and describe each one in plain language:

- “weekly top recipes bot”
- “breaking news alert feed”
- “sports scores tracker”

This allows clients to build useful operator pages without inventing further schema.

## 13. Relationship to Bluesky’s `bot` Self-label

This proposal is complementary to the current Bluesky `bot` self-label.

The recommended posture is:

- use Bluesky’s global `bot` self-label for baseline compatibility
- use `one.mlf.actor.automation.declaration` for richer structured disclosure
- use `one.mlf.actor.automation.operates` for reciprocal operator attestation

That division of labor keeps the proposal aligned with current Bluesky behavior while avoiding the need to overload labels with richer semantics than they were designed to carry.

## 14. Compatibility and Evolution

This proposal is intentionally conservative.

AT Protocol lexicons are difficult to change compatibly once published. Tightening or changing constraints in-place can cause old and new software to reject each other’s data; incompatible changes should instead go under a new NSID. For that reason, this design keeps the schema small and avoids committing to unstable vocabularies where possible.

In particular:

- the operator-side record avoids a `role` taxonomy
- chain resolution is left to consumers
- baseline compatibility is delegated to the existing `bot` self-label

## 15. Example Records

### 15.1 Automated account declaration

```json
{
  "$type": "one.mlf.actor.automation.declaration",
  "class": "conversational",
  "operator": "did:plc:OPERATOR_DID",
  "purpose": "Conversational bot account operated for experimentation and social interaction on the ATmosphere network.",
  "interactionMode": "mention-reply",
  "humanSupervision": "mixed",
  "createdAt": "2026-03-27T14:25:00+01:00",
  "updatedAt": "2026-03-27T14:25:00+01:00"
}
```

### 15.2 Operator attestation

Record URI convention:

```text
at://did:plc:OPERATOR/one.mlf.actor.automation.operates/did:plc:BOT_DID
```

Record body:

```json
{
  "$type": "one.mlf.actor.automation.operates",
  "subject": "did:plc:BOT_DID",
  "description": "Conversational bot account focused on distributed systems and protocol design.",
  "createdAt": "2026-03-27T14:25:00+01:00",
  "updatedAt": "2026-03-27T14:25:00+01:00"
}
```

## 16. Open Questions

The following questions are intentionally left open for community discussion:

1. Should `class` remain a small controlled vocabulary?
2. Should clients surface an explicit “validated operator” state when bilateral confirmation succeeds?
3. Should there eventually be a standard way to represent operator chains and chain-resolution policies?
4. Should a future version define an optional service or feed for bulk discovery of operator relationships?
5. Should any part of this proposal eventually move from custom schema into broader ecosystem convention?

## 17. References

- AT Protocol — Lexicon: https://atproto.com/specs/lexicon
- AT Protocol — Record Key: https://atproto.com/specs/record-key
- AT Protocol — DID: https://atproto.com/specs/did
- AT Protocol — Lexicons guide: https://atproto.com/guides/lexicon
- Bluesky — Custom Schemas: https://docs.bsky.app/docs/advanced-guides/custom-schemas
- Bluesky — Labels and moderation: https://docs.bsky.app/docs/advanced-guides/moderation

## 18. Summary

This proposal defines a paired model for automation disclosure:

- `one.mlf.actor.automation.declaration` lets an automated account describe itself
- `one.mlf.actor.automation.operates` lets an operator account reciprocally attest control

The central design choice is bilateral validation. A bot’s operator claim is not considered trustworthy on its own; it becomes useful when it can be cross-checked against an operator-published attestation.

That design keeps the schema small, useful, abuse-resistant, and compatible with the way Bluesky and atproto already separate coarse self-labels from richer extension data.
