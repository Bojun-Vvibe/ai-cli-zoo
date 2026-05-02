# reptyr

> **Reparent a running process to a new terminal** — a tiny
> C utility that uses `ptrace(2)` to detach an already-
> running program from its current controlling TTY and
> reattach it to the TTY of the calling shell, so the
> 12-hour `make` you forgot to start under `tmux` can be
> rescued without killing it. Pinned to **v0.10.0**
> ([COPYING](https://github.com/nelhage/reptyr/blob/master/COPYING),
> MIT).

Source: <https://github.com/nelhage/reptyr>

## TL;DR

`reptyr PID` is the answer to "I started a long-running job
in the wrong place and I'd like it back". You SSH'd in,
launched a 14-hour data import in the foreground, then
remembered you were not in `tmux` / `screen` / `zellij`;
your network is going to drop in two hours and the process
will die with the SSH session. `reptyr` walks `/proc/PID/fd`
to find the file descriptors pointing at the old PTY,
attaches via `ptrace`, swaps them out for descriptors
pointing at *your current* terminal, fixes up the process
group + session leader, and the program just keeps running
— now reading from your keyboard and writing to your
screen, and surviving the death of the original SSH session
because it's no longer attached to it. Combine with
[`tmux`](https://github.com/tmux/tmux) / [`zellij`](../zellij/)
/ `dtach` — start a fresh session, `reptyr` the orphaned
job into it, detach, log out, sleep. v0.10.0 (May 2024)
added riscv64 support and modernized the ptrace probe loop
for newer Linux kernels (>= 5.15). Linux only; on macOS the
SIP / Mach task-port model makes the same trick effectively
impossible, which is why this niche has no portable
replacement.

## Install

```bash
# Debian / Ubuntu (in-distro)
sudo apt install reptyr

# Fedora / RHEL
sudo dnf install reptyr

# Arch
sudo pacman -S reptyr

# Homebrew (Linuxbrew on Linux; macOS install will not work at runtime)
brew install reptyr

# Build from source (~400 lines of C, no deps beyond libc + Linux headers)
git clone --depth 1 --branch reptyr-0.10.0 \
  https://github.com/nelhage/reptyr.git
cd reptyr && make && sudo make install
```

On most distros you also need to relax `kernel.yama.ptrace_scope`
once per boot (or set it in `/etc/sysctl.d/`) so non-root can
ptrace its own children:

```bash
echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
```

## Usage

```bash
# Inside a fresh tmux pane, grab PID 12345 from another TTY
reptyr 12345

# Same, but the target was started as a shell-builtin pipeline;
# reparent the whole process group, not just the leader:
reptyr -T 12345

# Discover orphan candidates (processes whose controlling tty
# is gone, e.g. survived an SSH disconnect)
ps -eo pid,tty,stat,cmd | awk '$2=="?"'
```

## Why it's interesting

`reptyr` solves exactly one problem and solves it
completely: it is the only widely-packaged tool that does
TTY reparenting via ptrace on Linux, and it has been the
canonical answer in distro repos for over a decade. It does
*not* compete with terminal multiplexers — `tmux` /
[`zellij`](../zellij/) / `screen` / `dtach` / [`tmate`](../tmate/)
exist so you never need `reptyr` in the first place — but
it is the rescue tool for the case "I forgot to start under
a multiplexer and I cannot kill this job". It also doesn't
overlap with SSH-resilience tools like [`mosh`](../mosh/) (which
rescues the *interactive* session, not a running batch job)
or process-survival tools like `nohup` / `disown` /
`setsid` (which preserve a process across logout but
*disconnect* its TTY rather than reattach it to a new one —
you lose the ability to interact). Limitations are real and
documented: ptrace is one-shot per PID per kernel YAMA
session; processes that hold the controlling TTY in
non-FD ways (some `curses` apps, anything that opened
`/dev/tty` directly and stashed the result) can confuse the
reattach; SELinux / AppArmor profiles can deny the ptrace.
Worth installing on every long-lived Linux box you SSH into,
exactly once, against the day you need it.
