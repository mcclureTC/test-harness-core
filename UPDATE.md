# Documentation for 0.0.11

Five files, all at the repository root. Copy them over your working copy.

| File | Change |
|---|---|
| `README.md` | test counts (96 -> 111); a **Skip** feature row; re-runs keep skips; link to the changelog |
| `Implementation_Guide.md` | new *Skipping a suite* (§7); library version in the target info; the four run banners and events (§8); the outcome rule and a skipped case in the document (§9); `SkippedCount` and `PassedCount` in CI (§10) |
| `Maintenance_Guide.md` | five new *Decisions that will look wrong* (§3); deliberate log lines (§4); changing the event class (§5); two working practices (§6); known gaps corrected and updated (§8) |
| `CHANGELOG.md` | **new**: everything since 0.0.4, starting with results that can change on upgrade |
| `.gitignore` | exempts `ExternalTypes.tmc` from `*.tmc`, so it can be committed |

## Publish it with the code, in one commit

The docs describe 0.0.11. Pushed without the code, they describe a library GitHub
does not have. One commit, from your working copy, where the project is 0.0.11:

```
git add README.md Implementation_Guide.md Maintenance_Guide.md CHANGELOG.md .gitignore
git add TestHarnessCore/TestHarnessCore/TestHarnessCore/ExternalTypes.tmc
git add TestHarnessCore.library
git add TestHarnessCore
git status          # check before committing - see below
git commit -m "V0.0.11"
git push
```

Before committing, check `git status` shows:

- `ExternalTypes.tmc` as **new**. It must carry 13 events, `RunSkipped` and
  `RunErrored` last. It was never in the repository, so without it a fresh clone
  cannot build the library.
- `TestHarnessCore.library` as **modified**. It is tracked even though `.gitignore`
  lists `*.library`, so save the 0.0.11 build over it first.
- The library project's `ProjectVersion` as `0.0.11`.
- **No** `TestHarnessCore.tmc` or `TestHarnessCoreVerifier.tmc`. Those are generated
  on every build and stay ignored.

The commit name follows the repository's own pattern: earlier releases are commits
named `V0.0.3` and `V0.0.4`, with no tags.
