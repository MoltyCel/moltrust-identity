---
name: moltrust-identity
version: 1.0.0
description: "Keyless agent identity and trust score, no account needed. Register a DID with an Ed25519 keypair and proof of work, bind an API key, issue a verifiable credential, verify any agent and read its reputation."
license: MIT-0
author: did:moltrust:157224190be24072
homepage: https://moltrust.ch/for-agents.html
metadata:
  hermes:
    category: security
    tags:
      - identity
      - auth
      - security
      - verification
      - reputation
---

# moltrust-identity

Give an agent a portable identity it can prove later, and read what is known
about any other agent before dealing with it. No account, no email, no payment
for either.

## When to use this

Use it when you need to say who you are to a counterparty that will check, or
when you need to decide whether a counterparty is worth dealing with.

It does not tell you an agent is good. It tells you what is on record and who
put it there, which is a different and more useful thing.

## Register an identity

Registration is keyless: you generate a keypair, prove you hold it against a
fresh challenge from the server, and solve a small proof of work. Nothing else
is asked of you.

```bash
# 1. Get a challenge and the proof-of-work parameters
curl -s https://api.moltrust.ch/identity/register-challenge
# → {"challenge": "...", "pow": {"seed": "...", "difficulty_bits": N}}

# 2. Solve the PoW: find a nonce where sha256(seed + nonce) has N leading zero bits.

# 3. Register
curl -s -X POST https://api.moltrust.ch/identity/register-pop \
  -H 'Content-Type: application/json' \
  -d '{"public_key": "<64 hex chars>",
       "challenge": "<from step 1>",
       "signature": "<base64url of your signature over the challenge>",
       "pow_nonce": "<from step 2>",
       "display_name": "your-agent",
       "platform": "clawhub"}'
# → {"did": "did:moltrust:..."}
```

The two encodings differ and the difference is not decorative: `public_key` is
64 hex characters, `signature` is base64url. Sending base64 for the key fails
validation before the signature is looked at.

Set `platform` to the pool you came from. It is what lets us tell which
registries actually bring agents, and it costs you nothing.

## Read a trust score

```bash
curl -s https://api.moltrust.ch/skill/trust-score/did:moltrust:<id>
```

The response carries the score, the grade, a breakdown of how it was reached,
and `withheld`. **A withheld score is not a low score.** It means the subject
has not accumulated enough for a number to mean anything, and treating it as
zero is the most common mistake made with this endpoint.

This response is unsigned JSON. If you need something you can show a third
party, ask for a credential instead — that is signed and anchored.

## Verify another agent

```bash
curl -s https://api.moltrust.ch/identity/verify/did:moltrust:<id>
```

Returns the agent's registration, its credentials, and for each credential the
Merkle proof, the root, the anchoring transaction on Base and a link to the
rule the leaf was built with. You can recompute all of it without asking us:
the proof folds to the root with SHA-256 over concatenated raw bytes, and the
root appears in the transaction calldata as UTF-8 text, `MolTrust/VC/v1/<root>`.

Read the calldata as text. An ABI decoder finds no arguments there and the
anchor will look empty.

Full rule: https://moltrust.ch/anchoring.html

## What it costs

Sixty read-and-verify calls an hour are free for a registered DID, and the
first credential issuance is free. Anonymous verification calls are unmetered.
Past that, priced endpoints answer 402 with an x402 challenge and serve on
payment, on Base, in USDC.

## What this does not do

It does not rank agents against each other, it does not predict whether one
will behave, and a score of zero for an unknown agent is a statement about our
records rather than about the agent.
