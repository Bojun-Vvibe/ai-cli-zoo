# pyinfra

> **Infrastructure automation as ordinary Python** — declare
> server state with idempotent operations (`apt.packages`,
> `files.template`, `systemd.service`, …) in regular `.py`
> files, run them ad-hoc against SSH hosts / Docker containers
> / the local box, and watch pyinfra diff the desired state
> against reality and apply only the changes — at speeds
> several times faster than Ansible because everything is
> async over SSH and there is no Python runtime required on
> the target. Pinned to **v3.8.0** (SPDX: `MIT`,
> [LICENSE.md](https://github.com/pyinfra-dev/pyinfra/blob/3.x/LICENSE.md)).

Source: <https://github.com/pyinfra-dev/pyinfra>

## TL;DR

`pyinfra` is a Python package (`pip install pyinfra`) that
exposes a library of declarative *operations* — each one
inspects current state on the target host (over SSH, Docker
exec, or local shell), computes the diff against the
arguments you passed, and emits the shell commands that
would converge it. You run it via the `pyinfra` CLI, point
at an inventory (an `INI` file, a `hosts.py`, an SSH host,
or `@local` / `@docker/...`), and pass either a one-shot
operation (`pyinfra @docker/ubuntu apt.packages curl`) or a
deploy script (`pyinfra inventory.py deploy.py`). The model
is "Python the language, not Python a YAML-with-Jinja DSL":
loops, conditionals, functions, imports, type hints, and
ordinary debuggers all work.

## Install

```bash
# pipx (recommended — isolated venv)
pipx install pyinfra

# pip
pip install pyinfra

# uv
uv tool install pyinfra

# verify
pyinfra --version   # pyinfra: v3.8.0
```

## License

MIT — see
[LICENSE.md](https://github.com/pyinfra-dev/pyinfra/blob/3.x/LICENSE.md).

## Representative Commands

```bash
# 1. one-shot ad-hoc operation against a Docker container
pyinfra @docker/ubuntu apt.packages curl,git update=true

# 2. run an arbitrary shell command across an inventory and
#    print stdout/stderr per host
pyinfra inventory.py exec -- uptime

# 3. apply a deploy script (preview first with --dry, then run)
pyinfra inventory.py deploy.py --dry
pyinfra inventory.py deploy.py
```

A minimal `deploy.py`:

```python
from pyinfra.operations import apt, files, systemd

apt.update(name="Refresh apt cache", cache_time=3600)
apt.packages(name="Install nginx", packages=["nginx"], update=True)

files.template(
    name="Render nginx site config",
    src="templates/site.conf.j2",
    dest="/etc/nginx/sites-enabled/site.conf",
    server_name="example.test",
)

systemd.service(
    name="Ensure nginx is running",
    service="nginx",
    running=True,
    enabled=True,
    restarted=True,
)
```

## Why It Matters

Configuration management has historically meant Ansible
(YAML + Jinja, slow, requires Python on targets), Chef/Puppet
(agents, Ruby DSL, heavy server), or shell scripts (no
idempotence, no state diff, no inventory model). `pyinfra`
takes the operations-and-inventory model from Ansible but
expresses it in straight Python: a deploy is a `.py` file
with imports and function calls, the inventory is a `.py`
file that returns a list, and the runtime is async over SSH
so a 200-host fleet converges in the time Ansible takes for
20. Targets need only a POSIX shell — no agent, no Python
interpreter installed, no bootstrap problem. For small
fleets, homelabs, ephemeral CI runners, and "I want
Ansible-style declarative ops without Ansible" cases, it's
the most ergonomic option in the ecosystem, and it composes
cleanly with the rest of the Python toolchain (mypy, pytest,
ruff, debuggers).
