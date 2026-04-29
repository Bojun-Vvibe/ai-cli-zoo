# csvkit

- **Repo**: https://github.com/wireservice/csvkit
- **Version**: 2.2.0
- **License**: MIT ([COPYING](https://github.com/wireservice/csvkit/blob/master/COPYING))

## What it is

A suite of small Unix-style command-line tools for converting to and working with CSV. Each tool does one thing: `in2csv` converts Excel/JSON/fixed-width into CSV, `csvcut` slices columns, `csvgrep` filters rows, `csvstat` describes a column, `csvjoin` joins two files, `csvsql` runs SQL against CSVs (via SQLAlchemy, with a real DB if you give it `--db`), `csvlook` pretty-prints to a terminal table.

## Why it's in the zoo

Pandas is overkill when the question is "give me the unique values of column 4 from this 200 MB Excel export." csvkit is the awk/grep/cut family for tabular data — composable, pipe-friendly, and it understands header rows, quoting, encodings, and dialect detection so you don't have to. `in2csv weird.xlsx | csvcut -c name,email | csvgrep -c email -m '@example.com' | csvlook` is a real one-liner that would otherwise be a 20-line Python script.

## Install

```sh
brew install csvkit
# or
pipx install csvkit
```

## Quick example

```sh
in2csv sales.xlsx | csvsql --query "SELECT region, SUM(amount) FROM stdin GROUP BY region ORDER BY 2 DESC" | csvlook
```
