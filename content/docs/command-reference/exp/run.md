# exp run

Run an [experiment](/doc/command-reference/exp): reproduce a variation of a
committed [pipeline](/doc/command-reference/dag) in a hidden project branch.

> Similar to `dvc repro` but for
> [experimentation](/doc/user-guide/experiment-management).

## Synopsis

```usage
usage: dvc exp run [-h] [-q | -v] [-f]
                   [<repro_options> ...]
                   [--params [<filename>:]<params_list>]
                   [-n <name>] [--queue] [--run-all] [-j <number>]
                   [targets [targets ...]]

positional arguments:
  targets               Stages to reproduce. 'dvc.yaml' by default
```

## Description

Provides a way to execute and track `dvc experiments` in your
<abbr>project</abbr> without polluting it with unnecessary commits, branches,
directories, etc.

> `dvc exp run` is equivalent to `dvc repro` for <abbr>experiments</abbr>. It
> has the same behavior when it comes to `targets` and stage execution (restores
> the dependency graph, etc.). See the command [options](#options) for more the
> differences.

Before using this command, you'll probably want to make modifications such as
data and code updates, or <abbr>hyperparameter</abbr> tuning. You can use the
`--set-param` option to change `dvc param` values on-the fly.

Each `dvc exp run` creates a variation based on the latest project version
committed to Git, and tracks this experiment automatically with an automatic
name like `exp-bfe64` (which can be customized with the `--name` option).

<details>

### How does DVC track experiments?

Internally, `dvc exp` uses actual commits under custom
[Git references](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
(found in `.git/refs/exps`). Each commit has the Git `HEAD` as parent and has
it's own SHA-256 hash. These are not pushed to the Git remote by default (see
`dvc exp push`).

> References have a unique signature similar to the
> [entries in the run-cache](/doc/user-guide/project-structure/internal-files#run-cache).

</details>

The results of the last experiment can be seen in the <abbr>workspace</abbr>. To
display and compare your experiments, use `dvc exp show` or `dvc exp diff`. Use
`dvc exp apply` to roll back the workspace to a previous experiment

Successful experiments can be made
[persistent](/doc/user-guide/experiment-management#persistent-experiments) by
committing them to the Git repo. Unnecessary ones can be removed with
`dvc exp gc`, or abandoned (and their data with `dvc gc`).

## Checkpoints

To track successive steps in a longer <abbr>experiment</abbr>, you can register
checkpoints with DVC during your code or script runtime (similar to a logger).

To do so, first mark stage `outs` with `checkpoint: true` in `dvc.yaml`. Having
at least a checkpoint <abbr>output</abbr> is needed so that the experiment can
later restart based on that output's last <abbr>cached</abbr> state.

⚠️ Using the `checkpoint` field in `dvc.yaml` is only compatibly with
`dvc exp run`, `dvc repro` will abort if any stage contains it.

Then, in the corresponding code, either call the `dvc.api.make_checkpoint()`
function (Python), or write a signal file (any programming language) following
the same steps as `make_checkpoint()` — please refer to that reference for
details.

Once this is setup, you can use `dvc exp run` to begin the experiment. When the
process finishes or gets interrupted (e.g. with Ctrl + `C`), DVC will
[apply](/doc/command-reference/exp/apply) the last checkpoint to the
<abbr>workspace</abbr> (overwriting any further changes done by the stage).

`dvc exp run` again will continue from this point (useful for interrupted runs).
Use `--reset` to roll-back the workspace to `HEAD` and restart the whole
experiment. Alternatively, you can use `--rev` to continue from a specific
(previous) checkpoint.

Note that `dvc exp show` displays checkpoints with a special branching format.

<details>

### How are checkpoints captured by DVC?

When DVC runs a checkpoint-enabled experiment, a custom Git branch (in
`.git/refs/exps`) is started off the repo `HEAD`. A new commit is appended each
time a checkpoint is registered by the code. These are not pushed to the Git
remote by default (see `dvc exp push`).

</details>

## Queueing and parallel execution

The `--queue` option lets you create an experiment as usual, except that nothing
is actually run. Instead, the experiment is put in a wait-list for later
execution. `dvc exp show` will mark queued experiments with an asterisk `*`.

Use `dvc exp run --run-all` to process this queue. Note that if the queued
experiments use checkpoints, `--run-all` implies `--reset` (restarts them).

Adding `-j` (`--jobs`), experiment queues can be run in parallel for better
performance. This creates a temporary workspace copy for each subprocess (in
`.dvc/tmp/exps`). See also `--temp`.

⚠️ Parallel runs are experimental and may be unstable at this time. ⚠️ Make sure
you're using a number of jobs that your environment can handle (no more than the
CPU cores).

> Note that each job runs the entire pipeline (or `targets`) serially — DVC
> makes no attempt to distribute stage commands among jobs.

## Options

> In addition to the following, `dvc exp run` accepts all the options in
> `dvc repro`, with the exception that `--no-commit` has no effect here.

- `-S [<filename>:]<params_list>`, `--set-param [<filename>:]<params_list>` -
  set the specified `dvc params` for this experiment. `filename` can be any
  valid params file (`params.yaml` bu default). `params_list` accepts a
  comma-separated list of key-value pairs in the form `key1=val1,key2=val2...`.
  This will override the param values coming from the params file.

- `-n <name>`, `--name <name>` - specify a name for this experiment. A default
  name will generated by default, such as `exp-f80g4` (based on the experiment's
  hash).

- `--queue` - place this experiment at the end of a line for future execution,
  but do not actually run it yet. Use `dvc exp run --run-all` to process the
  queue.

- `--run-all` - run all queued experiments (see `--queue`). Use `-j` to execute
  them [in parallel](#queueing-and-parallel-execution). For checkpoint
  experiments, this implies `--reset`.

- `-j <number>`, `--jobs <number>` - run this `number` of queued experiments in
  parallel. Only applicable when used in conjunction with `--run-all`.

- `--temp` - run this experiment in a separate temporary directory (in
  `.dvc/tmp/exps`) instead of your workspace.

- `-r <commit>`, `--rev <commit>` - continue an experiment from a specific
  checkpoint name or hash (`commit`). This is needed for example to resume
  experiments from `--queue` or `--temp` runs.

- `--reset` - restart a checkpoint experiment from scratch (resets the workspace
  and clears existing checkpoints before the run). Implies `--force`, so that
  cached checkpoint results are regenerated.

- `-f`, `--force` - reproduce pipelines even if no changes were found (same as
  `dvc repro -f`).

- `-h`, `--help` - prints the usage/help message, and exit.

- `-q`, `--quiet` - do not write anything to standard output. Exit with 0 if all
  stages are up to date or if all stages are successfully executed, otherwise
  exit with 1. The command defined in the stage is free to write output
  regardless of this flag.

- `-v`, `--verbose` - displays detailed tracing information.
