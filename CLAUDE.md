# CLAUDE.md

Guidance for working in this repository.

## What this repo is

A specification repo, not a code repo — there is no build system here. It defines the wire protocol (transports, handshake, authentication, subscription model, push message formats, heartbeat/failover) that every per-language client implementation must conform to, plus a compliance checklist.

Do not look for a solution or project file to build, test, or lint — none exists, and none should be added. Treat any change here as a documentation/specification change.

## Editing conventions

- Each protocol concept is documented with the same shape: a rules list, a representative wire-format example, and compliance-checklist entries. Match that shape for new sections.
- Keep normative language (must/should/may) precise — client implementations in other repos are generated or hand-written directly against this wording.
- Changing accepted wire formats, required fields, or handshake/auth sequencing here has downstream impact on every implementing client; call out breaking vs. additive changes explicitly in the change description.

## Change tracking

A changelog follows a standard keep-a-changelog shape with an unreleased section. Update it alongside any substantive spec change.
