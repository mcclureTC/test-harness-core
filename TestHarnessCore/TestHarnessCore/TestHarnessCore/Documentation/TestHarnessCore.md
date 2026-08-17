# TestHarnessCore

The core of a PLC unit-test framework: the contracts a test author implements, the record kept for
each test, the value comparison the assertions are built on, and the verdict handle a test body
reports through.

This project holds no suite, no sequencer and no reporter. It is the layer those are written
against — `I_TestSuite`, `I_TestReporter` and `I_TestRunStatus` are declared here and implemented
elsewhere.

## Environment

| | |
|---|---|
| TwinCAT version | 3.1 Build 4026 (record the exact build before release) |
| Target platform | Platform-independent — no I/O, no motion, no ADS |
| Visual Studio | VS2022 / TcXaeShell |

## Libraries

Versions are **not** pinned below because they were not read from an installed system. Record the
resolved version from the Library Manager before release — "newest" is not a version.

| Library | Version | Used for |
|---|---|---|
| Tc2_System | record it | `MEMCPY`, `T_MaxString`, `GETCURTASKINDEXEX` and `_TaskInfo` for the cycle-counting stopwatch |
| Tc2_Standard | record it | `CONCAT`, `LEFT`, `LEN`, `SEL`, `MIN`, `MAX`, `ABS` in diagnostic composition |
| Tc2_Utilities | record it | `LREAL_TO_FMTSTR`, `BYTE_TO_HEXSTR`, `WSTRING_TO_STRING2` for rendering values |
| Tc3_Module | record it | Always referenced by a TwinCAT 3 PLC project |

This project must **not** reference a `TestHarness` library. It defines those types itself, and a
reference would be circular.

## Licences required on the target

None beyond the base PLC runtime. The project uses no motion, no fieldbus and no TF-numbered
function.

## Structure

- `DUTs/` — the enumerations and structs that cross a boundary
- `Interfaces/` — the contracts; one file per interface
- `POUs/` — the implementations and the pure helper functions

### The interface closure

| Interface | Extends | Declares |
|---|---|---|
| `I_Cyclic` | `__SYSTEM.IQueryInterface` | `CyclicLogic` |
| `I_Labelled` | `__SYSTEM.IQueryInterface` | `Label` |
| `I_Test` | `__SYSTEM.IQueryInterface` | `Run` |
| `I_TestSetup` | `__SYSTEM.IQueryInterface` | `Setup` — optional |
| `I_TestTeardown` | `__SYSTEM.IQueryInterface` | `Teardown` — optional |
| `I_TestLocation` | `__SYSTEM.IQueryInterface` | `Path` — optional |
| `I_TestOutcome` | `__SYSTEM.IQueryInterface` | `RecordPass`, `RecordFailure`, `Fail`, `ReportError`, `SetStep`, `Status`, `AssertionCount` |
| `I_TestTiming` | `__SYSTEM.IQueryInterface` | `IsFirstCycle`, `ElapsedCycles`, `ElapsedTime` |
| `I_TestAssertions` | `I_TestOutcome`, `I_TestTiming` | the twelve `Assert*` methods |
| `I_TestExecution` | `__SYSTEM.IQueryInterface` | `Run`, `Skip` |
| `I_TestRegistration` | `__SYSTEM.IQueryInterface` | `Register`, `RegisteredCount`, `IsOpen` |
| `I_TestDefaults` | `__SYSTEM.IQueryInterface` | `SetDefaultTimeoutCycles`, `SetDefaultTimeout` |
| `I_TestRunStatus` | `I_Labelled` | `IsComplete`, `Outcome`, the counts, `TotalDuration` |
| `I_TestSuite` | `I_TestExecution`, `I_TestRunStatus` | nothing of its own |
| `I_TestReRun` | `__SYSTEM.IQueryInterface` | `ReRun`, `ReRunSingle` |
| `I_TestReporter` | `I_Cyclic` | `TestFinished`, `SuiteFinished`, `RunFinished`, `FrameworkError`, `IsIdle` |

### Function blocks

- `TestVerdict` implements `I_TestAssertions`. One instance per suite, retargeted to whichever
  journal is active. Its methods are grouped into a folder per interface in the closure —
  `I_TestAssertions`, `I_TestOutcome`, `I_TestTiming` — with `Target` at the top level as its own.
- `TestJournal` — the record of one test. Implements no interface; the suite owns it.
- `TestStopwatch` — elapsed seconds derived from the task cycle counter.

### Functions

All four are pure and testable on their own.

- `TestValueCompare` — compares two `ANY` values, returns `ST_CompareResult`
- `TestValueAsLReal` — widens a numeric `ANY`, returns `ST_NumericValue`
- `TestValueString` — renders any value as text
- `TestSummaryLine` — composes the 80-character summary
- `TestValueIsFloat`, `TestValueIsText` — type-class predicates

## Conventions

Naming and structure follow the project coding standard: PascalCase, `_PascalCase` private fields,
no Hungarian notation. The `p` prefix survives only on the raw-memory signatures
(`AssertMemEqual`, `AssertBufferEqual`), which deliberately mirror the `MEMCPY`-family calls they
wrap.

Function blocks declare no `VAR_INPUT` and no `VAR_OUTPUT`; state enters through methods and leaves
through properties. No function or method declares `VAR_OUTPUT` — where two values are needed, a
struct is returned.

## Building

This is a library project. Set Company, Title and Version in the project properties before
**Save as library**; those fields are not carried in the `.plcproj` and the generator does not
write them.

There is no task and no `MAIN`, which is correct for a library. Referencing projects supply both.
