# CLAUDE.md

## What This Repo Is

A specification repo, not a code repo — no build system. Defines the wire protocol (transports, handshake, authentication, subscription model, push message formats, heartbeat/failover) every per-language client implementation must conform to, plus a compliance checklist.

No solution/project file to build, test, or lint — none exists, none should be added. Treat any change as a documentation/specification change.

## Editing Conventions

- Each protocol concept: same shape — rules list, representative wire-format example, compliance-checklist entries. Match this shape for new sections.
- Keep normative language (must/should/may) precise — other repos' clients are generated or hand-written directly against this wording.
- Call out breaking vs. additive changes explicitly when changing accepted wire formats, required fields, or handshake/auth sequencing — downstream impact on every implementing client.

## Change Tracking

Keep-a-changelog shape with an unreleased section — update alongside any substantive spec change.
