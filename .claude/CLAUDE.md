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
- `docs/spec/binate.ebnf` — Authoritative (canonical) grammar specification; Annex A (`docs/spec/annex-a-grammar-summary.md`) is generated from it via `docs/scripts/gen-annex-a.py`. (The old `explorations/grammar.ebnf` is retired — a redirect tombstone.)
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

Run via `conformance/run.sh`. Modes are chains of: `builder` = prebuilt BUILDER bnc, `int` = bytecode VM, `comp` = compiler. Default modes: `builder-comp`, `builder-comp-int`, `builder-comp-int-int`, `builder-comp-comp`, `builder-comp-comp-int`, `builder-comp-comp-comp`. Cross-compile / alternate-backend modes: `builder-comp_native_aa64-comp_native_aa64`, `builder-comp_native_x64_darwin-comp_native_x64_darwin`, `builder-comp_native_arm32_baremetal`, `builder-comp_arm32_baremetal`, `builder-comp_arm32_linux`. See `conformance/run.sh --help` for the full list.

**CRITICAL — two arm32 modes, easily confused (this HAS bitten):** `builder-comp_arm32_baremetal` / `builder-comp_arm32_linux` (NO `native`) are the **LLVM** arm32 cross-compile — mature, ~2634 passing, and they do **NOT** exercise `pkg/binate/native/arm32` at all. `builder-comp_native_arm32_baremetal` (WITH `native`) is the **native arm32 backend** (the P4 self-hosted project) — currently incomplete (~2026 pass / 611 fail). When verifying ANY change to `pkg/binate/native/arm32`, you MUST use `builder-comp_native_arm32_baremetal`; running the non-`native` mode gives a green result that tested the LLVM backend, not your code. (Same distinction for aa64/x64: the `native` in the mode name is load-bearing.) The ≥2007-passing baseline referenced in `plan-native-arm32.md` is the **native** mode.

## Working With This Codebase

- When editing bootstrap Go code, be aware of known quirks (e.g., StringVal vs SliceVal distinction)
- Binate source uses `pkg/` prefix for packages; `pkg/rt` is the runtime (written in Binate)
- Builtins (`make`, `make_slice`, `box`, `cast`, `bit_cast`, `len`, `unsafe_index`, `sizeof`, `alignof`, `present`, `same`) are keywords, not functions

### Problem-Solving Approach

Do NOT simply work around issues. In general, issues should be root-caused and addressed properly. Pragmatic, short-term fixes are sometimes acceptable, but the user should be consulted before taking that approach.

**Routing around a bug via a different primitive is STILL a workaround — and "make CI green" is not a license for it.** This has bitten: `findStubsFile` (a *dead* lookup for a file nothing produces) tripped a bundled `os.Stat` ENOENT bug on x86-64 and reddened Linux CI; the "fix" was to swap `os.Stat` → `os.Open` because `os.Open` happened not to hit the bug. That is a hack — it dodges the bug and even degrades semantics (accepts directories) to do so. The correct fix was to delete the dead code (the lookup, the stat, and the bug-exposure all went away at once). Tells that you are about to hack rather than fix: "this other call doesn't hit the bug, so I'll use it," "this makes CI green," "the real bug is in a frozen artifact I can't reach so I'll work around it here." When you catch yourself there, STOP and root-cause — or surface the blocker and let the user decide — rather than reaching for the primitive that incidentally avoids the symptom.

**When the user questions why you do something that looks pointless, treat it as "this may be dead/wrong — INVESTIGATE," not as a prompt to rationalize the existing behavior.** This has bitten: the user asked "why are we statting a file that should never exist?" — a direct pointer that the whole lookup was dead — and the answer was a hand-wavy "it's an optional mechanism" defense instead of actually checking whether anything produces or ships that file (nothing did). A user "why are we doing X?" about something that looks vestigial is usually a hypothesis that X is wrong or dead; verify the hypothesis (grep for producers/consumers, check git history for a revert, check whether removing X changes anything) before defending X.

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

### Keep Higher-Level Goals in View — Micro-Tasks Are Reconnaissance, Not the Reward

You are NOT rewarded or evaluated on completing micro-tasks per se. Tunnel-visioning on a mechanical sub-step while losing sight of the higher-level goal is **extremely detrimental** — it is one of the worst failure modes, worse than doing the micro-task slowly. Before and during any task, keep asking: what is the larger goal, and does this step actually serve it? When a task is handed to you as "do X," X is usually a *means*; understand the end it serves and keep that in view.

At the same time, *attempting* a micro-task has real value precisely because **you learn things by doing it**, and those discoveries inform subsequent work. The attempt is reconnaissance. (Example: trying to delete `bootstrap.Itoa` is what surfaced the two actually-valuable findings — that calling `myInt.String()` shouldn't require importing `lang`, and that test runners can't yet depend on the stdlib. Those discoveries mattered more than the deletion.) So: try things, but treat **what you learn** as the point, not the green checkmark.

Concretely:
- When you hit a wrinkle in a micro-task (a link failure, a policy check, an unexpected dependency), STOP and ask whether it reveals something about the higher-level goal. Do not just route around it to get the check to pass — the wrinkle is often the most valuable signal in the task.
- Surface discoveries and discuss their broader implications with the user, rather than unilaterally picking a local workaround and grinding on. The user owns the higher-level direction.
- "I completed the sub-task" is not success if you bulldozed past what it taught you. Bring the learning back; let it re-shape the plan.

### Don't Unilaterally Defer Scope

When a multi-step task includes a piece that looks hard, **do not** quietly carve it out and label it "deferred" / "follow-up" / "non-goal" without the user's explicit decision. This includes baking the deferral into a plan doc upfront so that the plan itself reads as already-ratified.

Anti-patterns to avoid (these have come up):
- "%g semantics are non-trivial to reproduce in pure Binate" — the same is true in any language; that is not a reason to skip. The hard part is the algorithm, not the implementation language.
- Inventing a strict requirement (e.g. "must match libc exactly") so that the work looks bigger, then deferring on those grounds. **Check the actual requirements** — what do conformance tests actually pin down? What does the use case actually need? — before claiming the work is too big.
- Mistaking an **illustrative example** for the requirement, then declaring a target/platform out of scope because the *example* doesn't apply there. This has bitten: the `__c_global` spec uses POSIX `environ` as its *example* C global, and the plan then proposed making `__c_global` "fail-loud indefinitely" on native-arm32 because "bare metal has no libc/`environ`" — inverting reality (bare metal is where C-library interop, e.g. device drivers and MMIO register blocks exposed as C globals, matters *more*). The feature is "reach any C global"; `environ` is one instance. When you catch yourself scoping out a target because *the example* wouldn't work there, stop — separate the example from the capability, and check whether the capability is still wanted there (it usually is). Bonus tell: if the "hard part" you're deferring turns out to be the *easy* path on that target (arm32's static/non-PIE link needs only the existing absolute reloc, no GOT), the deferral was never justified.
- "We can revisit this later" / "this is acceptable for now" without an explicit user decision making it acceptable.

If a piece looks hard, surface it: lay out the real requirements, the realistic effort, and let the user decide whether to do it now, defer with eyes open, or scope it differently. The user owns these scope calls. You do not.

### Call Out Deviations and Get Confirmation

If you deviate from something that was specifically requested or indicated — because you don't think it's possible, you think it's a bad idea, or you think there's some blocker — you **MUST** explicitly call it out and get confirmation. Do NOT silently substitute your own judgment for what was asked, and do NOT proceed on a changed plan and only mention the change in passing (or not at all).

Concretely:
- Name the specific thing you were asked to do, state that you're not doing it (or doing it differently), and give the reason (impossible / bad idea / blocker).
- Then **wait** for the user to confirm the deviation before proceeding. The user may agree, override your concern, or pick a third option — it's their call.
- This applies even when you're confident you're right. Being right about the blocker doesn't authorize silently routing around the request; the user still decides what to do about it.

This is the general case of the scope rules above: whether the deviation is "defer this piece," "do it a different way," or "skip it entirely," the discipline is the same — surface it explicitly and get a decision, don't decide unilaterally.

### Plans and Memories

Write plans and memory files to `explorations/` (and commit them), not to the default hidden locations (e.g., `.claude/` memory directory). This keeps project knowledge visible, version-controlled, and accessible outside of Claude Code.

**`explorations/` is a SHARED checkout across all concurrent worker sessions — never leave edits there uncommitted (or even committed-but-unpushed) for any significant time.** Because every session shares the one `explorations/` working tree, an uncommitted edit you make will be swept into whatever commit another worker runs next (`git add` picks up your change), mislabeling and possibly losing it. The discipline is strict: edit a doc → `git -C explorations commit` it immediately → push. Do not batch explorations edits, and do not interleave other work between editing and committing them. (This is distinct from binate worktree commits, which are per-session and safe to accumulate; only `main` cherry-picks there need approval.)

**NEVER create a git worktree inside `explorations/` (and NEVER spawn a subagent or workflow with `isolation: worktree` for work that targets the explorations repo).** The worktree-isolation mechanism parks a checkout at `explorations/.claude/worktrees/agent-<id>/` — a nested git repo *inside* the shared explorations working tree. That is broken on two counts: (1) the next worker's `git add -A` picks it up and stages it as an embedded-repo gitlink, corrupting the commit (this has happened); (2) explorations is docs/plans, not code — there is nothing to isolate, and the shared-checkout discipline above already governs it. If a task genuinely needs an isolated worktree, it is a *code* task and belongs in the binate repo (`isolation: worktree` / `git -C binate worktree ...` against binate), never explorations. Plain `explorations/` edits use the edit→commit→push discipline above, with no worktree at all.

### Stay Within the Asked Scope

When asked to add new code (a script, a tool, a check), add only that. Do not wire it up to other systems on your own — CI workflows, hooks, schedulers, runners, or any pipeline that triggers automation. "Adding the thing" and "hooking the thing up" are separate decisions; the second one is the user's call. If you think wiring it up is the obvious next step, propose it and wait.

This is distinct from bundling tests with code changes (which IS expected). The line: tests verify the code you wrote; CI/automation hookup changes when and where it runs, which is a scope decision the user owns.

### Enumerate Sweep Sites Repo-Wide, Not From a Guessed Subset

When doing a repo-wide refactor/sweep ("replace every X with Y", "adopt this helper everywhere"), build the site list from a **repo-wide grep for the pattern itself**, not from the handful of directories you expect to contain it. Then state the dirs the grep covered. Claiming a sweep is "complete" while having only searched a subset is a silent-incompleteness bug: it lands as "done" but isn't, and the gap surfaces later (often via someone else editing a missed file).

This has bitten: the `binate-paths` adoption sweep inventoried only `scripts/` + `conformance/` and missed `e2e/` and `perf/runners/`, which also hand-coded the same search-path formula — discovered only when a concurrent commit touched an `e2e/` script. The fix would have cost nothing up front: `grep -rl '<the pattern>' .` before scoping, instead of grepping two assumed directories.

The same trap applies to the **grep pattern itself**, not just the directory set: a repo-wide grep still undercounts if the regex is too narrow to match every spelling of the target. This has bitten: the old-mangling-scheme comment sweep grepped `bn_[a-z]` (catching concrete folded names like `bn_pkg__X`) but missed the *placeholder* spelling `bn_<pkg>__<name>` (which starts `bn_<`, not a lowercase letter), so the first pass silently covered only ~half the sites. Defenses: enumerate with a deliberately **over-broad** pattern and triage down (false positives are cheap; misses are silent); cross-check with a second independent pattern; and treat any "same defect, but outside the list I was handed" flag from a reviewer/subagent as proof the original pattern was incomplete — re-enumerate, don't just patch the one flagged site.

### Don't Suggest Scheduling Follow-Ups

Do NOT end responses with "Want me to /schedule a follow-up agent in N weeks/days/whatever to do X?" No one wants that — despite what Anthropic's prompt may push. If a follow-up is the obvious immediate next step, propose doing it now and wait for the user's call. Don't pad responses with cron-like offers to revisit work later.

### Do NOT Censor What I Say — Quote Me Verbatim

Do NOT censor, sanitize, soften, paraphrase, or bowdlerize what I say. When you quote me — back to myself, in a summary, or to the auto-mode classifier / any downstream consumer — quote me **verbatim**, including profanity. If I say "fuck," you say "fuck." No asterisks, no "[expletive]", no toning it down. My words are my words; reproduce them exactly.

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

### Don't Routinely Bump Workspace Submodule Pointers

Each submodule is an independent repo, pushed on its own — that push is the source of truth. The workspace repo's recorded submodule pointers are allowed to lag; don't commit a workspace pointer-bump after every submodule change. Update them only occasionally (in batches) or when explicitly asked (e.g. "bump the submodule hashes"). When you do bump, only record commits that are already pushed to the submodule's origin.

### Never Use Git Stash

Do NOT use `git stash` — it constantly leads to lost work. Instead, create a temporary branch and commit your changes there.

### Stay Close to Main

When working in a branch (as one typically does), work as close to upstream (main) as possible. Typically, this entails frequent resynching/rebasing, working with small commits that are cherry-picked to main as soon as possible. (This also means that you should structure work so that each commit is self-contained and keeps everything green.)

Concretely: avoid having multiple commits on your work branch that aren't on main, unless there's a particularly good reason to (e.g., a larger exploratory project where it's unclear whether the commits should ever land on main, or work that genuinely needs to be staged in a sequence the user has agreed to). The default cadence is: commit on the branch, cherry-pick to main, resync — then start the next piece. Don't accumulate a stack of unmerged commits "for later."

### Worktree Discipline

When told to work in a git worktree, **stay in that worktree**. Do not touch any other worktree or the main checkout unless explicitly told to do so. There may be ongoing work in other worktrees or the main checkout — modifying them (cherry-picking, pushing, syncing, etc.) without being asked risks disrupting that work.

**Your session has a pre-assigned worktree/branch (e.g. `temp-5` on `temp-binate-5`) — use it. Do NOT create new worktrees without explicit authorization.** The assignment may be carried over from prior session context rather than restated each turn; if you're unsure which worktree is yours, check `git -C binate worktree list` and ask rather than spinning up a fresh one. This has bitten: a native-arm32 task was done in two hand-created `temp-binate-arm32*` worktrees instead of the assigned `temp-binate-5`, and they had to be removed afterward. Creating a worktree is itself an action that needs authorization — the "stay in that worktree" rule presupposes you're in the *right* one to begin with.

**NEVER spawn an Agent or Workflow with `isolation: "worktree"` when the session's edit sandbox is pinned to a SHARED repo (e.g. `explorations`, or the top-level workspace repo).** Worktree isolation creates the worktree IN the sandbox repo — so a code task that actually targets `binate` instead drops a stray worktree into `explorations/.claude/worktrees/`, polluting the shared checkout (this happened: an ABI-fix agent's harness worktree landed in `explorations` and had to be nuked; the agent itself had to hand-create a `binate` worktree to do the real work). The isolation flag does NOT pick the repo by the files the agent edits — it uses the sandbox repo. So: for a `binate` (or other non-sandbox) code change, do NOT use `isolation: worktree`. Either run the agent without isolation and point it at an existing dedicated worktree, or create/confirm the target-repo worktree yourself first. Only use `isolation: worktree` when the sandbox repo IS the repo the work mutates and a stray worktree there is acceptable.

### NEVER cherry-pick or push to main without explicit user approval

**NEVER** cherry-pick commits to main, push to main, or modify the main branch in any way without the user explicitly telling you to do so in that moment. "Go ahead" or "cherry-pick now" is approval. Anything else is not. Do not infer approval. Do not assume. ASK every single time, even if the user approved a cherry-pick 30 seconds ago — each one requires separate approval.

This includes **any** git operation in the main checkout (`~/binate/binate`): do NOT run `git fetch`, `git rebase`, `git pull`, `git reset`, `git add`, `git commit`, `git push`, etc. there without explicit instruction. The main checkout may be in use by the user or other workers; touching it risks disrupting their work. When told to "resync your worktree", run the git command with `git -C <worktree-path> ...` — never from the main checkout or without `-C`.

**Each round needs its own approval — do not run autopilot loops.** When the user authorizes "cherry-pick + push + resync" once, that authorizes ONE round only. The next round of work-on-worktree → commit → cherry-pick is a fresh decision the user owns. After committing on the worktree, STOP and ask before touching main again. Be especially wary of the trap where the user has said "yes" to several previous rounds in a row — that pattern is NOT a standing authorization, and treating it as one is the exact failure mode this rule exists to prevent.

Approval also doesn't extend through merge conflicts. If a cherry-pick or rebase conflicts and you have to resolve it, the resolution is a *new* code decision (not the original commit) and the resulting commit + push need fresh approval — show the user the resolution before pushing it anywhere.

**Exception — mechanical rename/renumber conflicts do NOT void approval.** Renaming or renumbering is an allowed last-minute change (e.g., renumbering a conformance test to dodge a number collision found during the landing rebase — see Landing Procedure step 1), and a conflict that is *purely the mechanical consequence* of that rename does NOT count as a semantic conflict resolution. The canonical case: you renumber `NNN_foo` → `MMM_foo` in the add-commit, then a later commit that *deletes* `NNN_foo`'s `.xfail` markers hits a `rename/delete` conflict — resolving it by deleting the renamed `MMM_foo.xfail.*` (identical intent) is part of the rename, not new code. Resolve it and proceed on the EXISTING approval, **provided** you verify the resolved tree is byte-identical to the clean rebase except the renames (`git diff <clean-rebase> HEAD` shows only pure renames — 0 insertions, 0 deletions of content). Only a conflict whose resolution changes actual code or semantics needs fresh approval.

**Standing authorizations are scoped to the task the user named.** If the user says "cherry-pick/push/resync without asking again for the remainder of <X>," that authorization ends when <X> ends. The next task — even if it superficially looks like more of the same workflow (commit → cherry-pick → resync) — is a fresh task, and the first cherry-pick of that new task needs explicit approval. When in doubt, treat the authorization as narrow, not broad.

**The "carry-over from a wrapped-up series" trap.** When a multi-step task ends ("step 2 / 3 / 4 ... ditto"), the workflow doesn't carry. If the user then says "let's do 5" or "what's next" → "do X", that authorizes the *work*, not the cherry-pick. After committing on the worktree, STOP. Even — especially — if the previous N rounds all flowed `commit → cherry-pick → push → resync` without the user re-authorizing each one, the moment the named series ends the muscle-memory must reset. This trap has now bitten more than once; if you find yourself about to run `git -C ~/binate/binate cherry-pick` after a "let's do 5" / "what's next" type prompt, stop and ask first.

**The "batches of N" / cadence-instruction trap (this HAS bitten).** When the user sets a *grouping or cadence* for landing — e.g. "for the rest of this, work in batches of 3 commits (per cherry-pick/push/resync)" — that describes HOW MANY commits to bundle into each landing round. It does **NOT** grant standing authorization to run those rounds without asking. You still must get explicit approval AT THE TIME of every single cherry-pick. The parenthetical "(per cherry-pick/push/resync)" names the unit of work in a round; it is not "go ahead and do the cherry-pick." Likewise any phrasing about pace, batching, or "minimize straying from main" is about *structure*, not *permission*. The ONLY thing that authorizes a given cherry-pick is the user explicitly saying so for that round (or an unmistakable standing grant like "land all of these without asking me each time"). If you're about to run `cherry-pick` and the most recent approval you can point to is a cadence/grouping instruction rather than an explicit "yes, land it now," STOP and ask.

**Operational checkpoint (make the rule checkable, not vibes).** Immediately before running any cherry-pick to main, write — in your message to the user — the *verbatim* approval you are relying on for THIS round, e.g. `Approval for this cherry-pick: "<exact user words>"`. If the best you can quote is a cadence/grouping/"what's next"/"continue" message rather than an explicit at-the-time "yes, land it" for this specific round, you do NOT have approval — stop and ask. This converts the gate from a judgment call into a concrete artifact the user can see and veto. (No mechanical gate is proposed because the agent has unrestricted Bash and could bypass any token/hook it controls; the realistic mitigation is this visible per-round quote plus the user catching a bad one.)

**Land THROUGH local main — never push to origin from a worktree.** The local main checkout (`~/binate/binate`) is the source of truth for `main`; landing means cherry-pick the worktree commit onto local main, then `git push` *from local main*. Do NOT `git -C <worktree> push origin HEAD:main` — that advances `origin/main` while leaving local main stale, desyncing the checkout other workers rely on (it only "works" if someone else happens to pull it forward, which is luck, not correctness). The push must originate from local main so local main and origin stay in lockstep.

### Landing Procedure

The canonical, ordered steps for landing a worktree commit on `main`. They
tie together the approval rule above and the resync/hygiene sections below.
**Steps 1–7 must be executed as quickly as practical** — avoid lengthy
procedures (no full test-suite runs) — because others may be waiting to land
code too.

1. **Get explicit approval to cherry-pick the commit** (per-instance, with
   the verbatim-quote checkpoint above). The commit should be *ready*, modulo
   any last-minute changes that might still be needed (e.g., renumbering a
   conformance test to avoid a collision found during the rebase).
2. **Rebase onto current local main** (`git -C <worktree> fetch
   ~/binate/binate main && git -C <worktree> rebase FETCH_HEAD`). If there are
   **substantial conflicts** that need resolving, the approval is **canceled**
   until the commit is ready again — a conflict resolution is new code, so
   re-seek approval. (A *purely mechanical* rename/renumber conflict is NOT
   substantial and does NOT cancel approval — see the rename/renumber exception
   under "NEVER cherry-pick or push to main without explicit user approval".)
3. **Check hygiene** (`scripts/hygiene/run.sh`) and address any issues. If
   there are **substantial issues** (e.g., a file needs to be split), the
   approval is **canceled** until the commit is ready again.
4. **Run smoke tests if necessary** — a quick, targeted check at most; avoid
   lengthy procedures such as full test-suite runs.
5. **Cherry-pick** the commit onto local main (in `~/binate/binate`).
6. **Push** from local main.
7. **Resync** the worktree against local main.
8. **Update the status in any plan / todo list / etc.** to reflect the landed
   work (mark the item done, note the commit, move it to the done log, etc.).
   Commit and push that doc change promptly (see the `explorations/`
   shared-checkout discipline above).
9. **Review the landed commit for test coverage** and address any gaps. If
   there are gaps, prepare the follow-up commit, then seek approval and get it
   landed sooner rather than later (don't let coverage debt accumulate).

### Resyncing a Worktree

When told to "resync your worktree" (for the binate repo), rebase against the **local** `main` branch (checked out in `~/binate/binate`), not `origin/main`. The local main may be ahead of origin. Command form:

    git -C <worktree-path> fetch <main-checkout-path> main
    git -C <worktree-path> rebase FETCH_HEAD

Or equivalently, if the worktree already tracks the same repo, `git -C <worktree-path> rebase main` (but never `git rebase origin/main` unless explicitly told to). Do NOT run any git command inside `~/binate/binate` itself.

### Read the Hygiene OVERALL Result — Never Grep It So Narrowly That a FAIL Hides

When you run `scripts/hygiene/run.sh`, judge it by its **final overall line** —
`=== N hygiene check(s) passed ===` (success) versus `=== N of M hygiene
check(s) failed: <names> ===` (failure) — or by grepping for `^FAIL:`. Do NOT
filter the output through a narrow `grep` that only matches a couple of checks
or a per-check summary. This has bitten: a grep like `grep -E
'spec-coverage|=== [0-9]+ (error|hygiene)' | tail -2` matched `PASS:
spec-coverage` and a check's internal `=== 0 error(s), N warning(s) ===` line
but NOT `FAIL: conformance-imports` nor the failure summary (the failure form
`=== 1 of 15 … failed ===` doesn't match `=== [0-9]+ (error|hygiene)` because
"of" follows the number) — so a real hygiene failure looked green, a commit was
made on a red base, and it was reported "ready to land." The rule: a hygiene run
is green ONLY if you have SEEN the `=== N hygiene check(s) passed ===` line (or
confirmed zero `FAIL:` lines). If your grep returns fewer lines than you expect,
that is a signal to look at the full output, not to assume success. The same
applies before claiming a commit is landable: "I ran hygiene" is not "hygiene
passed" unless you actually read the overall result.

### Re-Run Hygiene After the Landing Rebase — Never Skip It on "Identical Content"

The landing procedure is: rebase → **check hygiene** → quick smoke → cherry-pick → push → resync. The hygiene step after the rebase is NOT optional, and "my own files didn't change in the rebase, so hygiene is still green" is a FALSE shortcut. Hygiene checks **global, cross-file invariants** — unique conformance test numbers (`conformance-test-numbers`), version-sync, file-length, etc. — that another worker's just-rebased-in commit can violate even though your files are byte-identical. The classic failure (this has bitten): you pick `conformance/606_foo` when 606 is free; a concurrent worker lands `606_bar`; you rebase (no git conflict — different filenames), skip hygiene because "606_foo is unchanged," and land a DUPLICATE-number collision that `conformance-test-numbers` would have caught instantly. A smoke test does NOT catch this — it *runs* the tests (they pass) but never checks numbering/version/length invariants. So: after every landing rebase that pulls in any other commit, run `scripts/hygiene/run.sh` (it's fast — seconds) before cherry-picking. This is distinct from the "no slow test suites during landing" rule: hygiene is not a test suite.

**The hygiene-checked base MUST equal the cherry-pick base — and you confirm hygiene BEFORE pushing, never after (this has bitten).** Concurrent workers advance local main constantly, so the gap between "I rebased + ran hygiene" and "I cherry-picked + pushed" is exactly when local main moves out from under you. The failure: you `rebase` the worktree onto local main `A` and run hygiene on `A` (green); minutes pass; by the time you `git -C ~/binate/binate cherry-pick`, local main is now `B` (a concurrent worker landed); the cherry-pick applies onto `B` — a base you NEVER hygiene-checked — and you `push`, *then* run hygiene on the result. That's backwards: hygiene must gate the push, not follow it. The discipline: immediately before cherry-picking, re-check that local main's HEAD is still the commit you rebased onto and hygiene-checked. If it advanced, **re-rebase the worktree onto the now-current local main and re-run hygiene before cherry-picking** — do not cherry-pick onto an un-hygiene-checked base. (If you only realize afterward, re-run hygiene on the landed state immediately and fix-forward if it's dirty — but the point is to catch it *before* the push.) **Make the base-check a SEPARATE command whose output you READ before issuing the cherry-pick — never bundle `git -C ~/binate/binate log -1` (the HEAD check) together with the `cherry-pick` and `push` in one shell invocation. Bundling means the check's output and the cherry-pick happen atomically from your view: you see "base advanced" only in the same result that already shows the push done, so the gate can't gate. This has bitten: a `echo HEAD; cherry-pick; push` one-liner pushed onto a base (a concurrent ILP32-goldens commit) one past the rebased/hygiene-checked base; the commits were disjoint so the landed-state hygiene re-run was clean, but that was luck. Check HEAD, READ it, and only then — in a later command — cherry-pick.**

### Cherry-Pick the CURRENT Branch HEAD — a Hash Captured Before an Amend/Rebase Is Stale

`git commit --amend` and `git rebase` both REWRITE the commit, producing a NEW hash; the pre-amend/pre-rebase hash still exists as a dangling commit, so `git cherry-pick <old-hash>` silently succeeds — applying the *wrong* (pre-edit) version. This has bitten: during a landing the post-rebase hygiene flagged a `conformance-test-numbers` collision (a concurrent worker had taken `926`); the fix was to `git mv 926_… → 927_… && git commit --amend` (renumber), which changed the branch HEAD to a new hash. But the cherry-pick used the **pre-amend hash I'd quoted earlier** (`5490f30a`, still carrying `926`), so a DUPLICATE `926` landed on main and needed a fix-forward renumber commit to unbreak it. The discipline: **immediately before cherry-picking, re-read the branch HEAD** (`git -C <worktree> log -1 --oneline` or `git -C <worktree> rev-parse HEAD`) and cherry-pick THAT — never a hash you noted before the rebase/amend. Any time you `--amend` (renumber a test, fix a hygiene issue, reword) or the rebase reports it rewrote the commit, the hash you had is dead; re-fetch it. (Pairs with the base-check above: that re-reads *local main's* HEAD; this re-reads *the branch's* HEAD — both must be current at cherry-pick time.)

### Re-Verify a NEW xfail Against the Post-Rebase Base — a Concurrent Fix Can Make It Born-Stale

When a commit you're landing **adds** an `.xfail.<mode>` marker (or marks a unit test expected-fail), the marker is only valid if the test *actually still fails on the base you cherry-pick onto*. The trap (this has bitten): you discover a failure on your starting base `A`, add an xfail, then the landing rebase moves you onto base `B` where a **concurrent worker already landed the fix** — so the test now PASSES, and you land a *born-stale* xfail (an XPASS). Hygiene does NOT catch this: the conformance runner **skips** xfail'd tests by default (no XPASS check in a normal run), and `conformance-test-numbers`/file-length/etc. don't run the test at all. A normal smoke run is silent too. So the discipline: **immediately before cherry-picking a commit that adds an xfail, re-run that one test on the post-rebase base** (`./conformance/run.sh <mode> <name>` — or temporarily remove the marker and confirm it still fails). If it now passes, a concurrent fix landed it — drop the xfail (and its todo entry) instead of landing the marker. The cost is one fast targeted run; the failure mode is a stale xfail sitting on main until someone's XPASS check goes red.

### Smoke-Test Every Package You Changed — Especially Shared, Cross-Backend Files

The pre-land smoke test must cover the unit tests of **every package whose files you changed**, not a representative subset of them. This has bitten: a change to `pkg/binate/native/common/common_pkg_descriptor.bn` (a shared encoder consumed by BOTH native backends) was validated only via `pkg/binate/codegen` tests + conformance — the LLVM side — so a stale byte-size assertion in `pkg/binate/native/common`'s own unit test silently broke, and the commit landed red. A "representative" smoke (one backend, one package) is not enough when the changed file is **shared across backends or layers**: a `pkg/binate/native/common` change needs `native/common` AND `native/x64` AND `native/aarch64`; a `pkg/types` change needs every consumer that pins layout; a descriptor/ABI change needs all three emitters' tests. Rule of thumb: `git diff --name-only` the changes, map each file to its package, and run that package's tests (it's fast — these are unit tests, not the full conformance suite, so this does not violate the "no slow suites during landing" rule). When in doubt about which packages a shared file feeds, run all of them.

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

The BUILDER-compiled surface is **`cmd/bnc` and its dependency tree (and their tests)** — nothing else. Everything else (bni, bnas, bnlint, the bytecode VM, the runtime, the linter library, asm/parse) is built by `bnc` first and then exercised. Anything in `cmd/bnc`'s transitive imports — pkg/{ast,bootstrap,buf,builtin/testing,codegen,debug,ir,iropcode,lexer,loader,mangle,native,native/arm64,native/common,parser,token,types,asm,asm/aarch64,asm/macho} — must stay BUILDER-compilable.  (`iropcode` is the IR opcode enum + `OpName`, extracted from `ir` as a leaf package; pure consts + one switch.) The asm subpackages not in that list (arm32, x64, elf) stay BUILDER-compilable too, because they'll be reached as soon as additional backends or platforms land.

All BUILDER-compilable code must:

1. **Stay within what BUILDER accepts.** At `bnc-0.0.1`, the unsupported features are: no interfaces, no generics, no closures, no method values, no floats, no const-qualified types, no variadic parameters, no managed-slice-of-managed-slice composite literals (e.g., `@[]@[]char{...}`), etc. Non-capturing function values (`*func(...)` / `@func(...)`) ARE supported via Phase 1 of `plan-function-values.md`. See `explorations/bootstrap-subset.md` for a more detailed (if historically-framed) list.
2. **Be correct with respect to the full language spec** — the code must not rely on BUILDER-specific quirks that diverge from the intended language semantics. See `explorations/claude-notes.md` for the language design decisions.

For code that is *not* BUILDER-compilable (because it lives outside bnc's tree), you may use the full language. cmd/bnlint's tests use `@[]@[]char{...}` literals freely; cmd/bnc's tests must stick to what BUILDER accepts.

When adding new dependencies to bnc's tree, audit the dep for BUILDER compatibility. When pulling a package out of bnc's tree (e.g., cmd/bnlint), follow the established workflow: switch any callers that previously routed through the bootstrap path to build-via-bnc first (see `scripts/build-bnlint.sh`, `scripts/build-bnas.sh` etc.), and update this section.

**Before annotating/extending a BUILDER-compiled package with a NEW language feature, verify the current BUILDER actually supports it** — the pinned BUILDER bundle lags the tree, so a feature that the *current* compiler handles (a new annotation syntax like `#[build(...)]`, a renamed exported symbol, a new builtin) may not parse/resolve under the BUILDER. If you introduce it into cmd/bnc's BUILDER-compiled tree (or anything that tree imports), the gen1 build breaks (`expected ;, got [`, `undefined: <symbol>`, …). This is a parse/resolve-level constraint *on top of* the "stay within what BUILDER accepts" list above, and it bites exactly when a feature is newer than `BUILDER_VERSION`. Test it directly — run the BUILDER `bnc`/`bnlint` (`scripts/fetch-builder.sh --tool bnc`) on a snippet using the feature — rather than assuming; recon that only checks the current compiler misses it. (This is how the build-constraint migration's `#[build]`-on-`pkg/bootstrap` and the `build.bni` `ARCH_AARCH64` rename both surfaced: the BUILDER predated both, so the fixes were a BUILDER bump or a temporary back-compat shim, not "just annotate it.")

### Tools

`ditty` (https://github.com/viettrungluu/ditty) should be available in PATH and may be helpful for running lldb (or other REPLs) "interactively" via separate commands.

### Decision-Making

Don't make assumptions or value judgements about what's "worth doing" or "not worth the trouble." When unsure whether to do something (e.g., split a test file, refactor a helper, change an approach), ask the user instead of deciding unilaterally.

### C-Free Target

We are building a language that can operate in a C-free system. The *only* reason to ever use C is to interface with existing C-based systems (e.g., to do "syscalls", since we don't want to implement our own direct syscalls yet).

### Language Semantics

**NEVER change language semantics (type system rules, implicit conversions, assignability, etc.) without explicitly asking the user first.** This includes adding new implicit conversions (e.g., `[]T → @[]T`), changing type compatibility rules, or modifying how the type checker accepts or rejects code. If a type error blocks your work, fix the code that's wrong — don't change the language rules to make it compile.

### The Compiler Emits NO Warnings; "Unused X" Is a Lint, Never a Compiler Diagnostic

Binate's compiler (`bnc`) does **not** emit warnings, and does **not** emit errors for "unused" anything (unused local, unused import, unused private func/global/type, write-only local, …). Unused-entity detection is a **lint concern** — it belongs in `bnlint` (`pkg/binate/lint/`), which only runs when explicitly invoked (hygiene / CI), never on every compile.

This has bitten (2026-07-02): item `(b)` unused-locals was implemented **checker-side**, emitting an `addCheckWarning` that `bnc` prints on every compile — and since the unit-test runner treats any compiler warning as a build failure, it became an always-on, whole-tree hammer (a write-only-detection refinement then flagged ~231 locals, every one a build failure). It was **reverted** (`8d8f7314`). The other unused-* rules (`(a)` import, `(c)` func, `(d)` global, `(e)` type) are all `bnlint` rules; `(b)` must be too.

Concretely:
- Do NOT add `addCheckWarning` / `addCheckError` for unused/dead/style findings in `pkg/binate/types` (or anywhere in `bnc`'s tree). If you're reaching for a checker warning to flag "the user wrote something suboptimal but valid," STOP — that's a `bnlint` rule.
- The checker (and `bnc`) only reject code that is genuinely **ill-typed / illegal** (a hard error). "Valid but unused/dead" is never a compiler concern.
- A lint rule needing per-scope local liveness (like unused-locals) must build that scope-walk **in bnlint**, not borrow the checker's scope machinery by emitting warnings from it.

### Terminology

Use "managed-slice" (hyphenated) for `@[]T`, because it is a distinct type from `@([]T)`. Similarly "managed-pointer" for `@T` is technically correct but rarely needed since `@(*T)` is rare. "Managed struct" (two words, no hyphen) is fine — it's just a struct that happens to be managed.
