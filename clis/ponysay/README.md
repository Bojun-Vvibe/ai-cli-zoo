# ponysay

> **A `cowsay` rewrite that prints a message in a
> speech-bubble next to a colourful My Little Pony ASCII /
> Unicode portrait** — ships ~400 ponies (`-l` to list,
> `-f` to pick), supports truecolour KMS / 256-colour /
> mono terminals, balloon-link styles (`-b unicode/ascii`),
> stdin piping, "quote of the pony" mode (`-q`), and a
> companion `ponythink` for the thoughtful variant; a
> drop-in joy upgrade for shell prompts, motd, and CI bot
> messages. Pinned to **3.0.3**, SPDX `GPL-3.0`,
> [LICENSE](https://github.com/erkin/ponysay/blob/master/LICENSE).

Source: <https://github.com/erkin/ponysay>

## TL;DR

`ponysay` is a community-maintained continuation of the
original `ponysay` project — a fully Unicode-aware,
truecolour-capable replacement for `cowsay` that swaps the
ASCII bovine for one of ~400 hand-curated pony portraits.
Same calling convention as `cowsay` (`echo hi | ponysay`,
or `ponysay "hi"`), so any place a `cowsay` line lives —
shell startup, motd, `fortune | cowsay` pipelines, build
success notifications — can be hot-swapped without rewiring.
Adds: `-f <name>` to pick a specific pony, `-l` to list all,
`-W <cols>` to wrap, `-b round/unicode/ascii` for balloon
style, `-q` for a quote-of-the-pony mode that pulls a
character-appropriate fortune from the bundled quote DB, and
`ponythink` for the cowthink-equivalent. Renders cleanly in
truecolour terminals (kitty, wezterm, alacritty, foot,
ghostty); gracefully degrades to 256-colour and mono.

## Install

```bash
# Arch
sudo pacman -S ponysay

# Debian / Ubuntu
sudo apt install ponysay

# Homebrew
brew install ponysay

# From source
git clone https://github.com/erkin/ponysay && cd ponysay
./setup.py --freedom=partial install

# verify
ponysay --version    # ponysay 3.0.3
```

## License

GNU GPL-3.0 — see
[LICENSE](https://github.com/erkin/ponysay/blob/master/LICENSE).
The pony artwork is bundled under permissive sub-licences
(see `LICENSE.BSD`, `LICENSE.CC-BY-SA-3.0`, etc. in the
repo); the program itself is GPLv3.

## Representative Commands

```bash
# 1. simplest form
ponysay "Hello, terminal!"

# 2. pipe from another command
fortune | ponysay

# 3. pick a specific pony
ponysay -f pinkiepie "let's bake some cupcakes"

# 4. list every available pony
ponysay -l | less

# 5. quote-of-the-pony (uses the bundled quote DB)
ponysay -q

# 6. wrap to 60 columns and use Unicode balloon
echo "build #4711 succeeded in 3m 12s" | ponysay -W 60 -b unicode

# 7. thoughtful variant
echo "should I refactor this?" | ponythink

# 8. add to ~/.zshrc for a friendly login banner
ponysay -q >> ~/.config/login-banner
```

## Niche / Category

Terminal whimsy / ASCII art / motd & notification dressing.

## Why It Is Orthogonal

The catalogue's existing whimsy tier covers different
moods: [`figlet`](../figlet/) renders **block letters** for
banners, [`cbonsai`](../cbonsai/) grows an **ambient bonsai
tree**, [`pipes-sh`](../pipes-sh/) draws **animated screen
savers**, [`hyfetch`](../hyfetch/) shows a **system
information card with pride flag colourways**, and
[`gowall`](../gowall/) is a **wallpaper-palette converter**
— none of them wrap a *message* in a character-driven
speech bubble. `ponysay` is the only entry in the
`cowsay`-family niche, and it is a strict superset (more
characters, truecolour, Unicode balloons, quote DB) that
slots into existing `cowsay` pipelines unchanged. Pairs
with [`figlet`](../figlet/) (banner above a ponysay
quote), [`fortune`](../fortune/-)-style pipelines via
`-q`, [`gum`](../gum/) (`gum spin -- long-task && echo
"done" | ponysay`), and [`noti`](../noti/) (CI bot that
posts a pony for every successful release). Reach for
ponysay when the deliverable is a *human-scale moment of
delight* — login motd, team chat bot, CI success / failure
notifier in a shared terminal, a kid-friendly demo of "the
shell is fun" — anywhere a stoic cow no longer fits the
brand. Caveat: GPLv3 means downstream redistribution must
also be GPLv3-compatible; for permissive contexts use
`cowsay` itself.
