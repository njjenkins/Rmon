# Rmon

Run an R script and watch CPU and RAM usage in a status bar pinned to the bottom of your terminal, while the script's normal output scrolls above it. No tmux, no extra windows, no code changes to your script.

```
$ Rmon code/02_build/build_regression_panel.R
Attaching package: ‘dplyr’
The following objects are masked from ‘package:data.table’:
    between, first, last
Building Regression Panel...
Loading Master Panel: data/cleaned/analysis_panel_2026_07_03.parquet
Merging transaction history...
 CPU   554.1%   RAM   48.72 GB   elapsed 00:12:47
```

## Why

Long-running R jobs — big joins, panel construction, anything reading multi-GB parquet — leave you guessing whether the process is working or thrashing, and whether it's about to hit a memory ceiling. The usual answers are to open a second SSH session for `htop`, or to find out after the fact when the OOM killer gets there first.

Rmon puts the numbers in the same terminal as the output, live.

## Install

```bash
curl -o ~/.local/bin/Rmon https://raw.githubusercontent.com/USER/Rmon/main/Rmon
chmod +x ~/.local/bin/Rmon
```

Make sure `~/.local/bin` is on your `PATH`. No dependencies beyond bash, coreutils, and `util-linux`.

## Usage

Drop-in replacement for `Rscript`:

```bash
Rmon myscript.R
Rmon myscript.R --arg1 value        # arguments pass through
```

Exit codes propagate, so it composes normally in pipelines and Makefiles.

If stdout isn't a terminal — a batch job, or `Rmon script.R > log.txt` — Rmon detects it and `exec`s straight into plain `Rscript`. You can leave it in scripts that run both interactively and under a scheduler.

## What the bar shows

| Field | Meaning |
|---|---|
| `CPU` | Summed `%cpu` across the R process and all descendants. **Relative to one core**, so 400% means four cores' worth. |
| `RAM` | Summed PSS across the process tree — see below. |
| `elapsed` | Wall clock since launch. |

If the label reads `RAM~` rather than `RAM`, the kernel is too old for PSS and the figure is a summed-RSS upper bound.

## How it works

Two things make this less trivial than it looks, both learned the hard way.

**Rmon is the sole writer to the terminal.** The R process runs under a pty (via `script`) and its output is piped back to the wrapper, which prints the content *and* paints the status bar from a single loop. The obvious implementation — a background monitor painting the bar while R writes directly to the same TTY — corrupts output: the monitor saves the cursor, R writes, the monitor restores the cursor to a now-stale position, and R's next write lands in the middle of a line it already printed. A pty rather than a plain pipe also keeps R line-buffered instead of block-buffered, and keeps `isatty()` true so progress bars and colored output behave normally.

**Every line ends in an explicit CRLF.** `script` puts the outer terminal into raw mode, which disables `ONLCR`, the output post-processing that normally turns `\n` into CR+LF. Without an explicit `\r`, each line starts at the column where the previous one ended and the output staircases off the right edge of the screen.

**Memory is PSS, not RSS.** Summing `ps` RSS across forked workers counts the parent's copy-on-write pages once per worker. For a 50 GB parent forked six ways by `mclapply`, that reports ~300 GB. Rmon sums `Pss` from `/proc/<pid>/smaps_rollup`, which divides each shared page among the processes mapping it. A quick demonstration — one 400 MB allocation, five forks:

```
processes: 6
summed RSS = 2.38 GB   <- double-counts COW pages
summed PSS = 0.40 GB   <- true footprint
```

The terminal state is saved with `stty -g` at startup and restored on exit, and a `SIGWINCH` handler recomputes the scroll region on resize.

## Requirements

- **Linux.** Uses `/proc` and Linux `ps`; will not work on macOS or BSD.
- **bash 4+** — the script uses bash-only syntax and will misbehave under `dash` or `sh`.
- **`util-linux` ≥ 2.22** for `script -f -e`.
- **Kernel ≥ 4.14** for PSS via `smaps_rollup`. Older kernels (RHEL 7) fall back to summed RSS, labeled `RAM~`.
- A VT100-capable terminal, which in practice means any of them.

## Limitations

- The CPU figure is summed across processes and relative to a single core. Compare it against the cores you actually requested — if you asked for 4 and see 550%, something (`data.table::setDTthreads()`, an OpenMP BLAS) is oversubscribing, which slows you down on a shared scheduler.
- PSS is sampled once a second, so a short allocation spike between samples can be missed. For a hard peak figure, use `/usr/bin/time -v`.
- If your R code does its own cursor positioning near the bottom of the screen, it can collide with the status bar. A tmux split pane is immune to this; Rmon is not.

## Prior art

Rmon is narrow by design and there are good tools for adjacent problems:

- [**psrecord**](https://github.com/astrofrog/psrecord) — records CPU and memory to a log or plot, and can launch the process itself. Use `--include-children`. RSS-based, so it double-counts forked workers.
- [**bottombar**](https://pypi.org/project/bottombar/) — a Python library implementing the same scroll-region status line, more generally and more portably than this does.
- [**syrup**](https://simonpcouch.github.io/syrup/) — R-native, snapshots `ps` to profile parallel R code, returns a tibble. Post-hoc rather than live; also RSS-based.
- **`/usr/bin/time -v`** — peak RSS after the fact, nothing to install. If your question is only "will this OOM," this is usually enough.
- **`seff` / `sstat`** — under SLURM, these report what the scheduler is actually measuring and will kill you for. Authoritative for sizing `--mem`.