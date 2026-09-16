# Suites

A suite is a named selection of scenarios and cases in `antioch.yaml`:

```yaml
suites:
  acceptance:
    description: Warehouse acceptance checks
    select:
      - tags: ["warehouse", "smoke"]
        exclude_tags: ["slow"]
      - scenarios: ["dock_alignment"]
        cases: ["narrow", "wide"]
```

Fields in one selector all apply; separate selectors form an ordered union.
Selectors match paths, scenario names, cases, required tags, and excluded tags.

## Collect, run, and inspect

```bash
antioch suite collect --json
antioch suite run acceptance
antioch suite show SUITE_RUN_ID --json
antioch suite list --suite acceptance --json
antioch suite summary --json
```

Collection expands local definitions without compute. Fix unexpected selections
before submitting. A suite records selected inputs and a frozen revision,
with one child scenario record per member.

Interactive suites run serially in the project's session. Detached execution
uses headless background capacity and can fan out within quotas. Follow/detach
semantics match [scenarios](scenarios.md).

Interactive suites stop after a child fails, errors, or times out and cancel unstarted
members. Detached suites continue the remaining cases, so use `--detach --follow`
for a full comparison that must retain both passing and failing cases. This
policy also applies when one scenario command selects several runs; it is not
a separate CLI flag.

Read each failing child's checks, logs, and artifacts using its scenario run
ID. List and summary filter by suite, phase/outcome, author, and time. Summary
is a bounded view over recent runs; show reads one invocation.

## Cancel and repeat

```bash
antioch suite cancel SUITE_RUN_ID
antioch suite rerun SUITE_RUN_ID
```

Cancel stops active work and prevents unstarted members from starting;
completed evidence remains. Reruns use saved images and inputs under new IDs.
Use `antioch suite delete --run SUITE_RUN_ID` only for requested deletion.
