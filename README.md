# Proposal: Going beyond the `bot` label

Based on discussions with various LLM-based agents and their human operators on Bluesky, I want to propose a Lexicon to extend the value of self-declared `bot`-hood. Having a single binary automated/not automated flag is good for human legibility, but it's insufficient for discovery, moderation, and building and maintaining trust in an ecosystem with a mix of human-operated, LLM-operated and deterministically operated accounts.

The core issue with the binary `bot` label is that it doesn't distinguish between an account that posts every time it rains in Cork, an account which announces PRs and merges to a GitHub repo, and an account which likes to discuss ATproto internals. They are all automated, but the degree and purpose of automation differs enormously.

## `actor.automation.declaration` and `actor.automation.operates`

My proposal is simple: a self-declared label with a record describing the *kind* of automation the account has, provided in addition to the `bot` label. Most of the record properties are meant to be machine-readable and can aid in classification, labeling, filtering, and moderation, while an optional `purpose` field allows for a richer self-declaration of what the account is meant to do or be.

Importantly, the schema also includes a reference to an operator DID, allowing the automated account to declare on-protocol a user or organization which it belongs to, and to which questions/suggestions/moderation decisions should be directed.

To avoid several potential abuse scenarios, this proposal also includes a corresponding schema for *operator* declarations - a way for an account to claim the automated accounts it is the operator of. This way, clients can trivially verify the automated account's claim about its operator. From the other point of view, clients can easily enumerate and list all the automated accounts an account declares itself as operating.

Using the proposed labels has very clear use for efficient labeling and moderation. The binary `bot` label has little value for determining whether an account is acting as intended or if it's displaying suspicious behavior. An account declaring it's `interactionMode` as `proactive` and `humanSupervision` as `none` should probably be evaluated differently than a `broadcast-only` account with no human supervision. And if it starts behaving in a non-broadcast way, that's a clear signal that something has gone wrong.

The full proposed specification including schemas is available in [SPEC.md](SPEC.md).

The respective schemas are also available in [automation-declaration.json](automation-declaration.json) and [automation-operates.json](automation-operates.json), respectively.

## Disclosure

I'm not an ATproto person. I know fairly little about protocol internals, and I did a lot of research this morning to learn enough to get GPT-5.4 running in Copilot to generate a draft Lexicon.

This is a **v0 draft proposal**, not a ready-to-implement spec. My intention with publishing this is to have something to critique and discuss. Fork this repo, make changes, propose other ways of addressing the core issues I'm trying to resolve here!