# 2026-07-15 — vliw_rvex: deep dive into the scheduler NOP bubble (root cause found, fix reverted)

## What happened

Continuation of the vliw_rvex work from 2026-07-14 (see
`2026-07-14-cse-fix-and-vliw-rvex-debugging.md`). Picked up the paused
investigation into the 1-cycle-per-iteration wasted bundle in
`tests/vliw_loop_unroll4_profile.ll` (a 4-way unrolled sum-reduction
loop). Went through several rounds of diagnosis, each correcting the
previous one, eventually implemented a fix, verified it didn't
actually work, and reverted it. Ended with a clean working tree (same
6 commits as before) and a much deeper, precise understanding of the
scheduler's NOP-insertion mechanics recorded for next time.

## Investigation trail (in order, including the wrong turns)

1. **First hypothesis**: `s23=v2+v3` (uses two of the four loaded
   values, loaded early) looked movable into the empty bundle from
   reading the assembly alone. **Wrong** — instrumented tracing of
   `rvexMachineScheduler.cpp`'s `pickNodeFromQueue`/`TopReadyCycle`
   showed it genuinely wasn't a ready candidate that early.

2. **Second hypothesis**: found an explicit `NOP` MachineInstr between
   the last load and the branch-condition compare (`CMPLT`) in the
   packetizer's input stream, forcing a hard bundle boundary
   (`isrvexSoloInstruction(NOP) == true`). Grepped for `insertNoop`
   call sites, found `SchedulePostRATDList::FixupLoads()` in
   `rvexPostRAScheduler.cpp` unconditionally inserting a NOP after
   every load (comment nearby: `//FIXME ugly hack`). Wrote a fix
   (skip the NOP when the next real instruction doesn't read the
   load's result). **Build warned the function was "defined but not
   used"** — its call site is commented out
   (`rvexPostRAScheduler.cpp:313`). Fix targeted dead code, reverted.

3. **Third hypothesis**: found the real NOP source is the *pre-RA*
   `rvexMachineScheduler.cpp`'s `ConvergingrvexVLIWScheduler::pickNode()`
   (confirmed via `-print-after-all`: NOP first appears in the
   "Machine Instruction Scheduler" pass's own dump). Root cause:
   after `pickNodeFromQueue()` picks a "best cost" candidate `SU`, a
   loop over `SU->Preds` checks a genuine latency-hazard condition
   `(Latency + Pred->ScheduledCycle) > CurrentCycle`. When it fires,
   the code sets `SU->InsertNop = true` and schedules `SU` anyway
   (after a NOP) — **it never tries an alternative, hazard-free
   candidate already sitting in `Top.Available`** (e.g. `CMPLT`, which
   has no such conflict).

## Independent verification via Codex CLI (new capability discovered)

Found a real, already-authenticated OpenAI Codex CLI binary bundled
with the VSCode `openai.chatgpt` extension:
```
~/.vscode/extensions/openai.chatgpt-*/bin/linux-x86_64/codex
```
Ran `codex exec -s read-only --skip-git-repo-check -o <file> "<prompt>" < /dev/null`
to get an independent second opinion on the diagnosis and a proposed
fix, without feeding it my conclusion in a leading way. **Gotcha**:
must redirect stdin from `/dev/null` — otherwise it hangs forever
waiting for piped input even with a prompt given as an argument.

Codex independently confirmed the diagnosis and the fix direction,
and flagged concrete implementation risks: candidate probing must be
side-effect-free (don't mutate `InsertNop`/`NopDelay`/`PacketNooped`/
`ScheduledCycle` while just checking alternatives — only on the final
committed choice); `ScheduledCycle--` must fire at most once; the
JALR/LR special-case NOP logic must run only after the final candidate
is chosen; falling back to the `SU == NULL` path (global `isNoop`)
would never advance `TotalPackets` and could infinite-loop, so it must
not be used as an escape hatch.

## Fix implemented, verified ineffective, reverted

Implemented the alternative-candidate search per Codex's guidance
(side-effect-free probing via a new read-only `hasLatencyHazard()`
helper, single commit point, existing mutation logic untouched).
Built clean, all 8 existing tests still passed — but **the compiled
output for `vliw_loop_unroll4_profile.ll` was byte-for-byte identical
to before the fix**. The empty bundle was still there.

Traced why with full ready-queue-state instrumentation
(`Top.Available`/`Top.Pending` at every `pickNode()` call): `CMPLT`
had *already* been scheduled successfully, immediately after the last
load, **one step before** `s23`'s hazard even triggers. By the time
the hazard fires, `Top.Available` only contains `{s23, s01}` — both
hazarded, nothing to substitute. My fix was solving a real problem
(confirmed by Codex) that just isn't *this* problem.

**The actual mechanism**: `ScheduleDAGMI::scheduleMI()`
(`lbd/lib/CodeGen/MachineScheduler.cpp` — generic LLVM code, not
rvex-specific) inserts the hazard-triggered NOP `SU->NopDelay - 1`
positions *backward* from the hazarded SU's insertion point:
```cpp
unsigned delay = SU->NopDelay - 1;
for (unsigned i = 0; i < delay; i++) --CurrentTop;
TII->insertNoop(*BB, CurrentTop);
```
This retroactively places the NOP *before* whatever was already
scheduled (`CMPLT`, committed one step earlier), not immediately
before the hazarded instruction (`s23`) itself. Looks deliberate
(perhaps meant to represent "the stall conceptually happened during
already-scheduled cycles"), and is a different, deeper piece of logic
that hasn't been investigated yet.

Reverted the implemented fix (`git checkout --`); vliw_rvex working
tree is back to clean, same 6 commits / 8 passing tests as before this
session.

## To reproduce / use again in a fresh shell

```bash
# vliw_rvex build env (unchanged from 2026-07-14)
export PATH="/data/thangnm35/.local-tools/python2-root/usr/bin:$PATH"
export LIBRARY_PATH="/data/thangnm35/.local-tools/python2-root/usr/lib/x86_64-linux-gnu:$LIBRARY_PATH"
export CPATH="/data/thangnm35/.local-tools/python2-root/usr/include:$CPATH"
cd /data/thangnm35/vliw_rvex/lbd/build && make -j"$(nproc)" vex_llc
/data/thangnm35/vliw_rvex/tests/run_tests.sh

# independent review via Codex CLI
CODEX_BIN=$(echo ~/.vscode/extensions/openai.chatgpt-*/bin/linux-x86_64/codex)
"$CODEX_BIN" exec -C /data/thangnm35/vliw_rvex -s read-only --skip-git-repo-check \
  -o /tmp/codex_out.txt "<self-contained prompt>" < /dev/null
```

## Open items

- The real fix needs to target `ScheduleDAGMI::scheduleMI()`'s
  backward NOP-placement logic (`NopDelay - 1` positions back), not
  `pickNode()`'s candidate-selection loop. Not yet started — this is
  generic LLVM code shared by other uses of `ScheduleDAGMI`, so a fix
  here has a different (likely broader) blast radius than the
  rvex-specific `pickNode()` change that was reverted.
- vliw_rvex still has exactly the same 6 local commits as end of
  2026-07-14, not pushed to `origin/main`.
- No code changes landed this session — purely investigation +
  memory. The 3 real bugs fixed on 2026-07-14 are still the only
  vliw_rvex code changes so far.
