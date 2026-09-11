# Maintenance Guide

For changing TestHarnessCore itself. Read `Implementation_Guide.md` first if you
have not written a test with it.

- [1. Read this before touching anything](#1-read-this-before-touching-anything)
- [2. The object map](#2-the-object-map)
- [3. Decisions that will look wrong](#3-decisions-that-will-look-wrong)
- [4. The verifier](#4-the-verifier)
- [5. TwinCAT facts that cost time](#5-twincat-facts-that-cost-time)
- [6. Working practice](#6-working-practice)
- [7. Extending it](#7-extending-it)
- [8. Known gaps](#8-known-gaps)

---

## 1. Read this before touching anything

**Run `PRG_FloorCheck` and confirm `AllGood` is TRUE before trusting any other
number in the verifier.**

The verifier tests the library using the library. If the runner never advances,
every test reports `NotStarted` — which looks exactly like a wiring mistake. There
is no way to tell a framework defect from a setup error, because the thing
reporting the result is the thing under test.

`PRG_FloorCheck` closes that hole. It drives `TestRegistry`, `TestArbiter` and
`TestRunner` **by hand**, one step per scan, and writes what it finds into plain
variables — including that a body ran exactly once. Nothing between you and those
objects is itself under test.

Re-run it after any change to those three.

---

## 2. The object map

Twelve objects, each with one reason to change. The split is the point: one
function block doing all of it is how such a thing becomes unmaintainable.

| Object | Owns |
|---|---|
| `TestJournal` | one test's record — verdict, counts, clocks, budgets |
| `TestVerdict` | the assertion surface. Retargeted at each journal in turn |
| `TestRegistry<Capacity>` | how tests are stored and found |
| `TestDeadline` | what a deadline means |
| `TestArbiter` | ordering **policy** — which test is next |
| `TestRunner` | the test **lifecycle** — setup, body, teardown |
| `TestRunTally` | what "the run passed" means |
| `TestReporterFanout` | how sinks are attached and notified |
| `TestSuite<Capacity>` | nothing of its own. A façade over the six above |
| `TestRun<Capacity>` | where a run begins and ends |
| `TestStopwatch` | elapsed time |
| six pure functions | comparison, rendering, formatting |

**One `TestVerdict` for the whole suite, not one per test.** It is retargeted at
each journal as the runner loads it. A verdict carries several hundred bytes of
machinery and the journal fifty; allocating all of it per slot is most of what
makes a test framework expensive to instantiate.

**Only `TestRegistry`, `TestSuite` and `TestRun` are generic.** Everything else
holds interfaces, because generic instances cannot cross a boundary as anything
but an interface (see §5).

### The interfaces

Segregated by *who is allowed to do what*, not by convenience:

- `I_Test`, `I_TestSetup`, `I_TestTeardown`, `I_TestLocation` — implemented by
  test authors. The last three are optional and discovered at registration.
- `I_TestAssertions` — what a test is handed. Deliberately does not include
  anything that could start, stop or inspect the run.
- `I_TestRegistration`, `I_TestConfiguration`, `I_TestDefaults` — the composition
  root's surface. Cyclic code must not reach these.
- `I_TestExecution` — the driving surface. Can run a suite, cannot change what is
  in it.
- `I_TestRunStatus` — the read model. This is what a sequencer, an HMI or an
  external poller sees.
- `I_TestReporter` — a result sink. `I_TestDetailSink` is the optional second
  face for per-check detail, found with `__QUERYINTERFACE`.

If you add a capability, ask which of these it belongs on before adding a method
to an existing one. Most maintenance damage to a framework like this is done by
widening an interface that was narrow on purpose.

---

## 3. Decisions that will look wrong

These are the ones most likely to be "corrected" by someone who does not know why.

### `I_Test.Run`, not `CyclicLogic`

The house rule bans an unconditional per-scan method. This is not one: it takes a
held `Enable` and reports completion, which makes it a **driven step**. The
framework calls it; it is not called by the task.

### A test that records nothing is `Errored`

Not `Passed`. A body that checked nothing usually means a branch never ran, and
reporting that green is a silent false pass — the single defect most worth
designing against.

### `Errored` is not `Failed`

`Failed` — a check did not hold; the code under test was shown to be wrong.
`Errored` — the test could not run; nothing was shown about the code at all.

`Errored` outranks `Failed` in an aggregate, because a framework or environment
problem casts doubt on every other result in the run.

### Floats compare exactly

No hidden epsilon. Tolerance has its own methods so that choosing it is visible
at the call site. A near miss appends a hint naming them.

### `ElapsedCycles` excludes the scan in progress

The journal is ticked *after* the body runs, so on the body's Nth call it reads
N−1. That keeps it consistent with `IsFirstCycle`, which is TRUE on the same call
this reads 0. Observing a duration therefore costs one scan more than the
duration — cheap, and honest; the alternative is a count that includes work not
yet done.

### `Summary` is composed by `Finish`, not per check

A test with forty checks composes one line, at the end. Reading `Summary` before
finishing the journal returns an empty string. This has caught test code more
than once.

### Only the first failure reaches the summary

Later failures are usually downstream of the first, and the summary has room for
one. The rest survive on the detail sink, which is why detail is a *stream*
rather than a value: `CONCAT` cannot exceed 255 characters, so one comparison of
two long values already overflows a per-test buffer.

### A terminal outcome from `TestRunner.Advance` is reported once

The scan the test finishes returns `Finished`; every call after that returns
`Idle`. Returning the stored value again republishes the previous run's last test
on the first `Advance` of a re-run — invisible in a single run, because the suite
stops calling once complete.

### The run raises `RunFinished`, never the suite

A suite is not a run. Having a suite raise a run-level event means it must know
whether anything sits above it — which is a flag, and a flag the caller can
forget. Here it is structural: a single suite is composed with one `TestRun`, a
sequencer *is* one, and exactly one object can announce the end.

`ReRun` lives at the run for the same reason: the document boundary is there.

### The watchdog measures progress, not duration

Scans since the suite last brought a test to a **verdict**, not scans since it
started. A rig suite waiting on hardware can sit for minutes and be healthy — it
still finishes tests one at a time. Counting from the start would abandon the slow
along with the stuck, which is worse than having no watchdog.

It is a backstop, hence the generous 60000 default: per-test deadlines already
bound everything that did not disable them.

### The arbiter's "none" is a null reference

Not a sentinel index. Letting index 0 double as "no active test" is the trap: a
long-running test in the first slot would let the next one start alongside it.

### "All skipped" is counted in tests, at every level

`TestRunTally.Finish` settles `Skipped` when at least one test was skipped and none
passed, once errors and failures are ruled out. It once compared `SkippedCount`
with the number of results folded in — tests in a suite, but *suites* in a run —
so a run of one pass and two skips across two suites settled `Skipped`, and a lone
suite of two skips settled `Passed`. Within a suite the two rules agree, which is
why it went unnoticed.

### A skipped suite is announced when the run reaches it

`Skip` settles the counts at once but tells no sink. It may only be called before
the suite is first enabled — before `RunStarted`, and usually before any sink is
attached — so an announcement made there reaches nothing, or arrives outside the
run. `Run` makes it on the first enable: each test (`TestFinished`, with no
`TestStarted`), then `SuiteFinished`. Cases first, because a document builder
closes the suite element on `SuiteFinished`.

The label and location come from the registry **slot**, not the journal. A journal
learns them in `Begin`, when its test becomes active, and a skipped test never
does. Reading the journal sent every skipped case out nameless.

### A skipped suite stays skipped through a re-run

A re-run re-executes what was composed, and a skip is composition — usually there
because running those tests *here* would be wrong. `TestSuite.ReRun` on a skipped
suite resets nothing and runs nothing; it re-arms the announcement, so the fresh
document still lists the skipped tests.

### `Skip` makes the checks `StartRun` makes

A skip bypasses `StartRun`, so it repeats that method's two checks, in its order:
the label, then re-applying the errors raised before the pass. The label comes
first because, with no run yet, it adds to the errors being re-applied. A wiring
error outranks a skip, so such a suite is `Errored`.

### A skipped test raises no event

The event class has per-test events for Passed, Failed and Errored only. Narrating
a skip as `TestPassed` would report work that never ran; the suite's `SuiteSkipped`
carries the count. A per-test skipped event needs an entry in the class first.

---

## 4. The verifier

`TestHarnessCoreVerifier` — 111 tests, plus the floor check. Its run banner names
the library version it ran against.

### How the tests are organised

| Folder | Covers |
|---|---|
| `Tests/Registry` `Tests/Arbiter` `Tests/Runner` `Tests/Suite` `Tests/Journal` `Tests/Verdict` | the objects |
| `Tests/Assertions` | the twelve assertions — the public surface |
| `Tests/PureFunctions` | comparison, rendering, formatting, the stopwatch |
| `Tests/Fanout` `Tests/LogReporter` `Tests/EventReporter` `Tests/JUnitReporter` | reporting |
| `Tests/Watchdog` | the suite watchdog |
| `Tests/Characterization` | the acceptance contract |

### Two patterns you will need

**Testing failure without failing.** Give the test its own private `TestJournal`
and `TestVerdict`, drive a deliberate failure into those, and assert the journal
noticed. The test passes; what it proves is that failing works.

```iecst
Handle.Target(Subject);
Subject.Begin('deliberate', '');
Handle.AssertEqual(One, Two, 'one is not two');
Test.AssertTrue(Subject.Kind = E_FailureKind.ValueMismatch, 'recorded');
Handle.Untarget();
```

**Testing something multi-scan.** The outer test holds a private runner, suite or
run and steps it one scan per call, returning `FALSE` until it finishes — the same
mechanism being verified, one level down. **Always carry a watchdog**, so a defect
that never completes fails its own test rather than hanging everything and looking
like a broken verifier.

### The doubles

`Doubles/` holds test doubles, not tests: passing, failing and erroring bodies;
hanging setup, body and teardown; a slow setup; a spy reporter that counts
everything; a recording reporter that captures results so a test can ask questions
of the *artifact*; a stalling suite; witnesses for ordering and concurrency.

Add to these rather than writing one-off fakes inside tests.

### Coverage is not the same as passing

Two gaps were found late by walking every public member against everything the
tests named — eight of the twelve assertions had never been *called* by a test,
and `TestStopwatch` had no coverage at all. Tests accumulate around whatever is
being built, so a surface finished early stops attracting them.

**Repeat that check after any significant addition.** It asks a different question
from "do the tests pass".

### Lines in the log that are there on purpose

A green verifier log still holds `FAILED`, `ERRORED`, `FRAMEWORK` and `SUITE FAILED`
lines. They come from tests that drive a private sink with a deliberate failure.
Lines prefixed `verifier sample -`, and events ending `on a sample, not this run`,
exist to be read by eye: a log line's wording and severity cannot be read back, so
those tests put every run banner and run event into every log. Only the final
`RUN` banner is the verifier's result.

---

## 5. TwinCAT facts that cost time

All verified on hardware or from Beckhoff documentation. None was settled by
reasoning about it.

### Generics

- A generic argument forwards to a member instance: `TestSuite<N>` can hold
  `TestRegistry<N>`. ✔
- `ARRAY[0..Capacity-1]` holds exactly `Capacity`. ✔
- **A generic instance cannot be passed as its concrete type** (C0201). It crosses
  boundaries only as an interface. This is why only three objects are generic.
- **`VAR_GENERIC CONSTANT` is not readable from outside the instance** (C0552).
- Two identically-written instantiations are different types.
- A generic FB can implement multiple interfaces, and `__QUERYINTERFACE` works
  between them. ✔

### Language

- **`VAR_IN_OUT CONSTANT` accepts a literal but not the result of a call.** A
  literal has an address; an expression result does not.
- **You cannot take a member off a call result** (C0185) — including a property
  returning a struct. Copy to a local first.
- An FB passed by `VAR_INPUT` is **copied**. Use `VAR_IN_OUT` or an interface.
- `REFERENCE TO X` is not substitutable for an interface at a call site. Where
  both are needed, hold both.
- A method returning `REFERENCE TO` that simply returns leaves the reference
  **undefined, not null** — and undefined reads as valid often enough to pass by
  luck. Assign `REF= 0` first.
- A loop variable used in a property getter must be declared in the getter.
- Reserved: `Step` (SFC); `Left`, `Right`, `Log`, `Find`, `Add`, `IndexOf` are
  standard functions. Case-insensitive, so `log` collides.
- `CONCAT` caps at 255 characters. This is a hard ceiling, and it is why detail
  streams.

### File format

- `<Folder>` elements must precede members in a `.TcPOU`. Out of order, XAE
  **silently ignores the file** — no error, the object simply is not there.
- A `VAR` block belongs in a property's `Get`/`Set`, never in its `Declaration`.

### Libraries

- **`FB_XmlDomParser`**: `GetDocumentNode()` is what an element attaches to.
  `GetRootNode()` returns the first *element*, which an empty document does not
  have — appending to it silently does nothing and produces a 23-byte file that
  reports success. An untouched parser already holds ~35 bytes, so "no document"
  is not "zero length".
- `SaveDocumentToFile` is asynchronous: it clears the `bExec` you hand it when the
  write ends, and its return is meaningful on that one call.
- **`Tc3_EventLogger`**: name the generated `TC_Events.<Class>.<Event>` members
  with `CreateEx`. A GUID literal with `Create` compiles whether or not the class
  is registered and raises events the logger cannot name — a silent, cosmetic-
  looking failure. The generated symbol does not resolve if the class is missing,
  so the build tells you instead.
- **The event class ships inside the library**, as an external type
  (`ExternalTypes.tmc`, exposing `Global.ST_TestHarnessEvents`). A consuming
  project needs no Type System entry of its own — adding the library reference is
  enough. Dropping a `.tmc` into a *PLC project* registers nothing; that is not
  what this is.
- **Changing the event class.** Edit it in the TMC editor, then check that
  `ExternalTypes.tmc` in the library project shows the change; re-pin with *Add
  Global Data Type* if it does not. TwinCAT recalculates the class GUID and lists
  the old one under Hidden Datatypes — expected, not damage. **Append events;
  never insert or remove one**, or positions in the generated structure move.
  `ExternalTypes.tmc` is source, not build output, so commit it: the repository's
  `.gitignore` excludes `*.tmc` and must exempt this file.
- `ADSLOGSTR`: a severity bit alone does not say where a message goes. OR in
  `ADSLOG_MSGTYPE_LOG`. Pass the text as `strArg`, never as the format string, or
  a `%` in a test label corrupts the output.

---

## 6. Working practice

**Work from the repository, not from a remembered state.** A `TestRunner` rebuilt
from a stale working copy once reverted a fix made seventeen deliveries earlier;
the verifier caught it on the next run, which is the only reason it cost minutes.

**Ask TwinCAT rather than reasoning about it.** Every wrong turn in this library's
construction came from answering a question that could have been checked — the
list in §5 is that list. Each was settled by one probe or one documentation page.

**A written test is not a result.** Say "compiles" or "written", never "green",
until something has actually run.

**Say what a test cannot establish.** `ADSLOGSTR` returns nothing, so a log test
proves which branch was taken and not what the line says. No PLC code reads the
JUnit file back, so a test proves the document is non-empty and not that it
parses. Both need one look by eye, once.

**Watch a new test fail before trusting it.** Run the verifier with the new tests
against the library as it was, see them fail for the stated reason, then apply
the fix. A test that has only ever passed has not been shown to detect anything.

**Bump the version for every library build you install.** The verifier resolves
the newest installed library, and its banner names that version — but a version
names an install, not its contents. Two builds saved under one number print the
same line.

---

## 7. Extending it

### A new reporter

Implement `I_TestReporter`. Add `I_TestDetailSink` only if you want per-check
detail; the fan-out asks once, at attachment, and remembers — so a sink that does
not want checks writes no empty method bodies.

If it does anything asynchronous, start it in the notification and advance it in
`CyclicLogic`, and make `IsIdle` FALSE until it lands. The fan-out ANDs every
sink, and CI waits on that.

Report a failure to deliver through a **separately attached** sink, never through
the fan-out you are part of — that calls back into you.

### A new assertion

Add it to `I_TestAssertions` and `TestVerdict`. Compose the diagnostic through
`TestValueCompare` and `Emit` so `Expected` and `Actual` reach the detail sink in
their own fields, in full, rather than only inside a summary string.

Then **write the test that would catch it being backwards.** Both directions, and
the boundary.

### A new failure kind

Add to `E_FailureKind` and to `TestJUnitReporter.KindName`. The two are not
checked against each other by anything.

---

## 8. Known gaps

- **The event class is checked against the reporter in one direction only.** The
  build proves every event `TestEventReporter` names exists — the generated symbol
  would not resolve otherwise. Nothing proves the reverse: `ResultNotWritten` (5)
  is declared and raised nowhere. Adding an event means editing both.
- **`<check>` elements are non-standard JUnit.** Harmless, ignored by CI, and the
  only way to carry per-check detail past the 255-character ceiling. If a CI tool
  ever objects, that is the trade to revisit.
- **No parameterised tests.** Register the same function block twice with
  different construction, or loop inside one test and use `SetStep` to say which
  case failed.
- **An error raised before any sink is attached is counted but never said.**
  Registration usually comes before `AddReporter`, so a refused registration makes
  its suite `Errored` while the message saying why reaches no sink. Holding such
  messages until the run starts would close it.
- **`ReRunSingle` does not check for a skip.** It runs a named test even in a
  skipped suite, and leaves that test's journal holding the single-run verdict,
  which a later re-run announces while the counts still say skipped.
- **`AssertEqual` on an enum reports bytes** — `differs at byte 0: expected
  16#05000000, actual 16#02000000` — and the author's message is the first thing
  cut from the 80-character summary. A helper that renders enum values by name
  would fix it, and touch many tests.
