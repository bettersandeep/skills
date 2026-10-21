# Review principles

Apply every principle to every unit the MR adds or changes: a function, class, file, config key, flag, dependency, migration, or test.
Each principle has a probe.
The probe is the question the reviewer must answer with evidence.

## 1. Necessity

Probe: what breaks if we delete this unit?
If no caller, user, or stated goal of the MR needs it, it goes.
Typical cases: code "for later", config for a value that never changes, an interface with one implementation, a factory for one product, an option nobody sets.

## 2. Reuse before writing

Probe: does the repo, the stdlib, the platform, or an installed dependency already do this?
Cite the existing code path or API.
A re-implementation of a helper that lives a few files away is a finding.

## 3. Simplest correct form

Probe: write the shorter replacement, and show that it handles the same cases.
Boring over clever.
Fewer files, fewer layers, fewer moving parts.
Use a known design pattern when the problem fits it, but no pattern for one case.

## 4. Readability

Probe: can a new reader tell what this does, and why, from names and structure alone, at 3am during an incident?
Comments explain why, not what.

## 5. Root cause

Probe: for a bug fix, does the change fix the shared function that all callers use, or only the one path the ticket names?
Grep every caller of the changed function.

## 6. Edge cases and failure

Probe: what happens on empty, null, huge, duplicate, concurrent, retried, partial, and timed-out input?
Who notices a failure?
A failure must surface in a log, metric, or alert.
A silent failure is a finding.

## 7. Second and third order effects

Probe: what else changes because of this?
Callers, data at rest, migrations, performance at prod scale, cost, consumers in other teams, rollback.

## 8. Verifiability

Probe: is there a check that fails if this logic breaks, and proof that the change does what the MR says?
Lint errors and flaky tests are findings, also when the MR did not cause them.

## 9. Quality over build cost

Probe: does the MR pick a weaker design because it is faster to build?
"Quick" is not a reason.
Prefer quality, simplicity, robustness, scalability, and long term maintenance.

## 10. Safety stays

Probe: does a simplification remove input validation at a trust boundary, error handling that prevents data loss, or a security control?
These stay, also when they cost lines.


## 11. No data loss, data corruption

Probe: are there cases where there is a possibility of data loss?
Ensure that there is no data loss.
