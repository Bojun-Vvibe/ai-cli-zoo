# elinks

> **A full-featured terminal web browser with tables, frames,
> background downloads, tabs, mouse support, ECMAScript via
> SpiderMonkey or QuickJS, cookies, HTTPS, and a rich
> menu-driven UI** — the maintained successor to the
> long-stalled original ELinks 0.13. Pinned to **v0.19.1**
> ([COPYING](https://github.com/rkd77/elinks/blob/master/COPYING),
> GPL-2.0).

Source: <https://github.com/rkd77/elinks>

## TL;DR

`elinks` is the most full-featured of the classic terminal web
browsers (`lynx`, `links`, `links2`, `w3m`, `elinks`). Where
`lynx` is text-only and modal, `elinks` is a real ncurses
application: pulldown menus across the top, tabs, mouse
support (clickable links and form fields under `xterm` /
`tmux`), background downloads with progress, HTTP authentication
and cookies that persist across sessions, HTTPS via OpenSSL or
GnuTLS, optional bidirectional Unicode (UTF-8) rendering, table
and frame layout, and an embedded JavaScript engine (SpiderMonkey
or QuickJS) for the small subset of pages where a couple of
`onclick` handlers are the only barrier to reading the content.
The configuration is a real DSL (Lua, Guile, Perl, Ruby, or
plain `~/.config/elinks/elinks.conf`) so keybindings, content
filters, URL rewrites, and per-host behaviour are all
script-driven. The `rkd77/elinks` fork is the actively
maintained line — the original SourceForge `elinks/elinks`
saw its last release in 2012, and `rkd77` has shipped 0.14
through 0.19 with HTML5 partial support, modern TLS, modern
build system, and dozens of CVE fixes.

## Install

```bash
# Homebrew (macOS / Linux) — likely the older 0.13 unless tapped
brew install elinks

# Arch
pacman -S elinks

# Debian / Ubuntu
sudo apt install elinks

# Fedora
sudo dnf install elinks

# from source — gets you the rkd77 maintained line
git clone https://github.com/rkd77/elinks.git
cd elinks
./autogen.sh
./configure --enable-utf-8 --with-openssl --enable-html-highlight
make -j
sudo make install
```

After install:

```bash
elinks https://news.ycombinator.com   # open a URL
elinks                                # opens the Welcome page; ESC for menu
elinks -dump https://example.com      # render to stdout, exit (the lynx -dump shape)
elinks -source https://example.com    # raw HTML source to stdout
```

## What it actually is

A pure-curses HTML user agent that handles enough of HTTP +
HTML + CSS-by-style-attribute + a fragment of JS to read the
text-content half of the modern web from a TTY:

- **Real menu UI.** `Esc` opens the pulldown menu bar
  (`File`, `View`, `Link`, `Tools`, `Setup`, `Help`); every
  action has a keybinding *and* a clickable menu entry, so
  newcomers don't have to memorise modal commands the way
  `lynx` requires.
- **Tabs.** `t` opens a new tab, `T` opens the link under
  cursor in a new tab, `<` / `>` cycles, `c` closes — the same
  shape as a graphical browser.
- **Forms + cookies.** Login flows that don't require JS
  work: submit a form, the cookie is stored in
  `~/.config/elinks/cookies`, the next request to the host
  sends it back, sessions persist across `elinks` restarts.
  Session restore (`Tools → Settings Manager → Document →
  Browsing → Resume previous session`) is on by default.
- **Background downloads.** Pressing `d` on a link starts a
  download in the background with a progress meter; `Tools →
  Downloads` shows the queue. Useful for large files over an
  SSH session where `wget` is not installed but `elinks` is.
- **Optional embedded JS.** Built with
  `--enable-ecmascript=quickjs` or
  `--enable-ecmascript=spidermonkey` to handle login forms
  whose submit button has an `onclick` handler. Not enough
  to render a SPA, plenty to read a paywalled article whose
  hide-content overlay is one JS line.
- **Image rendering opt-in.** With `--enable-libsixel`
  inline images render via SIXEL on supporting terminals
  (`xterm` with sixel, `wezterm`, `foot`, `mlterm`,
  `iterm2` >= 3.5).

## When to choose

- **You need to read the web from an SSH session, a tty1
  recovery shell, or a constrained container** where no GUI
  browser is available and the content is mostly text.
  `elinks -dump URL` is the canonical "render this page to
  text and pipe to `less` / `bat` / `glow`".
- **You want a one-binary HTTP client + form-filler that
  handles cookies + sessions across runs**, without writing
  a `curl` script that re-implements cookie jars and form
  encoding by hand.
- **You want a maintained terminal browser.** `lynx`
  development is glacial, `links2` is similar shape but no
  tabs / no JS, `w3m` excels at images-via-w3mimgdisplay
  but its UI conventions are unique to itself; `rkd77/elinks`
  is the active fork with modern TLS and steady releases.
- **You want CSS-aware text rendering.** `elinks` honours
  `style="color:..."` and a subset of stylesheet rules, so
  pages that use colour to convey meaning (diff views,
  syntax-highlighted code blocks) render with colour rather
  than collapsing to monochrome the way `lynx` does.

## Vs already cataloged

- **Vs [`browsh`](../browsh/) (Mozilla, MPL-2.0):** `browsh`
  embeds headless Firefox and renders its framebuffer to a
  TTY via Unicode block characters, so it handles arbitrary
  modern JS-heavy SPAs at the cost of a ~300 MB Firefox
  install and seconds of cold start. `elinks` is ~2 MB,
  starts instantly, and reads the half of the web that is
  text without needing Firefox at all. Pick `browsh` for
  Gmail / Google Docs / Notion; pick `elinks` for HN, MDN,
  documentation sites, blogs, mailing-list archives, and
  most of the static web.
- **Vs [`amfora`](../amfora/) / [`bombadillo`](../bombadillo/):**
  orthogonal — those are Gemini / Gopher / Finger clients,
  `elinks` is HTTP / HTTPS / FTP / file:// / news://. Run
  both: amfora for `gemini://`, elinks for `https://`.
- **Vs [`muffet`](../muffet/) / `wget --recursive`:**
  orthogonal — those are crawlers / link-checkers; `elinks`
  is the interactive reader. The `elinks -dump` mode is the
  closest scriptable shape (one URL → one text rendering).
- **Vs [`htmlq`](../htmlq/) + [`strip-tags`](../strip-tags/) +
  `curl`:** that pipeline gives you machine-friendly extract
  of named elements; `elinks -dump` gives you the
  human-friendly *rendered* version (paragraphs reflowed,
  links footnoted, tables laid out as boxes) of the whole
  page. Both are useful — `htmlq` to feed an LLM, `elinks
  -dump` to feed a human.

## Caveats

- **No SPA support.** A page whose content is rendered
  entirely client-side (`<div id="root"></div>` and 3 MB of
  JS) is blank in `elinks` even with the JS engine compiled
  in — the engine handles individual handlers, not React /
  Vue / Svelte rehydration. Use `browsh` or `chromium
  --headless --dump-dom` for those.
- **HTTPS depends on the build.** Many distro packages ship
  `elinks` linked against an older OpenSSL or GnuTLS; modern
  TLS 1.3-only sites may fail until you rebuild. The
  `rkd77/elinks` source builds against current OpenSSL 3.x
  cleanly.
- **Two ELinks lines exist.** `elinks/elinks` on SourceForge
  (the original, last release 2012) is what most distro
  packages historically tracked. `rkd77/elinks` on GitHub is
  the maintained line — verify `elinks -version` shows
  `0.19.x` to confirm you have the modern fork.
- **GPL-2.0 only, not 2.0+.** Worth a glance for shops with
  strict licence compatibility audits; the source COPYING is
  unambiguous about the version pin.
- **Unicode requires the `--enable-utf-8` build flag.**
  Distro packages usually enable it; source builds need the
  flag explicitly. Without it, non-ASCII pages render as
  `?`.
