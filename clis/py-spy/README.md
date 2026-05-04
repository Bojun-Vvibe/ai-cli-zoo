# py-spy

- **Repo:** https://github.com/benfred/py-spy
- **Version:** v0.4.2 (latest release, 2025)
- **License:** MIT ([LICENSE](https://github.com/benfred/py-spy/blob/master/LICENSE))
- **Language:** Rust (profiles Python processes)
- **Install:** `pipx install py-spy` · `pip install py-spy` ·
  `cargo install --locked py-spy` · `brew install py-spy` ·
  prebuilt manylinux / macOS / Windows wheels on PyPI · `cargo binstall py-spy`

## What it does

`py-spy` is a sampling profiler for **already-running CPython processes**
that does not require modifying the target script, importing a profiler
module, or restarting the process. Implemented in Rust, it reads the
target process's memory directly (Linux: `process_vm_readv` /
`/proc/$pid/mem`, macOS: `mach_vm_read`, Windows: `ReadProcessMemory`),
walks the CPython interpreter's `_PyRuntime` and frame-stack structures
for every Python thread, resolves filenames + function names + line
numbers from the in-memory code objects, and emits one of three output
shapes depending on subcommand:

- `py-spy record -o profile.svg --pid <pid>` — writes a flame graph SVG
  (or `--format speedscope` for the [Speedscope](https://www.speedscope.app)
  interactive viewer, or `--format raw` for the FlameGraph.pl format)
  after sampling for `--duration` seconds. The default 100 Hz rate is
  enough resolution for almost every "where is my Python service
  spending CPU" question.
- `py-spy top --pid <pid>` — `htop`-style live TUI showing the hottest
  Python functions in the target process, refreshing in real time.
  The killer demo: SSH into a box where a Django worker is pegged at
  100% CPU, run `sudo py-spy top --pid 1234`, and within two seconds
  you are looking at the function eating the time, no code change, no
  restart, no log dive.
- `py-spy dump --pid <pid>` — one-shot text snapshot of the current
  Python call stack of every thread in the target process. The
  `gdb-attached-to-Python` story without the gdb part: useful when a
  process appears hung and you want "what is each thread blocked on
  *right now*" in the form of Python file:line traces, not C frames.

`--subprocesses` follows `multiprocessing` / `os.fork` children, so a
Gunicorn / Celery / pytest-xdist deployment with N workers profiles as
a single merged stack tree. `--native` (Linux + macOS) interleaves C /
C++ frames with the Python frames in the same flame graph, so a
NumPy-heavy workload shows you both the Python caller and the BLAS
function eating the time. `--idle` includes threads currently waiting
on I/O (off by default — most users want CPU only). `--threads` adds a
per-thread label to every stack so you can filter the Speedscope view
to one thread at a time.

## When to pick it / when not to

Pick `py-spy` whenever a Python service is slow, hung, or pegged and
you do not want to add `cProfile` / `yappi` / `scalene` decorators and
restart the process to find out why. The bar to clear is low: as soon
as the question is "where is my Python program spending its time" and
the program is already running, this is the answer. It is the right
tool for **profiling a production worker without restarting it**
(`sudo py-spy record --pid $(pgrep -f gunicorn) --duration 30
-o gunicorn.svg`), for **debugging a hung process** (`py-spy dump
--pid …` shows the exact line each thread is blocked on, no `pdb`, no
`faulthandler`, no signal handler), for **profiling a pytest run that
is mysteriously slow** (`py-spy record -o tests.svg -- pytest -q`
spawns the command and profiles it end-to-end), and for **including a
profile in a PR description** (Speedscope SVG / JSON is shareable as a
file artifact).

Skip it for **microbenchmarks under ~10 ms** — sampling at 100 Hz
gives you 0–1 samples on a 10 ms function; use `cProfile` /
`scalene` / `yappi` for deterministic instrumentation when the unit of
work is sub-tick. Skip it for **memory profiling, allocation tracking,
or import-time analysis** — `py-spy` is a CPU sampling profiler;
[`memray`](https://github.com/bloomberg/memray) /
[`scalene`](https://github.com/plasma-umass/scalene) /
[`tracemalloc`](https://docs.python.org/3/library/tracemalloc.html) /
`importtime` are the right shape for memory and import work, and they
compose with `py-spy` (run them concurrently). Skip it for **non-CPython
runtimes** — PyPy stack walks are not supported, MicroPython /
Jython / IronPython are not supported. Skip it for **async coroutine
profiling where you need to attribute time to the awaited coroutine,
not the event loop frame** — `py-spy` shows what's executing on the
thread, which is usually `asyncio.base_events.run_forever` for
async-heavy services; `scalene` and `yappi` have better async
attribution. And skip it for **profiling Python code from inside a
non-Python parent process** that embeds CPython in a way that hides
`_PyRuntime`; `py-spy` cannot find the interpreter state in that case.

The Linux production caveat: `py-spy` needs `CAP_SYS_PTRACE` (or root,
or matching UID + non-strict `ptrace_scope`). On hardened hosts you
either run it as root, set `kernel.yama.ptrace_scope=0` temporarily,
or grant the binary the capability with `setcap cap_sys_ptrace+ep
$(which py-spy)`. Inside Docker, the container needs
`--cap-add SYS_PTRACE` (or `--privileged`); inside Kubernetes the pod
needs `securityContext.capabilities.add: ["SYS_PTRACE"]`.

## AI-native angle

`py-spy` is one of the rare profilers an agent can run *against a
running service* without coordinating a restart, which makes it the
right primitive for closing the loop on a "Python service is slow"
incident:

- **The flame graph SVG is a PR artifact.** `py-spy record --pid …
  --duration 30 -o regression.svg` is a one-line command an agent can
  attach to an issue or PR; reviewers open it in a browser and see
  the same picture the agent saw. Speedscope (`--format speedscope`)
  is even better — it is fully interactive in the browser without a
  server.
- **`py-spy dump` is grep-able.** A textual call-stack snapshot
  composes with [`ripgrep`](../ripgrep/) and [`jq`](../jq/) — an
  agent can dump every worker, grep for "are any of them blocked in
  the same place", and produce a structured "12/16 workers are stuck
  in `pool.acquire`" claim for the on-call human.
- **Pairs with [`samply`](../samply/) and [`hey`](../hey/).** `hey`
  drives load, `py-spy` shows the Python call attribution while load
  is in flight, `samply` shows the C-level attribution (e.g. inside
  NumPy / asyncio's `selectors`) for the same period. Three
  profilers, one running process, no source changes.
- **`--native` mode plus a Python service is the closest thing to
  "full-stack flame graph" without instrumenting code** — the same
  picture an APM agent would draw, but with no agent install and no
  per-request overhead.

## Alternatives

- **[`scalene`](https://github.com/plasma-umass/scalene)** — Python
  CPU + GPU + memory profiler that distinguishes Python time from
  native time and provides AI-suggested optimisations. Different
  shape: `scalene` requires running your code under it
  (`scalene script.py`), which is the wrong fit for "the worker is
  already running and I cannot restart it." Pick `scalene` for
  development-time profiling on your own laptop; pick `py-spy` for
  attaching to live processes.
- **[`memray`](https://github.com/bloomberg/memray)** — Bloomberg's
  Python memory profiler with the same low-overhead, attach-by-PID
  story. Orthogonal to `py-spy` (memory vs CPU) — run both at once.
- **[`yappi`](https://github.com/sumerc/yappi)** — deterministic
  (instrumenting) profiler with first-class async/coroutine
  attribution, which is the one place `py-spy` is weakest. Pick
  `yappi` when async-await time accounting matters more than
  zero-overhead production safety.
- **[`cProfile`](https://docs.python.org/3/library/profile.html)** —
  stdlib deterministic profiler. Pick `cProfile` for "I am writing
  the script, I will add `python -m cProfile` to the command line";
  pick `py-spy` for "I cannot edit the command line that started
  the process."
- **[`samply`](../samply/)** — sampling profiler for *native* code
  (Rust / C / C++ / Go binaries) with the Firefox Profiler as
  viewer. Different layer; pair, do not replace.
- **[`austin`](https://github.com/P403n1x87/austin)** — frame-stack
  sampler in C, with similar attach-by-PID UX and lower per-sample
  overhead. The closest direct competitor; pick `austin` for the
  smaller binary and the C dependency-free build, pick `py-spy` for
  the wider ecosystem (Speedscope export, `--native` mode, more
  battle-tested at scale).
- **[`pyinstrument`](https://github.com/joerick/pyinstrument)** —
  call-stack-sampling profiler that is library-friendly (decorate a
  function, get an HTML report). Pick when you want to profile a
  *block of code inside your script*; pick `py-spy` when the unit
  of profiling is "the whole running process."
