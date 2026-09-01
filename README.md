# keyword-planner-parser

Reads a Google Keyword Planner CSV export into clean Python objects, handling
every way that file is not what it claims to be. Zero dependencies.

```python
from keyword_planner.parser import load_gkp, clean_keywords

rows, report = load_gkp("Keyword Stats 2026-08-01.csv")
keywords = clean_keywords(rows, report, min_volume=100)

print(report.summary())
# read 1,482 rows; 63 dropped (no volume), 11 dropped (duplicate)
# volumes: 940 exact, 542 bucketed ranges
```

## Why this exists

You export keywords from Google Keyword Planner, point `pandas.read_csv` at it,
and get a `UnicodeDecodeError`. You fix the encoding and get one column. You fix
the delimiter and your headers are three rows of report metadata. You skip those
and the headers are in Slovenian. You map those and every volume is the string
`"1 tis. – 10 tis."` instead of a number.

Each of those is ten minutes and a search that turns up nothing useful. This
module is those problems already solved.

## What the export actually does

| The export claims | The export does |
|---|---|
| UTF-8 CSV | **UTF-16 with tab delimiters** |
| Header on row 1 | Two or three rows of **report metadata** first |
| English headers | **Headers localised** to the account's interface language |
| Numeric volumes | **Bucketed ranges** — `"1K – 10K"`, `"1 tis. – 10 tis."` — for any account not actively spending |
| `1234.5` | Whatever **decimal and thousands separators** the locale prefers |

The bucketed-volume one catches people out most. If the account has no active
spend, Google will not give you a number at all — every volume is a range with a
localised magnitude suffix. This parser resolves a range to its **geometric
mean**, not its arithmetic mean: search volume is log-distributed, and the
midpoint of `1K–10K` is much closer to 3,162 than to 5,500. Getting that wrong
systematically overstates the small end of your keyword set.

## What it does

- **Sniffs the encoding and delimiter** rather than assuming. UTF-16 LE/BE with
  or without BOM, UTF-8, tab or comma.
- **Finds the header row** by looking for one that matches known field aliases,
  rather than skipping a fixed number of lines — the preamble length varies.
- **Maps localised headers.** English and Slovenian ship; adding a language is
  adding a row to one dict.
- **Parses volumes**, exact or bucketed, with localised magnitude suffixes
  (`K`, `tis.`, `M`, `mio.`).
- **Normalises numbers** across locale separator conventions.
- **Never raises on one bad row.** A single unparseable row is counted and
  skipped. A file that cannot be understood *at all* fails loudly, and the error
  **names the headers it did find** — because that is the one case where
  guessing costs you an afternoon.
- **Reports the funnel**: rows read, rows dropped and why, how many volumes were
  exact versus bucketed. A keyword set that quietly lost 40% of its rows looks
  identical to one that did not.

## Usage

```bash
pip install -e .          # no dependencies; Python 3.11+
python -m pytest          # 71 tests
```

```python
from keyword_planner.parser import load_gkp, clean_keywords

rows, report = load_gkp(path)

keywords = clean_keywords(
    rows, report,
    min_volume=100,       # drop the long tail
    dedupe=True,          # normalised-form duplicates, highest volume wins
)

for keyword in keywords[:5]:
    print(keyword.keyword, keyword.volume_mid, keyword.is_bucketed)
```

Every row carries its raw text alongside the parsed value, so you can always see
what the file actually said.

## Notes and limitations

**Two languages, by design not by limit.** English and Slovenian are what the
source engagement needed. `HEADER_ALIASES` is a plain dict — a third language is
a pull request with one row in it, and the tests will tell you if you got it
wrong.

**Geometric mean is a choice, and it is the right one**, but it is a choice. If
you need the arithmetic midpoint, `volume_low` and `volume_high` are both on the
row.

**This is the ingestion half only.** Joining keywords to URLs — ownership, gaps,
cannibalisation — lives in [seocrawl](https://github.com/LukaHakl/seocrawl),
which is where this code was extracted from. Split out because the parsing is
useful entirely on its own and nobody should have to install a crawler to read a
CSV.

**Provenance.** Extracted from a working SEO audit tool that ran against live
client sites; every quirk handled here was hit in a real export, not
anticipated. The 71 tests run against synthetic fixtures in both languages.

## Licence

MIT — see [LICENSE](LICENSE).
