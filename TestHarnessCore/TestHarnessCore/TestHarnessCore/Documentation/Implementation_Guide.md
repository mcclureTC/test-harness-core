# Implementation Guide

How to write tests with TestHarnessCore.

- [1. The shape of a test](#1-the-shape-of-a-test)
- [2. Assertions](#2-assertions)
- [3. Multi-scan tests](#3-multi-scan-tests)
- [4. Setup and teardown](#4-setup-and-teardown)
- [5. Deadlines](#5-deadlines)
- [6. Testing something with dependencies](#6-testing-something-with-dependencies)
- [7. Composition](#7-composition)
- [8. Reporting](#8-reporting)
- [9. Reading a result](#9-reading-a-result)
- [10. CI](#10-ci)
- [11. Things that will catch you out](#11-things-that-will-catch-you-out)

---

## 1. The shape of a test

A test is a function block implementing `I_Test`:

```iecst
METHOD Run : BOOL
VAR_INPUT
    Enable : BOOL;
    Test   : I_TestAssertions;
END_VAR
```

- Return `TRUE` when the test is finished, `FALSE` to be called again next scan.
- `Test` is the handle you record checks against. It is valid only inside the
  call — do not store it.
- `Enable` is `FALSE` exactly once, if the run is abandoned. Put your own state
  back and return.

The minimum useful test:

```iecst
FUNCTION_BLOCK TEST_TwoPlusTwo IMPLEMENTS I_Test
VAR
    _Expected : DINT := 4;
    _Actual   : DINT;
END_VAR

METHOD Run : BOOL
VAR_INPUT
    Enable : BOOL;
    Test   : I_TestAssertions;
END_VAR

_Actual := 2 + 2;
Test.AssertEqual(_Expected, _Actual, 'two and two');
Run := TRUE;
```

**A test that records no check is reported `Errored`, not `Passed`.** A body that
checked nothing usually means a branch never ran, and reporting that green is a
silent false pass.

### Naming

The label you register a test under is what CI groups and trends by, so treat it
as an identifier rather than a caption. Write what should be true, not what the
test does:

```
'the valve opens within two seconds'      good
'test valve'                              useless in a report
```

---

## 2. Assertions

All twelve take a `Message : T_MaxString` last. The message appears in the
result; write the *expectation*, not the failure.

### Equality and ordering

| Method | Notes |
|---|---|
| `AssertTrue(Condition, Message)` | |
| `AssertFalse(Condition, Message)` | |
| `AssertEqual(Expected, Actual, Message)` | Any type. See the comparison rules below. |
| `AssertNotEqual(Expected, Actual, Message)` | |
| `AssertGreater(Actual, Bound, Message)` | Strict. Equal is not greater. |
| `AssertLess(Actual, Bound, Message)` | Strict. |

### Comparison rules for `AssertEqual`

- **Numbers compare by value across widths.** `INT#5` equals `DINT#5`; the
  storage width is incidental to what you are checking.
- **Strings compare by content**, not by declared capacity. A `STRING(20)` and a
  `STRING(80)` holding `'homed'` are equal.
- **`WSTRING` is compared as `WSTRING`.**
- **Structs, arrays and enums compare by size, then bytes.** Here a size
  difference *is* a type difference — two arrays of different length are two
  different types — and the diagnostic names the byte offset of the first
  difference.
- **`REAL` and `LREAL` compare exactly.** There is no hidden epsilon. A near miss
  appends a hint pointing at the tolerance methods.

### Floating point

```iecst
Test.AssertNear(10.0, Measured, 0.05, 'position within 0.05 mm');
Test.AssertWithin(1000.0, Measured, 0.001, 'speed within 0.1 per cent');
Test.AssertInRange(Measured, 1.0, 10.0, 'pressure in band');
```

- `AssertNear` takes tolerance in the **units of the value** — millimetres,
  volts, counts. Both bounds inclusive, band symmetric.
- `AssertWithin` takes a **fraction**, so it means the same thing at any
  magnitude. Near zero it collapses to exact equality: a fraction of nothing is
  nothing. If zero is the expected value, `AssertNear` is the method you want.
- `AssertInRange` is inclusive on both bounds.

### Buffers

```iecst
Test.AssertMemEqual(ADR(Expected), ADR(Actual), SIZEOF(Expected), 'the frame');
Test.AssertBufferEqual(ADR(Expected), 8, ADR(Actual), ActualLen, 'the reply');
```

`AssertBufferEqual` checks the **lengths first**, so a size difference is
reported as a `SizeMismatch` rather than as a difference at whatever byte
happened to differ. A null pointer is reported as `Environment`: nothing was
compared, so nothing was shown about the code.

### Timing

```iecst
Test.AssertCompletedWithin(50, 'homing took too long');
```

Compares against **completed** scans — the one in progress is not counted, so
assert it on the scan *after* the work finishes, which is where the answer is
known anyway.

### Ending a test without a comparison

```iecst
Test.Fail('the default case was reached');          // the code is wrong
Test.ReportError('the axis would not power');       // the test could not run
```

`Fail` produces `Failed`. `ReportError` produces `Errored`. Keep them apart: a
failure sends the reader to the code under test, an error sends them to the rig.

### Naming a step

```iecst
Test.SetStep('Homing');
```

The step label leads the summary line, and the step current *when a check ran* is
the one recorded. For anything multi-scan this is usually the whole diagnostic:
`Homing: timed out after 5000 cycles` ends an investigation that
`timed out after 5000 cycles` starts.

---

## 3. Multi-scan tests

Return `FALSE` and you are called again next scan. Use your own state:

```iecst
CASE _Phase OF

0:  _Axis.MoveAbsolute(Position := 100.0);
    _Phase := 1;

1:  IF _Axis.InPosition THEN
        Test.AssertNear(100.0, _Axis.ActualPosition, 0.01, 'reached target');
        Run := TRUE;
    END_IF

END_CASE
```

`Test.IsFirstCycle` is `TRUE` on the body's first call, and
`Test.ElapsedCycles` counts scans the body has **completed** — zero on the first
call.

Note: `Step` is an SFC keyword and cannot be a variable name. `Phase` works.

---

## 4. Setup and teardown

Implement `I_TestSetup` and `I_TestTeardown` as needed. Both are optional and
independent; the runner discovers them at registration.

```iecst
FUNCTION_BLOCK TEST_AxisHomes IMPLEMENTS I_Test, I_TestSetup, I_TestTeardown
```

```iecst
METHOD Setup : BOOL       // same signature as Run
METHOD Teardown : BOOL
```

- **Setup runs first**, may take several scans, and returns `TRUE` when ready.
- **If setup records a failure, the body never runs.** The code under test was
  never exercised, so reporting a failed check would send the reader after the
  wrong thing.
- **Teardown always runs**, whatever the verdict, including after a failed setup.
  It can read `Test.Status`, which is already terminal, so cleanup can branch on
  the outcome.
- **Teardown cannot change the verdict.** A passing test whose cleanup overran
  still passed; the overrun is reported separately as a framework error.

---

## 5. Deadlines

Every phase has its own budget, in scans and in wall-clock time:

```iecst
VAR
    Budgets : ST_TestBudgets;   // .Setup .Body .Teardown, each ST_TestBudget
END_VAR

Budgets.Body.Cycles      := 500;
Budgets.Body.IsCyclesSet := TRUE;

Budgets.Body.Timeout      := T#5S;
Budgets.Body.IsTimeoutSet := TRUE;

Suite.Register(MyTest, 'the axis homes', Budgets);
```

- **Zero disables** that deadline. `IsCyclesSet`/`IsTimeoutSet` distinguish
  "deliberately zero" from "not specified", which is why they exist.
- **Units are tracked separately.** Naming a wall-clock budget does not disable
  the inherited scan budget, and the reverse.
- **Unset units inherit the suite default**, applied when the run starts — so a
  default set after some tests are registered still reaches them.

```iecst
Defaults.Body.Cycles      := 5000;
Defaults.Body.IsCyclesSet := TRUE;
Suite.SetDefaults(Defaults);
```

Prefer **scan budgets** for anything asserted on. They are reproducible; wall
clock is not, and is only reported.

---

## 6. Testing something with dependencies

There is no mocking. Substitute through an interface the code under test already
depends on:

```iecst
FUNCTION_BLOCK SimulatedValve IMPLEMENTS I_Valve
VAR
    _Timer     : TON;
    _IsOpen    : BOOL;
    _TravelTime : TIME := T#500MS;
END_VAR
```

Your test composes the unit under test with the simulated dependency, drives it,
and asserts. If the production code takes a concrete type rather than an
interface, that is the thing to change — the test is telling you something true
about the design.

A fake that records what it was told is often more useful than one that just
answers:

```iecst
FUNCTION_BLOCK SpyValve IMPLEMENTS I_Valve
VAR
    _OpenCalls : UDINT;
END_VAR
// ... plus a PROPERTY OpenCalls : UDINT for the test to assert on
```

---

## 7. Composition

Three objects, wired once at startup.

```iecst
PROGRAM PRG_Tests
VAR
    Runner  : TestRun<2>;        // <> is how many SUITES it can hold
    Suite   : TestSuite<40>;     // <> is how many TESTS it can hold
    Budgets : ST_TestBudgets;
    Wired   : BOOL;

    ValveOpens  : TEST_ValveOpens;
    ValveCloses : TEST_ValveCloses;
END_VAR

IF NOT Wired THEN
    Wired := TRUE;

    Suite.SetLabel('Valves');
    Suite.Register(ValveOpens,  'the valve opens within two seconds', Budgets);
    Suite.Register(ValveCloses, 'the valve closes on power loss',     Budgets);

    Runner.SetLabel('nightly');
    Runner.SetTargetInfo('CX5140 rig 2');
    Runner.AddSuite(Suite);
END_IF

Runner.Run(Enable := TRUE);
```

**Every project needs a `TestRun`, including one with a single suite.** The
result document is written when the run ends, so without a run there is no
artifact.

- **Registration closes** the first time the suite is enabled. A late
  registration is refused *and reported* — it never vanishes silently.
- **`Register` returns `BOOL`.** Assert `Suite.RegisteredCount` against a number
  you expect; a test that stops being registered otherwise just makes the totals
  smaller, which reads as an improvement.
- **Suites run one at a time**, in the order added. That is where coarse
  exclusion comes from: two suites touching the same axis cannot overlap.
- **`TestRun` implements `I_TestRunStatus`**, so runs nest. Counts are *tests* at
  every level.

### Re-running

```iecst
Runner.ReRun();                  // the whole run again, fresh document
Suite.ReRunSingle('the valve opens within two seconds');   // debugging aid
```

`ReRunSingle` really runs the test but records **nothing** — no count moves and
nothing reaches a reporter. A session at the keyboard does not pollute the CI
artifact.

---

## 8. Reporting

Attach sinks to the **run**, which pushes them into every suite. One attachment
point.

```iecst
Runner.AddReporter(LogSink);      // TestLogReporter    - system log
Runner.AddReporter(EventSink);    // TestEventReporter  - TwinCAT EventLogger
Runner.AddReporter(JUnitSink);    // TestJUnitReporter  - the CI artifact
```

**`TestLogReporter`** writes lines via `ADSLOGSTR`. No project setup. Quiet about
passes by default; `SetVerbosity(TRUE)` narrates them. Brackets each run with a
banner, which is only useful if you sort the Logged Events window by time.

**`TestEventReporter`** raises TwinCAT 3 EventLogger events — structured,
filterable, HMI-visible, and routable to an alarm view. No setup: the event class
ships with the library as an external type.

**`TestJUnitReporter`** writes the XML document CI reads.

```iecst
JUnitSink.SetPath('C:\ProgramData\Beckhoff\TwinCAT\3.1\Boot\Results.xml');
JUnitSink.AttachErrorSink(LogSink);      // where a failed WRITE is reported
```

The path is required and has no default — it differs by platform and build, and a
default that is wrong writes somewhere nobody looks.

Severity is not decoration in any of them: a **failure** is an error, an
**errored** test is a warning. A build should react differently to each.

---

## 9. Reading a result

On the run:

| | |
|---|---|
| `IsComplete` | every suite has finished |
| `Outcome` | `Passed` / `Failed` / `Errored` / `Skipped` |
| `TestCount` `PassedCount` `FailedCount` `ErrorCount` `SkippedCount` | |
| `TotalDuration` | seconds |
| `StalledSuites` | assert this is zero |

**`Errors` above zero means the run cannot be trusted** — the framework could not
do something. Investigate that before reading any other number.

The document is JUnit XML:

```xml
<testsuites target="CX5140 rig 2">
  <testsuite name="Valves" tests="2" failures="1" errors="0" skipped="0" time="1.310000">
    <testcase name="the valve opens within two seconds" time="0.190000"
              cycles="18" assertions="4" checks-passed="3" checks-failed="1">
      <check step="Home" expected="1" actual="2" diagnostic="expected 1, actual 2"
             message="deliberate"/>
      <failure type="ValueMismatch" message="Home: expected 1, actual 2">
        <![CDATA[Home: expected 1, actual 2]]>
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

`<check>` elements are an extension: standard JUnit has nowhere to put per-check
detail, and a summary line cannot exceed 255 characters. CI ignores them and
reads `<failure type= message=>` normally.

---

## 10. CI

**Wait for `IsComplete` AND `IsIdle`.** A run finishing and its file reaching disk
are different events; copying between them yields a truncated artifact that reads
as a broken pipeline rather than a finished run.

```iecst
CiCanCollect := Runner.IsComplete AND JUnitSink.IsIdle;
```

Assert in CI, not just in the report:

- `TestCount` equals the number you expect
- `StalledSuites` is zero
- `JUnitSink.WriteFailures` is zero

A missing file is ambiguous — a configuration problem and a run with no tests
look identical. An empty *announced* run writes a document with `tests="0"`,
which says which.

---

## 11. Things that will catch you out

**Reserved words.** `Step` is SFC. `Left`, `Right`, `Log`, `Find`, `Add` are
standard functions. TwinCAT is case-insensitive, so `log` collides too.

**`ANY` parameters need addressable variables.** `AssertEqual(4, X, '...')` will
not compile — a literal has no address. Declare `Expected : DINT := 4;` and pass
that. The `LREAL` assertions take plain `VAR_INPUT` and accept literals.

**A property is a method call.** You cannot take a member off one:
`Journal.Budgets.Setup` fails. Copy the struct to a local first.

**Don't store the assertion handle.** It is valid for the call.

**Capacity is capacity.** `TestSuite<20>` holds twenty tests.

**One test, one instance.** Two registrations of the same function block instance
share its state.
