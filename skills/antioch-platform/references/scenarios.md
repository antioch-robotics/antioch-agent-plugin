# Scenario execution and history

A scenario definition is a Python function decorated with `@antioch.scenario`.
Its parameters describe inputs; cases provide named input sets. Each invocation
has a distinct run ID and record containing author, timestamps, inputs, checks,
results, and artifact descriptors. Phase describes execution progress; outcome
describes its verdict. Use `scenario-design` for the Python authoring contract.

Managed runs pin service images and retain process output. Existing scripts
and notebooks can also record through the SDK without CLI dispatch; see
[recording existing code](../../scenario-design/references/recording.md).
Caller-owned records have no revision or managed rerun.

## Collect and run

```bash
antioch scenario collect --json
antioch scenario run --scenario falling_cube --set drop_height=4.5
```

Collection imports local source without allocating compute or executing the
scenario body. Inspect names, typed parameters, cases, and source paths before
submission; collection does not prove that native API calls or task logic work.

Default dispatch uses the project's interactive session, creating one only
if absent. See [execution modes](sessions.md#choose-the-execution-mode) for
background dispatch and follow options. `--case` selects authored inputs;
`--set` supplies typed overrides, and they cannot be combined. Use `--no-stream`
beside an existing GUI producer. Follow shows progress and verdicts;
`--verbose` adds captured process output. Check leaf help for other options.

## Find and analyze

```bash
antioch scenario list --json
antioch scenario suggest tag --json
antioch scenario show SCENARIO_RUN_ID --json
antioch scenario logs SCENARIO_RUN_ID --json
antioch scenario download SCENARIO_RUN_ID --json
```

List filters include scenario, suite, tags, parameters, results, phase/outcome,
author, and time. `--user` or `--mine` selects authors; project scope
defaults to the current project, with `--all-projects` for wider history.
Use leaf help for predicate syntax and paging; pass returned cursors unchanged.
Suggest discovers values such as tags, names, project, and author.

Read checks and errors, not just terminal outcome. Compare params, case IDs,
revision/images, timings, and result values; download relevant evidence.
Finite JSON goes to stdout, progress/errors to stderr. Followed JSON is
line-delimited; structured errors include `retryable`.

## Cancel, rerun, delete

```bash
antioch scenario cancel SCENARIO_RUN_ID
antioch scenario rerun SCENARIO_RUN_ID
```

Cancel signals active work and preserves completed evidence. A rerun creates
new records using saved images and inputs, not local edits. It does not promise
identical physics or timing. Caller/source-free records lack managed reruns.

`antioch scenario delete --run SCENARIO_RUN_ID` removes history; use it only
when deletion is requested.
