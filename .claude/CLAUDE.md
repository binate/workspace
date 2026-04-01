# Binate Project Workspace

This directory is a git repo (`github.com/binate/workspace`) with submodules for each Binate project repo. The submodules are independent repos — work in them directly as usual.

## Repos

- **explorations/** — Design docs, notes, grammar, plans, TODOs (`github.com/binate/explorations`)
- **bootstrap/** — Bootstrap interpreter written in Go (`github.com/binate/bootstrap`)
- **binate/** — Self-hosted interpreter and compiler written in Binate (`github.com/binate/binate`)
- **website/** — Project website (`github.com/binate/website`)

## Key Reference Files

- `explorations/binate-coding-guide.md` — Coding conventions and best practices
- `explorations/claude-notes.md` — Language design summary (decisions, status, rationale overview)
- `explorations/claude-discussion-detailed-notes.md` — Extended rationale and discussion
- `explorations/grammar.ebnf` — Authoritative grammar specification
- `explorations/claude-plan-2.md` — Self-hosted toolchain plan (Phase 5)

## Language Overview

Binate is a systems programming language with dual-mode execution (compiled + interpreted, seamless interop via function pointers). Key design points:

- Reference-counted memory with managed (`@T`) and raw (`*T`) pointers
- No GC, no ownership/borrowing — raw pointers are the escape hatch for cycles and hot paths
- Explicit interfaces with separate `impl` declarations, vtable-based dispatch
- Monomorphized generics with interface constraints
- No exceptions — errors are values (Go-style multiple returns)
- `[]T` raw slices (2 words), `@[]T` managed-slices (3 words)
- No built-in maps, no string type, no `append` — growable collections are library concerns
- `.bn` implementation files, `.bni` interface files
- Targets 32-bit systems primarily, with 64-bit support

## Project Status

Self-hosted toolchain (10 packages) is implemented. Self-hosted interpreter passes all 70 conformance tests. Self-hosted compiler produces native binaries via LLVM IR. Self-compilation works (bootstrap interprets compile.bn to compile compile.bn) but the resulting binary segfaults when actually compiling — debugging this is the current frontier.

## Conformance Tests

Run via `conformance/run.sh` in the bootstrap repo. Five modes: bootstrap, selfhost, compiled, compiled-interp, compiled-compiler.

## Working With This Codebase

- When editing bootstrap Go code, be aware of known quirks (e.g., StringVal vs SliceVal distinction)
- Binate source uses `pkg/` prefix for packages; `pkg/rt` is the runtime (written in Binate)
- Builtins (`make`, `make_slice`, `box`, `cast`, `bit_cast`, `len`, `unsafe_index`) are keywords, not functions

### Plans and Memories

Write plans and memory files to `explorations/` (and commit them), not to the default hidden locations (e.g., `.claude/` memory directory). This keeps project knowledge visible, version-controlled, and accessible outside of Claude Code.

### Learning From Mistakes

Whenever you make a mistake (rejected edit, wrong assumption, incorrect behavior, etc.), update this CLAUDE.md file with a note or instruction that prevents the same mistake in future conversations.

### Git

Since the repos are sibling directories (not a monorepo), use `git -C <path>` rather than `cd <path> && git ...`. For example: `git -C bootstrap status`, `git -C explorations push`.

### Bootstrap Subset Constraint

The self-hosted interpreter and compiler (under `binate/`) are written in Binate but must currently be runnable by the Go bootstrap interpreter. This means all self-hosted code must:

1. **Conform to the bootstrap subset** — no interfaces, no generics, no closures, no floats, no const-qualified types, no variadic parameters, no function values. See `explorations/bootstrap-subset.md` for the full list of what is and isn't supported.
2. **Be correct with respect to the full language spec** — the code must not rely on bootstrap-specific behavior that diverges from the intended language semantics. See `explorations/claude-notes.md` for the language design decisions.

In other words: write code that uses only bootstrap-supported features, but that would also be valid and correct under the full language as specified.

### Decision-Making

Don't make assumptions or value judgements about what's "worth doing" or "not worth the trouble." When unsure whether to do something (e.g., split a test file, refactor a helper, change an approach), ask the user instead of deciding unilaterally.
