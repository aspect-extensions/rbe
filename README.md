# rbe — Aspect CLI extension

An installable [Aspect CLI](https://aspect.build/docs/cli) module that adds an
`aspect rbe` task group for evaluating a Bazel workspace's remote-build-execution
readiness — no RBE backend required.

- **`aspect rbe check [targets...]`** — static hermeticity checks over target
  tags/attrs (from `bazel query`) and `.bazelrc` red flags. Fast; no build.
  Pass `--fail-on-blocker` to gate CI.
- **`aspect rbe analyze [targets...]`** — runs an instrumented local
  `bazel test` (execution log + BES), then reports dynamic hermeticity
  violations plus a worker-pool capacity plan (executors and EC2/GCE instance
  shapes) for running the tests under RBE.

> **Experimental.** The sizing factors (memory heuristics, remote overhead,
> utilization) are uncalibrated. Treat the capacity plan as a starting point,
> not a quote.

## Install

Add to your repo's `MODULE.aspect`:

```python
axl_archive_dep(
    name = "rbe",
    urls = ["https://github.com/aspect-extensions/rbe/archive/<tag>.tar.gz"],
    integrity = "sha512-...",
    strip_prefix = "rbe-<version>/module",
    auto_use_tasks = True,
)
```

`auto_use_tasks = True` registers `aspect rbe check` and `aspect rbe analyze`
as CLI commands.

### Requirements

`aspect rbe analyze` depends on runtime features (`current_dir` on
`ctx.bazel.query()` / `ctx.bazel.info()` and workspace-directory threading for
the instrumented test run) introduced in
[aspect-build/aspect-cli#1229](https://github.com/aspect-build/aspect-cli/pull/1229).
Requires an aspect-cli release that includes those changes (set the concrete
minimum version here once published).

## Usage

```sh
# Static checks only, fail CI on any blocker:
aspect rbe check --fail-on-blocker -- //...

# Analyze a workspace from anywhere; write plain-text + JSON reports:
aspect rbe analyze --workspace /path/to/repo \
    --report-out rbe.txt --json-out rbe.json -- //...
```

Key flags:

- `--workspace <path>` — analyze any repo from anywhere (default: current dir).
- `--report-out <file>` — also write the report as plain text (no ANSI color).
- `--json-out <file>` — (`analyze` only) also write a machine-readable report.
- `--fail-on-blocker` — (`check`) exit non-zero on any blocker-severity finding.

See `docs/rbe-advisor-design.md` for the methodology (severity model, scoring,
and capacity-sizing math).

## Layout

```
module/MODULE.aspect       module manifest (exports check / analyze)
module/rbe.axl             task layer: Bazel I/O in, typed records, rendered report out
module/lib/rbe_advisor.axl pure analysis logic (hermeticity checks + sizing)
.aspect/axl.axl            unit tests — run `aspect tests axl` from the repo root
```

## Development

```sh
aspect tests axl        # run the rbe_advisor unit suite
aspect rbe check -- //...   # run against this repo (via the local dep in MODULE.aspect)
```
