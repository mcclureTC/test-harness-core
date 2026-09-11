# Changelog

## 0.0.11 — 2026-09-11

Everything since 0.0.4. Versions 0.0.5 to 0.0.10 were intermediate test builds and
were not released.

### Results that can change when you upgrade

- **The outcome of a run with skipped tests.** "All skipped" is now judged on test
  counts at every level. It used to depend on whether the number of skipped tests
  happened to equal the number of suites. A run with a pass beside skips is now
  always `Passed`; a run in which every test was skipped is now always `Skipped`.
- **A skipped suite with a wiring error is `Errored`.** A registration refused before
  `Skip`, or a skipped suite with no label, is now a framework error, and turns the
  run `Errored`. It used to read `Skipped`.
- **Run banners and run events.** A skipped run prints `RUN SKIPPED` and raises
  `RunSkipped` (Info); an errored run prints `RUN ERRORED` and raises `RunErrored`
  (Error). Both used to be `RUN FAILED` and `RunFailed`, at error severity.

### Fixed

- A skipped suite reached no sink when `Skip` was called before `AddReporter`, and
  was announced before `RunStarted` otherwise. It is now reported when the run
  reaches it, in its place, whatever the wiring order.
- Skipped tests now reach the sinks by name, so the JUnit document lists each one as
  a `<testcase>` with `<skipped/>`. They used to appear only as a count.
- A run-level `ReRun` executed the tests of skipped suites. They now stay skipped,
  and are reported again in the fresh document.
- With narration on, the log reporter names a skipped test `skipped` rather than
  `passed`, and the event reporter raises nothing for it rather than `TestPassed`.

### Added

- Event class `TestHarnessEvents`: `RunSkipped` (12) and `RunErrored` (13), appended
  after the existing eleven. The library's `ExternalTypes.tmc` changed with it.

### Repository

- `.gitignore` now tracks `ExternalTypes.tmc`. It is source; without it a fresh clone
  cannot build the library.

### Verifier

- 111 tests (was 97), suite capacity 130 (was 110). Its run banner now names the
  library version it ran against.
