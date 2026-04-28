# zenith

- **Repo:** https://github.com/bvaisvil/zenith
- **Latest version:** 0.14.3
- **License:** MIT (`LICENSE`)
- **Category:** dev-tools / system-monitor

A terminal system monitor that combines `top`-style process tables with
zoomable historical charts of CPU, memory, network, disk, GPU, and battery
in one screen. Choose zenith when `htop`/`btop` feel too snapshot-only and
you want to scrub back through the last few minutes of a spike without
launching a full Grafana stack. Written in Rust, runs on Linux and macOS,
keyboard-driven, and stays usable over slow SSH because it redraws
sparingly.
