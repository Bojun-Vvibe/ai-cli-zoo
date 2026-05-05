# howdoi

> **Instant coding answers from the command line** — a Python
> CLI that takes a natural-language programming question
> ("`howdoi format date python`"), scrapes the top-voted Stack
> Overflow / Stack Exchange answers, and prints the runnable
> code snippet (or the full answer with `-a`) directly to your
> terminal — no browser, no tabs, no cookie banners. Pinned to
> **v2.0.20** ([LICENSE.txt](https://github.com/gleitz/howdoi/blob/master/LICENSE.txt),
> MIT).

Source: <https://github.com/gleitz/howdoi>

## TL;DR

`howdoi <question>` is the original "AI coding assistant" from
2013, built before LLMs existed: it queries a search engine,
finds the highest-ranked Stack Overflow thread, parses the
accepted/top answer, extracts the first code block, and prints
it. With `-n 3` it shows three candidate answers; with `-a` it
prints the entire answer prose; with `-l` it prints the source
URL; with `-c` it syntax-colorizes; with `-e <engine>` it
switches between Google / Bing / DuckDuckGo / Stack Exchange's
own API.

It caches answers locally so repeat lookups are instant and
work offline. There's a `--save` / `--view` / `--remove` set of
commands that turn it into a personal snippet library — every
useful answer you found becomes a tagged entry you can recall
by keyword later.

## Why it's interesting

In an LLM-assistant world `howdoi` looks anachronistic, but
it's actually the **deterministic, sourced, attribution-preserving**
version of "give me the snippet": every answer is a real
Stack Overflow post with a real URL, a real author, a real
upvote count, and a real comment thread you can audit. There's
no hallucination surface — if the snippet doesn't compile, you
can click through and see the same broken snippet a human
posted, with the comments explaining why.

It's also self-contained (one Python package, no API keys, no
account, no quota), runs in 200 ms, and works inside locked-down
environments where you can't install or call out to a hosted LLM.
For "what's the awk incantation for ...", it's still faster than
opening a chat window.

## Install

```bash
# pipx (recommended, isolated)
pipx install howdoi

# pip
pip install howdoi

# Homebrew
brew install howdoi

# verify
howdoi --version    # 2.0.20
```

## Examples

```bash
# top snippet only
howdoi reverse a list in python

# show 3 candidate answers, each with its source URL
howdoi -n 3 -l format date in bash

# print the full answer text, not just the code block
howdoi -a git undo last commit

# colorized output
howdoi -c sed replace in file

# pick the search engine (default is google; bing/duckduckgo also work)
howdoi -e duckduckgo print env vars in fish shell

# personal snippet library
howdoi --save myreset 'git reset --hard HEAD'
howdoi --view myreset
howdoi --remove myreset

# clear the on-disk answer cache
howdoi --clear-cache
```

## Use when

- You want a one-shot, sourced answer to a tactical
  shell/code question without leaving the terminal and without
  opening a chat session.
- You're in an environment where calling a hosted LLM is
  blocked, rate-limited, or audit-unfriendly, but outbound
  HTTPS to a search engine is fine.
- You want every answer attached to a permanent URL you can
  cite in a code review or paste into a ticket.
- You want a lightweight personal snippet store (`--save` /
  `--view`) without setting up a separate tool.

Skip `howdoi` when your question is conceptual ("explain
borrow checker"), multi-turn ("now refactor that to async"),
or about your own private code — it can only return what's
already on Stack Overflow, and it can only return one snippet
at a time.
