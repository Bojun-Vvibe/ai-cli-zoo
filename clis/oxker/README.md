# oxker

- **Repo:** https://github.com/mrjackwills/oxker
- **Latest version:** v0.13.1
- **License:** MIT (`LICENSE`)
- **Category:** dev-tools / containers

A small Rust TUI for managing running Docker containers: list, inspect logs,
view live CPU/memory/network charts per container, and start/stop/restart
without leaving the keyboard. Pick oxker over `lazydocker` when you want
something lighter and more focused on day-to-day container ops, with no
images/volumes/swarm panels and a single-binary install. It speaks straight
to the Docker socket, so it also works against a remote host via
`DOCKER_HOST`.
