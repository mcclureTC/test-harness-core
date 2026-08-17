# Update — `TestRunTally`

The counts, and what they add up to. The read model of a run, and the first
implementation of `I_TestRunStatus`.

Separated from the suite because "what does this run amount to" is its own
question with its own consumers — CI, an HMI, a sequencer aggregating several —
and none of them should be handed the ability to start or stop anything.

## Incremental, not recomputed

Every figure is folded in as each test completes. The original derived its counts
by walking every slot each cycle, which is the O(tests × capacity) per-cycle cost
R6 exists to remove — and then exposed none of the result, so nothing could ask
whether a run had finished.

`_Recorded` is tracked separately from `TestCount`. They differ while a run is in
progress and after one that was abandoned, and conflating them would make an
abandoned run look complete.

## `ST_TestRunStatus`, the flat symbol

`Plan.md` §6 requires the figures to be readable as plain symbols as well as
through the interface, so ADS, TwinCAT HMI and OPC UA reach them without RPC.
That is the portable path and the one CI polls.

One struct holds them; `I_TestRunStatus` reads its members. There is no second
copy to keep in step.

## Aggregation

Applied in order, and deliberately **order-insensitive**:

```
any Errored -> Errored     the run cannot be trusted
any Failed  -> Failed
all Skipped -> Skipped
otherwise   -> Passed
```

`Errored` outranks `Failed` because a framework or environment problem casts doubt
on every other result. A test that errored then failed weighs the same as one that
failed then errored.

**An empty run reports `Passed`.** Nothing ran and nothing was skipped — a project
may legitimately deploy a suite with nothing registered, and a red build for that
would be noise. `TestCount` of 0 is the signal for anyone who cares.

**A framework error raises `ErrorCount` without touching `TestCount`,** so the
per-test figures still add up while the run is unmistakably untrustworthy. This is
how a teardown overrun turns the run red without altering any test's verdict.

**A non-terminal status is ignored, not bucketed.** There is no sensible home for
`Running`, and inventing one would let a half-finished test look like a result.

## Two self-corrections

**`Complete` → `Finish`.** I wrote a `Complete` method beside an `IsComplete`
property — precisely the collision `Plan.md` §2.1 bans by name. `TestJournal.Complete`
is still outstanding for the same reason (§11.1).

**`Record(Status : ...)` → `Record(Verdict : ...)`.** A parameter named `Status`
shadows the `Status` property on the same object, silently.

## Files

| File | Change |
|---|---|
| `POUs/TestRunTally.TcPOU` | **New.** `SetLabel`, `Begin`, `Record`, `RecordFrameworkError`, `Finish`, plus `I_TestRunStatus` |
| `DUTs/ST_TestRunStatus.TcDUT` | **New.** The flat symbol |
| `TestHarnessCore.plcproj` | Two new files |

## Not compiled, and nothing in this library has ever executed

`check_st.py` clean. XML, line endings, paren balance, `<Folder>` ordering,
property declarations, and member-access-on-call-result all checked mechanically —
the last of those added after `UPDATE_12`.

## Cases for the verifier

**Counting**
- Three tests recorded `Passed` → `PassedCount` 3, others 0.
- One of each terminal status → each count is 1.
- `Record` with `Running` or `NotStarted` → no count changes and `_Recorded` does
  not advance.
- `TotalDuration` is the sum of the elapsed times passed in.

**Aggregation**
- One `Failed` among passes → `Failed`.
- One `Errored` among failures → `Errored`, whichever order they were recorded in.
- All skipped → `Skipped`.
- All passed → `Passed`.
- Nothing recorded → `Passed`, `IsComplete` TRUE.

**Framework errors**
- `RecordFrameworkError` raises `ErrorCount`, leaves `TestCount` and
  `PassedCount` untouched, and makes `Finish` yield `Errored` even when every
  test passed.

**Lifecycle**
- `Outcome` is `NotStarted` before `Begin`, `Running` after it, terminal after
  `Finish`.
- `IsComplete` is FALSE until `Finish`.
- A second `Finish` changes nothing.
- `Begin` after `Finish` clears every figure but keeps the label.

**Struct symbol**
- `Status` returns the same values the individual properties do, after each of
  the above.

## Left to build

`TestReporterFanout`, then `TestSuite<Capacity>`. Then reporters (step 4) and
`TestRunSequencer` (step 6).
