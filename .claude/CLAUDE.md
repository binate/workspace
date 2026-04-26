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
- `explorations/ir-backend-guidelines.md` — IR vs backend responsibility split
- `explorations/ir-backend-cleanup-plan.md` — Work plan for multi-backend support
- `explorations/layout-extraction-plan.md` — Detailed plan for extracting layout to shared layer

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

Self-hosted toolchain is implemented and stable. Self-hosted compiler produces native binaries via LLVM IR; self-compilation works through gen1 and gen2 (boot-comp-comp / boot-comp-comp-comp), with all conformance modes green in CI. The bytecode VM (cmd/bni / pkg/vm) also passes all unit-test packages.

Current frontiers (parallel, both high priority):
1. **Language completeness beyond the bootstrap subset** — implementing the spec features the bootstrap doesn't yet support (interfaces, generics, closures, function values, etc.). See `explorations/bootstrap-subset.md` for the gap.
2. **Multi-backend support** — refactoring the IR/codegen boundary to extract shared logic, then adding a direct 32-bit ARM backend (tested via QEMU user-mode). See `explorations/ir-backend-cleanup-plan.md`.

## Conformance Tests

Run via `conformance/run.sh`. Modes (chains of: boot=bootstrap, int=bytecode VM, comp=compiler): boot, boot-comp, boot-comp-int, boot-comp-comp, boot-comp-comp-int, boot-comp-comp-comp. See `conformance/run.sh --help` for the full list.

## Working With This Codebase

- When editing bootstrap Go code, be aware of known quirks (e.g., StringVal vs SliceVal distinction)
- Binate source uses `pkg/` prefix for packages; `pkg/rt` is the runtime (written in Binate)
- Builtins (`make`, `make_slice`, `box`, `cast`, `bit_cast`, `len`, `unsafe_index`) are keywords, not functions

### Problem-Solving Approach

Do NOT simply work around issues. In general, issues should be root-caused and addressed properly. Pragmatic, short-term fixes are sometimes acceptable, but the user should be consulted before taking that approach.

### Don't Optimize for Quick Wins

Do NOT pick tasks based on "what's the cheapest unlock" or "what gives the most green checkmarks per unit effort." When triaging remaining work, the question is **what fix the codebase actually needs**, not what fix maximizes near-term test pass count. If a real bug requires a big change (ABI rework, IR refactor, layout extraction), say so and propose doing it — don't shop around the failure list for a smaller adjacent target so you can claim progress.

Specifically:
- Don't recommend deferring a real fix in favor of "easier" buckets unless the user has already expressed a deadline or scope constraint.
- Don't frame substantive fixes as optional ("we could do X, or skip to Y") when X is the actual root cause and Y is unrelated. Frame the choice as "here's the fix; want me to start?"
- Treat the user as a senior collaborator on a long-running project, not a stakeholder who needs morale-boosting deliverables. Most of the time, the right next step is the harder one.

### Plans and Memories

Write plans and memory files to `explorations/` (and commit them), not to the default hidden locations (e.g., `.claude/` memory directory). This keeps project knowledge visible, version-controlled, and accessible outside of Claude Code.

### Stay Within the Asked Scope

When asked to add new code (a script, a tool, a check), add only that. Do not wire it up to other systems on your own — CI workflows, hooks, schedulers, runners, or any pipeline that triggers automation. "Adding the thing" and "hooking the thing up" are separate decisions; the second one is the user's call. If you think wiring it up is the obvious next step, propose it and wait.

This is distinct from bundling tests with code changes (which IS expected). The line: tests verify the code you wrote; CI/automation hookup changes when and where it runs, which is a scope decision the user owns.

### Learning From Mistakes

Whenever you make a mistake (rejected edit, wrong assumption, incorrect behavior, etc.), update this CLAUDE.md file with a note or instruction that prevents the same mistake in future conversations.

### Bug Discovery Protocol

Whenever you discover a bug (whether or not you fix it immediately):
1. **Add a test**: conformance test if it's a runtime behavior issue, unit test if it's an IR/codegen/type issue.
2. **Mark it as failing**: add an `.xfail.<mode>` file for each mode where it fails (with a one-line description), or mark the unit test as expected-fail.
3. **Add a TODO**: describe the bug, root cause (if known), and which test covers it in `explorations/claude-todo.md`.

This ensures bugs are tracked, reproducible, and visible — even if the fix is deferred.

### Memory Management: Never Leak

The compiler must NEVER generate code that leaks memory. If a managed allocation is created, it must eventually be RefDec'd. The only acceptable "leaks" are user-created reference cycles (which are user error — Binate uses refcounting, not GC).

If user code creates a use-after-free (e.g., storing a temporary's raw slice in a variable and using it after the statement), that is **user error**, not a compiler bug. The compiler should not suppress RefDec to prevent UAF — that trades a detectable crash for a silent leak, which is worse.

Concretely: `consumeTemp` should only be used when ownership genuinely transfers (e.g., `var x @T = make(T)` — the variable owns it). It must NOT be used to "borrow" backing for raw slices — the temp stays in cleanup and gets RefDec'd at end of statement.

### Git

Since the repos are sibling directories (not a monorepo), use `git -C <path>` rather than `cd <path> && git ...`. For example: `git -C bootstrap status`, `git -C explorations push`.

### Never Use Git Stash

Do NOT use `git stash` — it constantly leads to lost work. Instead, create a temporary branch and commit your changes there.

### Worktree Discipline

When told to work in a git worktree, **stay in that worktree**. Do not touch any other worktree or the main checkout unless explicitly told to do so. There may be ongoing work in other worktrees or the main checkout — modifying them (cherry-picking, pushing, syncing, etc.) without being asked risks disrupting that work.

### NEVER cherry-pick or push to main without explicit user approval

**NEVER** cherry-pick commits to main, push to main, or modify the main branch in any way without the user explicitly telling you to do so in that moment. "Go ahead" or "cherry-pick now" is approval. Anything else is not. Do not infer approval. Do not assume. ASK every single time, even if the user approved a cherry-pick 30 seconds ago — each one requires separate approval.

This includes **any** git operation in the main checkout (`~/binate/binate`): do NOT run `git fetch`, `git rebase`, `git pull`, `git reset`, `git add`, `git commit`, `git push`, etc. there without explicit instruction. The main checkout may be in use by the user or other workers; touching it risks disrupting their work. When told to "resync your worktree", run the git command with `git -C <worktree-path> ...` — never from the main checkout or without `-C`.

### Resyncing a Worktree

When told to "resync your worktree" (for the binate repo), rebase against the **local** `main` branch (checked out in `~/binate/binate`), not `origin/main`. The local main may be ahead of origin. Command form:

    git -C <worktree-path> fetch <main-checkout-path> main
    git -C <worktree-path> rebase FETCH_HEAD

Or equivalently, if the worktree already tracks the same repo, `git -C <worktree-path> rebase main` (but never `git rebase origin/main` unless explicitly told to). Do NOT run any git command inside `~/binate/binate` itself.

### Preserving Command Output

Instead of directly grepping (etc.) the output of commands — especially ones that may take a long time, like test runs — pipe the output through `tee` to also write it to a temporary file, then grep the file. That way, if you want to see more of the output, you don't have to run the command again (which takes a long time). Worse, if the output isn't deterministic, re-running may not reproduce what you're looking for.

Example: `go run ... --test pkg/foo 2>&1 | tee /tmp/test_foo.out | tail -5` then `grep FAIL /tmp/test_foo.out`.

### Pre-Existing Test Failures

If tests are already failing when you start work, that is still your problem. Do not assume your work is fine just because tests were already broken — pre-existing failures can mask new issues introduced by your changes. It is sometimes acceptable to temporarily set aside a pre-existing failure and commit your own work first, but in general you should work to address already-failing tests next (or first), unless there is a good, deliberate reason not to. Ignoring pre-existing failures and moving on compounds the problem.

### IR/Backend Boundary

When modifying the compiler backend (`pkg/codegen`) or IR layer (`pkg/ir`), respect the split defined in `explorations/ir-backend-guidelines.md`:

- **Memory layout** (structs, arrays, slices, managed-slices, managed pointer headers, future interface values) is a **language-level contract** shared by all compiler backends AND the interpreter (for dual-mode interop). It belongs in `pkg/types`, not in any backend.
- **Other language-semantic logic** (name mangling, string constant collection, runtime function manifest) belongs in a **shared layer** (IR, types, or shared packages), not in any specific backend.
- **Target-specific logic** (instruction selection, register allocation, calling convention, type representation format, binary format, debug info) belongs in the **backend**.
- **Type layout functions** (`types.SizeOf`, `types.AlignOf`, `types.FieldOffset`) must be parameterized by target (pointer size, int size, alignment) — do not hardcode 64-bit assumptions.

If you're adding new functionality to `pkg/codegen`, ask: "would a different backend need this same logic?" If yes, put it in a shared location. For layout-related logic, also ask: "does the interpreter need to agree with this?" If yes, it must be in `pkg/types`.

### Coding Guide

Read `explorations/binate-coding-guide.md` before writing Binate code. Key rules: returning a raw slice (`[]T`) from a function that allocates is almost always wrong — use `@[]T` instead. Raw slices borrow; managed-slices own.

### Bootstrap Subset Constraint

The self-hosted interpreter and compiler (under `binate/`) are written in Binate but must currently be runnable by the Go bootstrap interpreter. This means all self-hosted code must:

1. **Conform to the bootstrap subset** — no interfaces, no generics, no closures, no floats, no const-qualified types, no variadic parameters, no function values. See `explorations/bootstrap-subset.md` for the full list of what is and isn't supported.
2. **Be correct with respect to the full language spec** — the code must not rely on bootstrap-specific behavior that diverges from the intended language semantics. See `explorations/claude-notes.md` for the language design decisions.

In other words: write code that uses only bootstrap-supported features, but that would also be valid and correct under the full language as specified.

### Tools

`ditty` (https://github.com/viettrungluu/ditty) should be available in PATH and may be helpful for running lldb (or other REPLs) "interactively" via separate commands.

### Decision-Making

Don't make assumptions or value judgements about what's "worth doing" or "not worth the trouble." When unsure whether to do something (e.g., split a test file, refactor a helper, change an approach), ask the user instead of deciding unilaterally.

### Language Semantics

**NEVER change language semantics (type system rules, implicit conversions, assignability, etc.) without explicitly asking the user first.** This includes adding new implicit conversions (e.g., `[]T → @[]T`), changing type compatibility rules, or modifying how the type checker accepts or rejects code. If a type error blocks your work, fix the code that's wrong — don't change the language rules to make it compile.

### Terminology

Use "managed-slice" (hyphenated) for `@[]T`, because it is a distinct type from `@([]T)`. Similarly "managed-pointer" for `@T` is technically correct but rarely needed since `@(*T)` is rare. "Managed struct" (two words, no hyphen) is fine — it's just a struct that happens to be managed.
