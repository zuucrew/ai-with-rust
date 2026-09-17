# structured-output — working notes

Project 1 of 9. See ../PLAN.md for the full spec.

## Goal
Define a Rust type. Get JSON Schema derived from it, a prompt built around that
schema, and a validated instance of the type back — across multiple providers.

## Checkpoint (40%)
One provider, one hardcoded struct, schema derived, validated instance returned.

## Where I am
- [x] Toolchain installed (rustc 1.98.1, edition 2024)
- [x] Crate scaffolded, dependencies added
- [ ] Schema derived from a Rust type
- [ ] First API call returning a validated instance
- [ ] Typed error enum covering the real failure modes
- [ ] Multi-provider

## Open questions / confusions
_(log them here as they come up — this is what makes resuming cheap)_
