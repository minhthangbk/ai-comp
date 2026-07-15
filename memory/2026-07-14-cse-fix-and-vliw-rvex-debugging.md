# 2026-07-14 — ai-comp CSE fix + vliw_rvex (real LLVM VLIW backend) debugging

## What happened

Started by asking "làm gì để test compiler này" (ai-comp), which led to
a much bigger arc: an ai-comp compile-time bug fix, then a pivot to
`/data/thangnm35/vliw_rvex/` (a separate, real LLVM-based VLIW backend
for the ρ-VEX architecture) to evaluate whether ai-comp's optimization
ideas could be ported into something that targets real hardware. Found
and fixed 3 real bugs there, built a regression test suite from
scratch, and did (but didn't finish) two scheduler-heuristic porting
investigations.

## Part 1 — ai-comp: CSE pass O(n²) compile-time blowup

`python3 programs/tree_hash.py` (default challenge size: forest-height
10, rounds 16, batch 256, ~4096 unrolled iterations) **never finished
compiling** — observed >2.5h with no completion.

Root cause: `compiler/passes/cse.py`'s value-number keys recursively
nested each operand's full value-number tuple, so key size (and
hash/compare cost) grew with program length instead of staying
O(#operands) — effectively O(n²) total.

Fix: hash-cons each expression's shallow `(opcode, operand-tokens[,
epoch])` key to a compact `("id", n)` token via a per-pass interning
table, and use that token — not the operand's full recursive key —
when building the parent expression's key.

Verified: 630/630 compiler tests pass, `tests/submission_tests.py`
correctness + all speed thresholds pass. Compile time: **18s** (was
>2.5h). Result: **1137-1138 cycles**, beating every recorded threshold
in that file, including the hardest (`opus45_improved_harness < 1363`).

Committed as `37bb94d` on branch `memory/cocoindex-code-setup`.

Also redirected `uv`/pip caches and PyTorch install to `/data` (root
disk was at 13GB/468GB free on this shared machine) — see `~/.bashrc`
exports for `UV_CACHE_DIR` etc.

## Part 2 — vliw_rvex: found and fixed 3 real bugs

`/data/thangnm35/vliw_rvex/` is a modified **legacy LLVM (3.3svn)**
source tree with a real backend for the **rvex (ρ-VEX)** VLIW
architecture (TU Delft open-source reconfigurable VLIW soft-core,
4-issue, functional units P0-P3). Remote: `github.com/vanhopenai/
vliw_rvex`. This is architecturally very different from ai-comp's own
simulated VM (subword SIMD in 32-bit registers vs. ai-comp's VLEN=8
multi-word vectors; real LLVM IR/SelectionDAG/TableGen framework vs.
ai-comp's hand-rolled Python IR).

### Toolchain bootstrap (no root on this machine)
Missing `python2`, `zlib1g-dev`, `libedit-dev`. No sudo. Fetched via
`apt-get download <pkg>` (no root needed) + `dpkg-deb -x <deb> <prefix>`
into `/data/thangnm35/.local-tools/python2-root/` (userspace, no root
needed for extraction either — runtime `.so`s were already present
system-wide, only `-dev` headers/symlinks + the `python2.7` interpreter
itself were missing). Build:
```bash
export PATH="/data/thangnm35/.local-tools/python2-root/usr/bin:$PATH"
export LIBRARY_PATH="/data/thangnm35/.local-tools/python2-root/usr/lib/x86_64-linux-gnu:$LIBRARY_PATH"
export CPATH="/data/thangnm35/.local-tools/python2-root/usr/include:$CPATH"
cd /data/thangnm35/vliw_rvex/lbd/build && make -j"$(nproc)" vex_llc
```
`vex_llc` (and `FileCheck`, built separately: `make FileCheck`) were
already present from a prior build (2026-03-06); incremental rebuilds
after single-file edits take ~55s on this 32-core machine.

### Bug 1+2 (commit `2b82489`): default-CPU segfault + silently-disabled VLIW mode
`rvexSubtarget.cpp` defaulted to CPU name `"rvex32"`, which isn't a
defined `Proc` in `rvex.td` (only `"rvex"`/`"rvex-vliw"` are). This
silently left `InstrItineraryData` empty, and
`DFAPacketizer::canReserveResources()` dereferences it with no
`isEmpty()` check → segfault on every default-CPU compile. Separately,
`"rvex"` (unlike `"rvex-vliw"`) doesn't set `FeatureVLIW`, so even
`-mcpu=rvex` explicitly never bundled more than 1 instruction/packet —
silently defeating the whole point of the backend, no warning at all.
**Fix:** default to `"rvex-vliw"`.

### Bug 3 (commit `356fe86`): packetizer bundle-overflow beyond issue width
Found only after testing with >4 mutually-independent instructions
(the original 4-instruction test wasn't enough to expose this). ρ-VEX
is *reconfigurable*; functional-unit config is meant to load from an
external `-config <file>` (a `config/` dir the README documents but
that doesn't actually exist in the repo). When the file can't open,
`read_config()` correctly signals failure (`return 1`), and the caller
correctly skips rebuilding the itinerary data on failure — but
`rvexBuildDFA(Stages)` (which builds the actual DFA *transition
table*) is called **unconditionally** regardless of that failure
signal, and used a broken 2-entry placeholder fallback (`FU=3` ×2,
only 2 of 4 units) that didn't correctly cap packets at the real
4-wide issue width. A test with 6 independent ALU ops packed all 6
into one bundle. **Fix:** made the fallback describe the real default
model (FU=1 for loads/stores, FU=15 for everything else) instead of a
placeholder.

**Debugging method that worked** for bug 3 (worth reusing): temporary
`fprintf(stderr, ...)` instrumentation directly in
`DFAPacketizer::canReserveResources`/`reserveResources`/`ReadTable`,
tagged with the `this` pointer to separate the scheduler's own
separate heuristic resource tracker (constantly resets, explicitly
commented "not precise") from the real final packetizer's tracker
(only active in `addPreEmitPass`). Removed via `git checkout --` before
committing the real fix (not `git stash` — stash grabbed uncommitted
new test files too and needed popping back).

### Regression test suite built from scratch
Backend had exactly **one** test before this session
(`tests/vliw4_ilp_legacy.ll`), with no RUN/CHECK directives at all —
that's *how* the 3 bugs above went unnoticed. Added:
- RUN/CHECK directives on the original test
- `vliw_loadstore_dep.ll`, `vliw_branch.ll`, `vliw_loop.ll` (correctness)
- `vliw_load_latency_fill.ll`, `vliw_load_latency_overflow.ll` (the
  overflow one is the actual regression test for Bug 3 — the others
  don't have enough parallelism to expose it)
- `vliw_load_priority_contention.ll`, `vliw_loop_unroll4_profile.ll`
  (from the porting investigations below)
- `tests/run_tests.sh`: minimal standalone runner (no `lit.cfg` wiring
  exists for this out-of-tree target) — greps each file's `; RUN:`
  line, substitutes `%s`, reports pass/fail.

Verified every fix's test actually fails against the pre-fix code
(temporarily restored via `git show HEAD~N:path > path`, rebuilt,
confirmed failure, restored fix) before committing — not just that it
passes now.

8/8 tests pass as of the last commit (`4503027`).

### Porting attempts (2 tried, both informative, neither landed a heuristic change)

**Attempt A — load-priority under contention: negative result.**
Tried porting ai-comp's `prioritize_load_unblock` idea (bias scheduler
to issue loads early to hide latency). Built adversarial tests where a
load competes with 5-6 other ready instructions (incl. two independent
long dependent chains with higher Height) for 4 issue slots. The load
won its slot every time tried — no demonstrated gap, no code change.

**Attempt B — realistic unrolled-loop profiling: real gap found, but
root cause deeper than expected, investigation paused.**
Per redirect to profile something real instead of guessing small
scenarios: compiled a sum-reduction loop unrolled by 4 (4 independent
loads + tree-reduce add per iteration). Bundle count showed an
apparent 1-cycle stall that looked fixable (`s23=v2+v3` looked movable
into an empty bundle since v2,v3 loaded early). But instrumented
scheduler tracing (`pickNodeFromQueue`, same fprintf-then-revert
method as Bug 3) showed `s23` genuinely isn't a ready *candidate*
until the very last scheduling steps — this is about
`TopReadyCycle`/latency-elapsed modeling in the top-down scheduler,
not a priority/tie-break bug like Attempt A. **Needs a focused dive
into `TopReadyCycle`/`MinLatency` tracking (rvexMachineScheduler.cpp,
`Top`/`Bot` boundary structs, ~line 240-270) before attempting a fix —
paused rather than guessing.**

## To reproduce / use again in a fresh shell

```bash
# vliw_rvex build env
export PATH="/data/thangnm35/.local-tools/python2-root/usr/bin:$PATH"
export LIBRARY_PATH="/data/thangnm35/.local-tools/python2-root/usr/lib/x86_64-linux-gnu:$LIBRARY_PATH"
export CPATH="/data/thangnm35/.local-tools/python2-root/usr/include:$CPATH"
cd /data/thangnm35/vliw_rvex/lbd/build && make -j"$(nproc)" vex_llc

# run the test suite
/data/thangnm35/vliw_rvex/tests/run_tests.sh

# ai-comp: verify the CSE fix / current cycle count
cd /data/thangnm35/ai-comp && source .venv/bin/activate
python3 tests/submission_tests.py
```

## Open items

- vliw_rvex: 5 local commits ahead of `origin/main`, not pushed.
- ai-comp: 1 local commit (`37bb94d`) on `memory/cocoindex-code-setup`,
  not pushed; `.claude/skills/ccc/` still untracked (user chose not to
  commit it when asked).
- Git identity was not configured on this machine at all (neither repo,
  neither local nor global) — set **locally per-repo** to
  `Thang Nguyen <minhthang711@gmail.com>` after asking once for
  ai-comp and reusing for vliw_rvex in the same session.
- vliw_rvex Attempt B (TopReadyCycle/latency modeling) is a real,
  reproducible, but *not yet understood well enough to fix* gap —
  next session on this thread should start there, not with a new
  synthetic microbenchmark.
- No `config/` directory exists in vliw_rvex despite the README
  documenting one (`config/rvex_W4_2`) — the Bug 3 fix works around
  its absence, but if the config-loading feature (reconfigurable
  issue-width) is ever actually needed, that directory would need to
  be created for real.
