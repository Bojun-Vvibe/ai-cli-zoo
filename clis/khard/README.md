# khard

> **Console address-book manager that reads and writes
> plain vCard files on disk** — backend-agnostic (any
> directory of `.vcf` files works, including those synced
> by `vdirsyncer` from a CardDAV server), `mutt`/`alot`/
> `khal` integration via a query interface that returns
> `email <TAB> name` lines, and an editor-driven add/edit
> flow that opens `$EDITOR` on the raw vCard. Pinned to
> **v0.20.1**
> ([LICENSE](https://github.com/lucc/khard/blob/main/LICENSE),
> GPL-3.0-only).

Source: <https://github.com/lucc/khard>

## TL;DR

`khard` treats the **vCard file** (RFC 6350) as the unit
of truth: one `.vcf` per contact, in a directory you
control, on a filesystem you back up the same way you back
up everything else. There is no database, no daemon, no
cloud, no proprietary store — the whole address book is
`grep`-able, `git`-versionable, and round-trips through
any CardDAV server (Nextcloud, Radicale, Baikal, Fastmail,
iCloud) when paired with [`vdirsyncer`](https://github.com/pimutils/vdirsyncer)
as the sync layer. Multi-account is just multiple
directories listed in `~/.config/khard/khard.conf`; each
account becomes a namespace you can target with `-a
<name>` or merge across with no flag. The CLI verbs are
small and orthogonal — `khard list` (filter by query),
`khard show <query>` (full vCard fields), `khard new`
(create a new contact, `$EDITOR` opens a YAML template),
`khard edit <query>` (edit existing), `khard merge`
(combine duplicates), `khard email <query>` (the
mutt-shaped output: one `addr <TAB> name <TAB> type` line
per match, which is exactly what `set query_command =
"khard email --parsable %s"` in `~/.muttrc` consumes for
`^X^T` address completion). Phone numbers do the same
through `khard phone`. The query language is a list of
substrings matched case-insensitively across name,
nickname, organization, email, and phone — no regex
gymnastics, no SQL. Sync to a phone via CardDAV →
`vdirsyncer` → khard's directory; or skip CardDAV
entirely and `git push` the vCard directory between
machines.

## Install

```bash
# Homebrew
brew install khard

# Debian / Ubuntu
sudo apt install khard

# pipx (recommended for the latest)
pipx install khard==0.20.1

# pip
pip install --user khard==0.20.1

# Build from source
git clone --depth 1 --branch v0.20.1 \
  https://github.com/lucc/khard.git
cd khard && pip install .
```

## Usage

Minimum viable config at `~/.config/khard/khard.conf`:

```ini
[addressbooks]
[[home]]
path = ~/.contacts/home/

[general]
debug = no
default_action = list
editor = nvim, -i, NONE
merge_editor = nvimdiff

[contact table]
display = first_name
preferred_email_address_type = pref, work, home
preferred_phone_number_type  = pref, cell, work, home
show_nicknames = yes
sort = first_name

[vcard]
private_objects = Jabber, Skype, Twitter
preferred_version = 4.0
search_in_source_files = no
```

```bash
# List everyone matching "alice"
khard list alice

# Full record
khard show alice

# Add a contact (opens $EDITOR on a YAML template,
# converted back to vCard 4.0 on save)
khard new

# Mutt query hookup — in ~/.muttrc:
#   set query_command = "khard email --parsable %s"
#   bind editor <Tab> complete-query
#   bind editor ^T    complete

# alot (notmuch front-end) hookup — in ~/.config/alot/config:
#   [accounts]
#     [[default]]
#       address_book = type=external, command=khard email --parsable -, regexp=...

# Pair with khal (calendar) and vdirsyncer (sync) for the
# pimutils trifecta — one config flow, three CLI tools,
# CardDAV + CalDAV without a desktop app.
```

## Why it's interesting

The address-book slot in the terminal is small but real,
and the trade is exactly **filesystem-of-vCards vs.
backend-of-its-own**. `khard` is the canonical
filesystem-of-vCards entry: it owns the *editing* layer,
defers *sync* to `vdirsyncer`, and integrates with mail
clients via a stable text interface (`addr <TAB> name`)
that has not changed since the project started in 2013.
The closest peers are [`abook`](https://abook.sourceforge.net/)
(curses TUI, custom binary database — fast, but the data
is locked in `abook`'s format and there's no CardDAV story),
`pimutils/khal` (the calendar sibling — same authors, same
config conventions, same sync model via `vdirsyncer`), and
the various CardDAV-native TUIs that ship with their own
sync (less Unix-y, harder to script). Pick `khard` when
(a) you already use `mutt` / `neomutt` / `alot` / `aerc`
and want `^X^T` address completion to hit a real address
book without an Outlook-shaped dependency, (b) you want
your contacts version-controlled in a `git` repo so a
"who is this email from" lookup works on a flight with no
network, or (c) you're building the pimutils stack
(`vdirsyncer` + `khal` + `khard`) for a fully
file-based, CardDAV/CalDAV-synced PIM that doesn't depend
on Evolution or Thunderbird. Not the right pick if you
want a single GUI app with calendar + contacts + tasks
fused (Evolution, Thunderbird, eM Client) — the pimutils
philosophy is deliberately the opposite. Maintained
steadily since 2013 with a 0.20.x line in 2024–2025
focused on vCard 4.0 conformance and Python 3.12+
compatibility; small dependency surface (`vobject`,
`ruamel.yaml`, `unidecode`).
