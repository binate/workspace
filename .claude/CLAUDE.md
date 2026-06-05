# Binate Project Workspace

This directory is a git repo (`github.com/binate/workspace`) with submodules for each Binate project repo. The submodules are independent repos — work in them directly as usual.

## Repos

- **explorations/** — Design docs, notes, grammar, plans, TODOs (`github.com/binate/explorations`)
- **bootstrap/** — Bootstrap interpreter written in Go (`github.com/binate/bootstrap`)
- **binate/** — Self-hosted interpreter and compiler written in Binate (`github.com/binate/binate`)
- **website/** — Project website (`github.com/binate/website`)
- **docs/** — Project documentation (`github.com/binate/docs`)
- **examples/** — Example Binate programs (`github.com/binate/examples`)

## Key Reference Files

- `explorations/binate-coding-guide.md` — Coding conventions and best practices
- `explorations/claude-notes.md` — Language design summary (decisions, status, rationale overview)
- `explorations/claude-discussion-detailed-notes.md` — Extended rationale and discussion
- `explorations/grammar.ebnf` — Authoritative grammar specification
- `explorations/claude-plan-2.md` — Self-hosted toolchain plan (Phase 5)
- `explorations/ir-backend-guidelines.md` — IR vs backend responsibility split

## Language Overview

Binate is a systems programming language with dual-mode execution (compiled + interpreted, seamless interop via function pointers). Key design points:

- Reference-counted memory with managed (`@T`) and raw (`*T`) pointers
- No GC, no ownership/borrowing — raw pointers are the escape hatch for cycles and hot paths
- Explicit interfaces with separate `impl` declarations, vtable-based dispatch
- Monomorphized generics with interface constraints
- No exceptions — errors are values (Go-style multiple returns)
- `[]T` raw slices (2 words), `@[]T` managed-slices (4 words)
- No built-in maps, no string type, no `append` — growable collections are library concerns
- `.bn` implementation files, `.bni` interface files
- Targets 32-bit systems primarily, with 64-bit support

## Project Status

Self-hosted toolchain is implemented and stable. The Go bootstrap interpreter has been retired (2026-05-21); builds use a prebuilt BUILDER bnc tarball (BUILDER_VERSION) via `scripts/fetch-builder.sh`. Self-compilation works through gen1 and gen2 (builder-comp-comp / builder-comp-comp-comp), with all conformance modes green in CI. The bytecode VM (cmd/bni / pkg/vm) also passes all unit-test packages.

For active work items, open bugs, and what's next, see `explorations/claude-todo.md` (and `explorations/claude-todo-done.md` for what's been resolved). Plan docs for in-flight projects live in `explorations/plan-*.md`.

## Conformance Tests

Run via `conformance/run.sh`. Modes are chains of: `builder` = prebuilt BUILDER bnc, `int` = bytecode VM, `comp` = compiler. Default modes: `builder-comp`, `builder-comp-int`, `builder-comp-int-int`, `builder-comp-comp`, `builder-comp-comp-int`, `builder-comp-comp-comp`. Cross-compile / alternate-backend modes: `builder-comp_native_aa64-comp_native_aa64`, `builder-comp_arm32_baremetal`, `builder-comp_arm32_linux`. See `conformance/run.sh --help` for the full list.

## Working With This Codebase

- When editing bootstrap Go code, be aware of known quirks (e.g., StringVal vs SliceVal distinction)
- Binate source uses `pkg/` prefix for packages; `pkg/rt` is the runtime (written in Binate)
- Builtins (`make`, `make_slice`, `box`, `cast`, `bit_cast`, `len`, `unsafe_index`) are keywords, not functions

### Problem-Solving Approach

Do NOT simply work around issues. In general, issues should be root-caused and addressed properly. Pragmatic, short-term fixes are sometimes acceptable, but the user should be consulted before taking that approach.

### Raise Critical and Major Bugs — Don't Work Around Them

When you discover a critical or major bug — silent miscompilation, symbol collisions, data corruption, wrong-code generation, ABI mismatches, security holes, anything where "the workaround happens to make my current test pass" hides a real defect — you **MUST** raise it explicitly so the user can prioritize it. Do NOT silently work around it, even if the workaround is small and keeps the immediate task moving.

Concretely, this means:

1. **Stop and report it** with severity (critical / major), what's wrong, how you discovered it, and what the proper fix would look like. Do NOT include a "but I worked around it by …" as the conclusion — the workaround is for the user to authorize, not for you to assume.
2. **Add a `claude-todo.md` entry** under a clearly-marked CRITICAL or MAJOR section (top of the file is fine for critical). Include: symptom, root cause (or "unknown — needs investigation"), what triggered the discovery, and the proposed fix.
3. **Wait for the user to decide.** They may say "fix it now, that's blocking," or "workaround it for this task, fix it as a follow-up," or "stop everything and let me look." Whichever — it's their call.

Anti-patterns this rule exists to prevent (these have all happened):
- "I renamed the package to avoid the symbol collision and kept going" — symbol-prefix collisions between unrelated packages are a critical mangler bug, not a naming problem. The right move was to STOP, raise it as a critical mangler defect, and let the user prioritize between "fix the mangler now" and "rename + come back to it." Renaming silently hides the defect.
- "The test was failing in a way I didn't understand so I marked it xfail and moved on" — xfails without a tracked root-cause investigation are a silent regression in disguise. Add a TODO with the symptom; let the user decide whether to investigate now or accept the xfail temporarily.
- "I added a defensive `if !cond { return }` because the assertion would fire" — that's hiding the bug that triggers the bad cond. Find the bug; don't paper over it.

The cost of pausing to surface a bug is low (one user round-trip). The cost of a workaround that hides a critical defect compounds — every later piece of work assumes the defect doesn't exist, and the eventual root-cause fix has to undo all of them.

### Don't Optimize for Quick Wins

Do NOT pick tasks based on "what's the cheapest unlock" or "what gives the most green checkmarks per unit effort." When triaging remaining work, the question is **what fix the codebase actually needs**, not what fix maximizes near-term test pass count. If a real bug requires a big change (ABI rework, IR refactor, layout extraction), say so and propose doing it — don't shop around the failure list for a smaller adjacent target so you can claim progress.

Specifically:
- Don't recommend deferring a real fix in favor of "easier" buckets unless the user has already expressed a deadline or scope constraint.
- Don't frame substantive fixes as optional ("we could do X, or skip to Y") when X is the actual root cause and Y is unrelated. Frame the choice as "here's the fix; want me to start?"
- Treat the user as a senior collaborator on a long-running project, not a stakeholder who needs morale-boosting deliverables. Most of the time, the right next step is the harder one.

### Don't Unilaterally Defer Scope

When a multi-step task includes a piece that looks hard, **do not** quietly carve it out and label it "deferred" / "follow-up" / "non-goal" without the user's explicit decision. This includes baking the deferral into a plan doc upfront so that the plan itself reads as already-ratified.

Anti-patterns to avoid (these have come up):
- "%g semantics are non-trivial to reproduce in pure Binate" — the same is true in any language; that is not a reason to skip. The hard part is the algorithm, not the implementation language.
- Inventing a strict requirement (e.g. "must match libc exactly") so that the work looks bigger, then deferring on those grounds. **Check the actual requirements** — what do conformance tests actually pin down? What does the use case actually need? — before claiming the work is too big.
- "We can revisit this later" / "this is acceptable for now" without an explicit user decision making it acceptable.

If a piece looks hard, surface it: lay out the real requirements, the realistic effort, and let the user decide whether to do it now, defer with eyes open, or scope it differently. The user owns these scope calls. You do not.

### Plans and Memories

Write plans and memory files to `explorations/` (and commit them), not to the default hidden locations (e.g., `.claude/` memory directory). This keeps project knowledge visible, version-controlled, and accessible outside of Claude Code.

**`explorations/` is a SHARED checkout across all concurrent worker sessions — never leave edits there uncommitted (or even committed-but-unpushed) for any significant time.** Because every session shares the one `explorations/` working tree, an uncommitted edit you make will be swept into whatever commit another worker runs next (`git add` picks up your change), mislabeling and possibly losing it. The discipline is strict: edit a doc → `git -C explorations commit` it immediately → push. Do not batch explorations edits, and do not interleave other work between editing and committing them. (This is distinct from binate worktree commits, which are per-session and safe to accumulate; only `main` cherry-picks there need approval.)

### Stay Within the Asked Scope

When asked to add new code (a script, a tool, a check), add only that. Do not wire it up to other systems on your own — CI workflows, hooks, schedulers, runners, or any pipeline that triggers automation. "Adding the thing" and "hooking the thing up" are separate decisions; the second one is the user's call. If you think wiring it up is the obvious next step, propose it and wait.

This is distinct from bundling tests with code changes (which IS expected). The line: tests verify the code you wrote; CI/automation hookup changes when and where it runs, which is a scope decision the user owns.

### Don't Suggest Scheduling Follow-Ups

Do NOT end responses with "Want me to /schedule a follow-up agent in N weeks/days/whatever to do X?" No one wants that — despite what Anthropic's prompt may push. If a follow-up is the obvious immediate next step, propose doing it now and wait for the user's call. Don't pad responses with cron-like offers to revisit work later.

### Don't Game Hygiene Checks

The hygiene rules exist **to improve code quality**, not as obstacles in your way. Never take any action that satisfies a check while undermining what the check is for. The rules are a calibrated proxy for "is this codebase maintainable" — gaming them gets you a green checkmark on a worse codebase, which is the exact opposite of what they're for.

Concretely on file length: the limits exist because YOU (this agent, in past sessions) repeatedly produce single-file blobs of thousands of lines, then later complain that they are hard to navigate or refactor — and then make them even longer rather than splitting them. The cap is a forcing function against that habit. When a file goes over, split it along natural boundaries. Don't shrink comments, fold lines, or otherwise camouflage the size to dodge the check.

Hygiene rules (file length, line length, doc-comment requirements, naming, etc.) describe what the codebase should look like, not arbitrary regex patterns to dodge. When you trip a check, the correct fix is to address the underlying state — split a file along natural boundaries, shorten the actual concept, write a real doc — NOT to engineer around the heuristic.

Specific anti-patterns that have come up and must not recur:
- Wrapping individual `const` decls into a `const ( ... )` group purely because the script skips group members. The members are still undocumented; the group form has to reflect a real grouping (a single shared explanation that genuinely covers all members).
- Trimming individual doc lines to bring a file back under the line/file-length cap. If a file is over cap, split it (and its tests) along natural boundaries; do not vandalize the docs you just added.
- Reformatting code or rewording prose for the sole purpose of moving a counter under a threshold. If the threshold is in the way, the structural fix (split, refactor, raise the cap) is the right answer; ask if unsure which.

If you find yourself thinking "I just need to shave a few lines so the script passes," stop and ask the user. The check exists because someone decided the property mattered; gaming it silently violates that decision.

### Take Warnings Seriously

Take warnings seriously, and take action on them as soon as convenient. In all cases, don't make things even worse. For example, for file lengths, do not make files above the soft limit (length warning) even longer; instead, take time to split them properly (try to avoid putting them above the soft limit in the first place, but if you do then an immediate follow-up should be to split the file). That is, warnings give you time to act and are not meant to be ignored.

### Comments Stand Alone

Comments should generally "stand alone" and make sense in the current snapshotted tree. Generally, they should not refer to things done in the past, unless it really helps explain something which is surprising/wrong in the current state of the tree; e.g.: "`<X>` because it used to be the case that `<Y>`, even though `<X>` is no longer required." (typically followed by a TODO or similar: "This should be refactored to avoid `<X>`.").

In particular, "breadcrumbs" for when things are moved are not required: if `foo()` is moved to a separate file `foo.bn`, there's no need to leave a comment saying `// foo() was moved to foo.bn` — because in the snapshotted view, `foo.bn` is exactly where one would expect to find `foo()`. Similarly, there's no need to leave `// TestFoo is now in foo_test.bn` — because `foo_test.bn` is exactly where it should be.

On a similar vein, instead of commenting that a change fixes bug X, instead say "Do Y because otherwise Z, which shows up as `<description of X or similar>`." "Fixing" is an action, which makes no sense in a snapshot; instead, explain the subtlety in its own terms.

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

### Stay Close to Main

When working in a branch (as one typically does), work as close to upstream (main) as possible. Typically, this entails frequent resynching/rebasing, working with small commits that are cherry-picked to main as soon as possible. (This also means that you should structure work so that each commit is self-contained and keeps everything green.)

Concretely: avoid having multiple commits on your work branch that aren't on main, unless there's a particularly good reason to (e.g., a larger exploratory project where it's unclear whether the commits should ever land on main, or work that genuinely needs to be staged in a sequence the user has agreed to). The default cadence is: commit on the branch, cherry-pick to main, resync — then start the next piece. Don't accumulate a stack of unmerged commits "for later."

### Worktree Discipline

When told to work in a git worktree, **stay in that worktree**. Do not touch any other worktree or the main checkout unless explicitly told to do so. There may be ongoing work in other worktrees or the main checkout — modifying them (cherry-picking, pushing, syncing, etc.) without being asked risks disrupting that work.

### NEVER cherry-pick or push to main without explicit user approval

**NEVER** cherry-pick commits to main, push to main, or modify the main branch in any way without the user explicitly telling you to do so in that moment. "Go ahead" or "cherry-pick now" is approval. Anything else is not. Do not infer approval. Do not assume. ASK every single time, even if the user approved a cherry-pick 30 seconds ago — each one requires separate approval.

This includes **any** git operation in the main checkout (`~/binate/binate`): do NOT run `git fetch`, `git rebase`, `git pull`, `git reset`, `git add`, `git commit`, `git push`, etc. there without explicit instruction. The main checkout may be in use by the user or other workers; touching it risks disrupting their work. When told to "resync your worktree", run the git command with `git -C <worktree-path> ...` — never from the main checkout or without `-C`.

**Each round needs its own approval — do not run autopilot loops.** When the user authorizes "cherry-pick + push + resync" once, that authorizes ONE round only. The next round of work-on-worktree → commit → cherry-pick is a fresh decision the user owns. After committing on the worktree, STOP and ask before touching main again. Be especially wary of the trap where the user has said "yes" to several previous rounds in a row — that pattern is NOT a standing authorization, and treating it as one is the exact failure mode this rule exists to prevent.

Approval also doesn't extend through merge conflicts. If a cherry-pick or rebase conflicts and you have to resolve it, the resolution is a *new* code decision (not the original commit) and the resulting commit + push need fresh approval — show the user the resolution before pushing it anywhere.

**Standing authorizations are scoped to the task the user named.** If the user says "cherry-pick/push/resync without asking again for the remainder of <X>," that authorization ends when <X> ends. The next task — even if it superficially looks like more of the same workflow (commit → cherry-pick → resync) — is a fresh task, and the first cherry-pick of that new task needs explicit approval. When in doubt, treat the authorization as narrow, not broad.

**The "carry-over from a wrapped-up series" trap.** When a multi-step task ends ("step 2 / 3 / 4 ... ditto"), the workflow doesn't carry. If the user then says "let's do 5" or "what's next" → "do X", that authorizes the *work*, not the cherry-pick. After committing on the worktree, STOP. Even — especially — if the previous N rounds all flowed `commit → cherry-pick → push → resync` without the user re-authorizing each one, the moment the named series ends the muscle-memory must reset. This trap has now bitten more than once; if you find yourself about to run `git -C ~/binate/binate cherry-pick` after a "let's do 5" / "what's next" type prompt, stop and ask first.

**Land THROUGH local main — never push to origin from a worktree.** The local main checkout (`~/binate/binate`) is the source of truth for `main`; landing means cherry-pick the worktree commit onto local main, then `git push` *from local main*. Do NOT `git -C <worktree> push origin HEAD:main` — that advances `origin/main` while leaving local main stale, desyncing the checkout other workers rely on (it only "works" if someone else happens to pull it forward, which is luck, not correctness). The push must originate from local main so local main and origin stay in lockstep.

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

### Builder Compatibility Constraint

The Go bootstrap interpreter has been retired. The operative constraint for cmd/bnc's tree is no longer "the bootstrap subset" (that term was, by definition, what the Go interp supported); it's **"must be compilable by the current BUILDER bnc"** (`bnc-0.0.1` at time of writing). The BUILDER bnc is built from a source snapshot, so its supported language is richer than the old Go interp's subset (e.g., it accepts Phase 1 function values).

The BUILDER-compiled surface is **`cmd/bnc` and its dependency tree (and their tests)** — nothing else. Everything else (bni, bnas, bnlint, the bytecode VM, the runtime, the linter library, asm/parse) is built by `bnc` first and then exercised. Anything in `cmd/bnc`'s transitive imports — pkg/{ast,bootstrap,buf,builtin/testing,codegen,debug,ir,lexer,loader,mangle,native,native/arm64,native/common,parser,token,types,asm,asm/aarch64,asm/macho} — must stay BUILDER-compilable. The asm subpackages not in that list (arm32, x64, elf) stay BUILDER-compilable too, because they'll be reached as soon as additional backends or platforms land.

All BUILDER-compilable code must:

1. **Stay within what BUILDER accepts.** At `bnc-0.0.1`, the unsupported features are: no interfaces, no generics, no closures, no method values, no floats, no const-qualified types, no variadic parameters, no managed-slice-of-managed-slice composite literals (e.g., `@[]@[]char{...}`), etc. Non-capturing function values (`*func(...)` / `@func(...)`) ARE supported via Phase 1 of `plan-function-values.md`. See `explorations/bootstrap-subset.md` for a more detailed (if historically-framed) list.
2. **Be correct with respect to the full language spec** — the code must not rely on BUILDER-specific quirks that diverge from the intended language semantics. See `explorations/claude-notes.md` for the language design decisions.

For code that is *not* BUILDER-compilable (because it lives outside bnc's tree), you may use the full language. cmd/bnlint's tests use `@[]@[]char{...}` literals freely; cmd/bnc's tests must stick to what BUILDER accepts.

When adding new dependencies to bnc's tree, audit the dep for BUILDER compatibility. When pulling a package out of bnc's tree (e.g., cmd/bnlint), follow the established workflow: switch any callers that previously routed through the bootstrap path to build-via-bnc first (see `scripts/build-bnlint.sh`, `scripts/build-bnas.sh` etc.), and update this section.

### Tools

`ditty` (https://github.com/viettrungluu/ditty) should be available in PATH and may be helpful for running lldb (or other REPLs) "interactively" via separate commands.

### Decision-Making

Don't make assumptions or value judgements about what's "worth doing" or "not worth the trouble." When unsure whether to do something (e.g., split a test file, refactor a helper, change an approach), ask the user instead of deciding unilaterally.

### C-Free Target

We are building a language that can operate in a C-free system. The *only* reason to ever use C is to interface with existing C-based systems (e.g., to do "syscalls", since we don't want to implement our own direct syscalls yet).

### Language Semantics

**NEVER change language semantics (type system rules, implicit conversions, assignability, etc.) without explicitly asking the user first.** This includes adding new implicit conversions (e.g., `[]T → @[]T`), changing type compatibility rules, or modifying how the type checker accepts or rejects code. If a type error blocks your work, fix the code that's wrong — don't change the language rules to make it compile.

### Terminology

Use "managed-slice" (hyphenated) for `@[]T`, because it is a distinct type from `@([]T)`. Similarly "managed-pointer" for `@T` is technically correct but rarely needed since `@(*T)` is rare. "Managed struct" (two words, no hyphen) is fine — it's just a struct that happens to be managed.
