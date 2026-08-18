# TestHarnessCore

A unit and integration test framework for TwinCAT 3, written in Structured Text.

Tests run **on the PLC**, in the task, against the code as it is actually
scheduled. Nothing is simulated off-target and nothing needs a PC to be
connected once the project is deployed.

```
Suite: 96 passed, 0 failed, 0 errored, 0 skipped, of 96
```

---

## Why this exists

A test that cannot run on the target is not testing the thing you ship. PLC code
is shaped by the scan: state machines advance one step per cycle, motion takes
hundreds of cycles to finish, and a race that only appears at a 1 ms task period
will never appear anywhere else.

TestHarnessCore is built around that rather than despite it.

- **A test may take as many scans as it needs.** It returns `FALSE` until it is
  done. There is no "wait for" that blocks the task.
- **Every test has a deadline**, in scans or wall-clock time or both, so a test
  that hangs ends the test rather than the machine.
- **Tests run one at a time.** They share the scan and usually share fixtures, so
  predictable beats fast.
- **Results leave the PLC in a form CI understands** — a JUnit XML document, plus
  TwinCAT event-log entries and system-log lines.

## What it is not

It does not mock, stub or intercept. There is no reflection and no code
generation. A test is a function block you write, and a fake is a function block
you write that implements the same interface as the real thing — which is
dependency inversion, and works because you designed for it, not because the
framework did anything clever.

It also does not run on a build server without a PLC. The tests execute on real
or simulated TwinCAT runtime; the artifact they produce is what your build server
reads.

---

## What a test looks like

```iecst
FUNCTION_BLOCK TEST_ValveOpensWithinTwoSeconds IMPLEMENTS I_Test
VAR
    _Valve : FB_Valve;
END_VAR

// ---- Run ----------------------------------------------------------------
METHOD Run : BOOL
VAR_INPUT
    Enable : BOOL;
    Test   : I_TestAssertions;
END_VAR

_Valve.Open();

IF NOT _Valve.IsOpen THEN
    Run := FALSE;          // not finished - called again next scan
    RETURN;
END_IF

Test.AssertTrue(_Valve.IsOpen, 'the valve reported open');
Run := TRUE;               // finished
```

and the composition root that runs it:

```iecst
PROGRAM PRG_Tests
VAR
    Runner    : TestRun<1>;
    Suite     : TestSuite<20>;
    JUnitSink : TestJUnitReporter;
    Valve     : TEST_ValveOpensWithinTwoSeconds;
    Budgets   : ST_TestBudgets;
    Wired     : BOOL;
END_VAR

IF NOT Wired THEN
    Wired := TRUE;

    Suite.SetLabel('Valves');
    Suite.Register(Valve, 'the valve opens within two seconds', Budgets);

    JUnitSink.SetPath('C:\ProgramData\Beckhoff\TwinCAT\3.1\Boot\Results.xml');

    Runner.SetLabel('nightly');
    Runner.AddReporter(JUnitSink);
    Runner.AddSuite(Suite);
END_IF

Runner.Run(Enable := TRUE);
```

That is the whole of it. `Runner.IsComplete` tells you when the run has finished
and `JUnitSink.IsIdle` tells you when the file has landed.

---

## Features

| | |
|---|---|
| **Multi-scan tests** | A test returns `FALSE` until it is finished. Setup and teardown may span scans too. |
| **Three-phase lifecycle** | Optional `Setup` and `Teardown`, each with its own deadline, each guaranteed to run. |
| **Deadlines** | Per test, in scans or wall-clock time, with suite-wide defaults. A hung test fails; it does not hang the run. |
| **Twelve assertions** | Equality across any type, ordering, absolute and relative tolerance, ranges, raw buffers, scan budgets. |
| **Typed failures** | `ValueMismatch`, `TypeMismatch`, `SizeMismatch`, `Timeout`, `Environment` — CI can group by them. |
| **Failed vs errored** | A failure means the code is wrong. An error means the test could not run. Counted apart. |
| **Reporting** | JUnit XML for CI, TwinCAT EventLogger events for HMI and operators, system-log lines for a glance. Attach any combination. |
| **Suite watchdog** | A suite that stops progressing is named, abandoned and reported rather than holding the run open. |
| **Re-run** | Repeat a whole run, or re-run one test while debugging without polluting the result document. |

## Requirements

TwinCAT 3.1 build 4026. References `Tc2_Standard`, `Tc2_System`, `Tc2_Utilities`,
`Tc3_Module`, `Tc3_EventLogger` and `Tc3_JsonXml`.

No other project setup. The event class the EventLogger sink raises against ships
inside the library as an external type, so adding the reference is enough.

## Getting started

Read **`Implementation_Guide.md`**. It covers writing tests, the assertion
surface, deadlines, setup and teardown, fakes, composition and CI.

**`Maintenance_Guide.md`** is for changing the library itself.

## Verification

`TestHarnessCoreVerifier` is a separate project that tests this one, using this
one. It runs 96 tests plus an independent floor check that drives the registry,
arbiter and runner by hand — so that a framework defect cannot disguise itself as
a passing run.
