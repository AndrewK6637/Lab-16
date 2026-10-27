# FIN 43900 - Lab 16 Checkout: Provenance Gate

This project is for educational and research purposes only.

## D - Define

The question is what information was actually available on March 31, 2026.
I am comparing the fund returns calculated from prices available on that date
with returns calculated using later publications and revisions. Using a price
before it was published would introduce look-ahead bias.

## R - Represent

The dataset has 12 rows. Each row identifies the fund, the month its price
measures, when that price was published, and the price itself. I keep the original
row numbers so it is clear which records get removed and why.

The main thing to watch is that month measured and published on mean different
things. Month measured is the period the price describes. Published on is when
that price became available. A price for March does not automatically mean I
could use it on March 31.

There are several issues in this table:

- Row 11 has no ticker, so there is no way to assign its price to a fund.
- Rows 6 and 7 contain the same data. Row 7 is a duplicate even though its row number is different.
- Rows 3, 8, and 10 were published after March 31. Row 10 is especially easy to miss because it measures February but was not published until April 5.
- Rows 2 and 4 give different February prices for AAA. These are different publications, not exact duplicates.
- Rows 10 and 12 give different February prices for CCC. Only row 12 was available by the decision date.

The suspicious part is not just missing or repeated data. Some rows look usable
because of their measurement month, but their publication dates show that they
were not available yet. The assignment treats the different prices as revisions;
the table does not explain the real-world reason for those revisions.

## I - Implement

The program removes the missing ticker first, then the exact duplicate, then the
late publications. After that, it keeps the latest remaining publication for
each fund and month. This leaves one eligible price per fund per month.

The naive calculation starts separately from the original records. It still
removes missing tickers and duplicates and selects the latest revisions, but it
does not apply the March 31 publication cutoff. That separation makes the
comparison meaningful instead of accidentally giving both calculations the
same filtered data.

The return is `(latest available price / previous available price - 1) * 100`.
The calculation keeps full precision and rounds the displayed return to two
decimal places. If there is not enough history or the previous price is zero,
the program reports that the return is unavailable.

## March 31 results

| Rule | Input rows | Removed | Output rows | Original removed IDs |
|---|---:|---:|---:|---|
| 1: Missing tickers | 12 | 1 | 11 | 11 |
| 2: Exact duplicates | 11 | 1 | 10 | 7 |
| 3: Publication cutoff | 10 | 3 | 7 | 3, 8, 10 |
| 4: Latest available vintage | 7 | 1 | 6 | 2 |

Reconciliation: 12 original rows - 6 removed rows = 6 surviving rows.

| Original row | Ticker | Month measured | Published on | Price |
|---|---|---|---|---:|
| 1 | AAA | 2026-01 | 2026-02-10 | 101.0 |
| 4 | AAA | 2026-02 | 2026-03-25 | 103.0 |
| 5 | BBB | 2026-01 | 2026-02-10 | 98.0 |
| 6 | BBB | 2026-02 | 2026-03-10 | 97.0 |
| 9 | CCC | 2026-01 | 2026-02-10 | 110.0 |
| 12 | CCC | 2026-02 | 2026-03-10 | 111.0 |

Signal = `(latest available price / previous available price - 1) * 100`.
Values below are fund returns computed from the supplied training prices.

| Ticker | As-of calculation | As-of signal/month | Naive calculation | Naive signal/month |
|---|---|---|---|---|
| AAA | (103/101 - 1) * 100 | +1.98%, February 2026 | (109/103 - 1) * 100 | +5.83%, March 2026 |
| BBB | (97/98 - 1) * 100 | -1.02%, February 2026 | (92/97 - 1) * 100 | -5.15%, March 2026 |
| CCC | (111/110 - 1) * 100 | +0.91%, February 2026 | (112/110 - 1) * 100 | +1.82%, February 2026 (revised) |

AAA's March measurement was published April 10. CCC's later February revision
was published April 5. Neither was knowable on March 31. Rule 4 selects AAA's
March 25 February revision of 103 instead of its March 10 value of 104.


## V - Validate

All 11 automated tests passed. The checks cover the removed row numbers, the
six surviving rows, both sets of returns, their months, rounding, same-day
publication, duplicates, revisions, ties, input order, unchanged original data,
and missing history. The row counts reconcile: 12 minus 6 removed equals 6 kept.

The independent April 15 test also passed. At that date, the as-of and naive
returns are identical: AAA is +5.83% for March, BBB is -5.15% for March, and CCC
is +1.82% for February. All publications in this table had become available by
April 15. The submitted program still defaults to March 31.

This is automated validation. A partner swap is not documented here.

## E - Evaluate the change

Moving the cutoff to April 15 changes what information can be used. It does not
change the return formula. AAA and BBB can now use their March prices, and CCC
can use its revised February price of 112.

A later date would not always make the two approaches match. If the original
file contained publications after that new date, the naive calculation could
still use information the as-of calculation could not.

## R - Reflection

The main takeaway is that a number can describe the past without being available
at the time. I would check publication dates before trusting a historical return
signal. Otherwise, a backtest can make a decision look better informed than it
could actually have been.

AAA makes the problem clear. Its +5.83% return uses the March price of 109, which
was published April 10. That return was not knowable on March 31. The return
available then was +1.98%, based on February's revised price of 103 and January's
price of 101.

Rule 4 is what changes AAA's February price from 104 to 103. The March 25
publication replaces the March 10 publication for the same month. Both were
available by March 31, so the later one is the relevant vintage.

CCC shows why a revision can cause look-ahead bias even without changing the
measurement month. Its as-of February price is 111, giving +0.91%. The naive
calculation uses 112, published April 5, and gets +1.82%. Both returns describe
February, but only one was available on March 31.

The cutoff has to come before vintage selection. If I picked CCC's latest
February publication first, I would keep the April 5 row and discard the March
10 row. Removing the late row afterward would leave no February price, even
though an eligible February price existed.

A naive backtest might claim a March 31 decision could use AAA's +5.83% return
or CCC's revised +1.82% return. That would give the strategy information from
the future. These calculations alone do not prove that a trading strategy
would be profitable.

## How rule 4 works

Rule 4 groups rows by both ticker and month. It keeps the latest publication
inside each group after the date cutoff has already been applied. It does not
keep just one row per ticker because the earlier month is still needed to
calculate the return.

```python
winners = {}
for row in records:
    key = (row['ticker'], row['month_measured'])
    rank = (date.fromisoformat(row['published_on']), -row['row_id'])
    if key not in winners or rank > winners[key][0]:
        winners[key] = (rank, dict(row))
return [winners[key][1] for key in sorted(winners)]
```

- `winners = {}` creates a place to store the chosen row for each group.
- `for row in records` checks each eligible row.
- `key` identifies the fund and the month together.
- `rank` compares publication dates first. The negative row ID makes the smaller original ID win if dates tie.
- The `if` statement accepts the first row in a group or a row with a higher rank.
- `winners[key] = ...` saves a copy of that row along with its rank, keeping its original ID.
- The last line returns the selected rows sorted by ticker and month.

## Program

```python
"""FIN 43900 Lab 16: point-in-time provenance gate; standard library only.

This project is for educational and research purposes only.
"""

# Editable inputs. Row IDs permanently identify original records.
DECISION_DATE = '2026-03-31'
TRAINING_TABLE = """row_id,ticker,month_measured,published_on,price
1,AAA,2026-01,2026-02-10,101.0
2,AAA,2026-02,2026-03-10,104.0
3,AAA,2026-03,2026-04-10,109.0
4,AAA,2026-02,2026-03-25,103.0
5,BBB,2026-01,2026-02-10,98.0
6,BBB,2026-02,2026-03-10,97.0
7,BBB,2026-02,2026-03-10,97.0
8,BBB,2026-03,2026-04-10,92.0
9,CCC,2026-01,2026-02-10,110.0
10,CCC,2026-02,2026-04-05,112.0
11,,2026-02,2026-03-10,99.0
12,CCC,2026-02,2026-03-10,111.0
"""

import csv
from datetime import date
from io import StringIO


def read_records(table=TRAINING_TABLE):
    records = list(csv.DictReader(StringIO(table)))
    for row in records:
        row['row_id'] = int(row['row_id'])
        row['price'] = float(row['price'])
    return records


def quarantine(records):
    """Rule 1: None, missing keys, empty strings and whitespace are blank."""
    return [dict(r) for r in records if r.get('ticker') and r['ticker'].strip()]


def deduplicate(records):
    """Rule 2: compare data fields; lowest original ID wins, not list order."""
    seen = set()
    kept = []
    for row in sorted(records, key=lambda r: r['row_id']):
        key = (row['ticker'], row['month_measured'], row['published_on'], row['price'])
        if key not in seen:
            seen.add(key)
            kept.append(dict(row))
    return kept


def latest_vintages(records):
    """Rule 4: latest publication per ticker/month; tied dates use lowest ID."""
    winners = {}
    for row in records:
        key = (row['ticker'], row['month_measured'])
        rank = (date.fromisoformat(row['published_on']), -row['row_id'])
        if key not in winners or rank > winners[key][0]:
            winners[key] = (rank, dict(row))
    return [winners[key][1] for key in sorted(winners)]


def asof_panel(records, decision_date):
    cutoff = date.fromisoformat(decision_date)
    current = list(records)
    funnel = []
    rules = (
        quarantine,
        deduplicate,
        lambda rows: [r for r in rows if date.fromisoformat(r['published_on']) <= cutoff],
        latest_vintages,
    )
    for rule in rules:
        kept = rule(current)
        kept_ids = {r['row_id'] for r in kept}
        removed = sorted(r['row_id'] for r in current if r['row_id'] not in kept_ids)
        funnel.append({'before': len(current), 'after': len(kept), 'removed': removed})
        current = kept
    return current, funnel


def naive_panel(records):
    # Start independently from ORIGINAL records. Reusing the as-of panel would
    # lose future releases/revisions and hide the very look-ahead bias we test.
    return latest_vintages(deduplicate(quarantine(records)))


def signals(panel):
    """Fund return: price change between last two available months, not annualized."""
    grouped = {}
    for row in panel:
        grouped.setdefault(row['ticker'], []).append(row)
    result = {}
    for ticker, rows in sorted(grouped.items()):
        rows = sorted(rows, key=lambda r: r['month_measured'])
        latest = rows[-1]
        value, reason = None, 'insufficient history'
        if len(rows) >= 2:
            previous = rows[-2]['price']
            if previous == 0:
                reason = 'previous price is zero'
            else:
                value = (latest['price'] / previous - 1) * 100
                reason = ''
        result[ticker] = {'month': latest['month_measured'], 'percent': value, 'reason': reason}
    return result


def format_signal(signal):
    if signal is None:
        return 'N/A (no available observations)'
    month = date.fromisoformat(signal['month'] + '-01').strftime('%B %Y')
    value = signal['percent']
    return (f'{value:+.2f}%, {month}' if value is not None
            else f"N/A ({signal['reason']}), {month}")


def main():
    records = read_records()
    panel, funnel = asof_panel(records, DECISION_DATE)
    naive = naive_panel(records)
    print(f'Decision date: {DECISION_DATE}\nROW FUNNEL')
    for number, step in enumerate(funnel, 1):
        print(f"Rule {number}: {step['before']} -> {step['after']}; "
              f"removed {len(step['removed'])}; original rows {step['removed']}")
    print('\nAS-OF PANEL\n row  ticker  month     published    price')
    for r in panel:
        print(f"{r['row_id']:4}  {r['ticker']:6}  {r['month_measured']}  "
              f"{r['published_on']}  {r['price']:7.1f}")
    actual, biased = signals(panel), signals(naive)
    print('\nSIGNALS (fund returns from price changes, %)')
    print(f"{'Ticker':8} {'AS-OF':30} NAIVE (includes later publications/revisions)")
    for ticker in sorted(set(actual) | set(biased)):
        print(f'{ticker:8} {format_signal(actual.get(ticker)):30} {format_signal(biased.get(ticker))}')


if __name__ == '__main__':
    main()
```

## Automated tests

```python
"""Independent expected answers exercising the actual provenance functions."""
import copy
import io
import random
import unittest
from contextlib import redirect_stdout
import asof


class ProvenanceTests(unittest.TestCase):
    def setUp(self):
        self.rows = asof.read_records()

    def test_known_funnel_and_panel(self):
        panel, funnel = asof.asof_panel(self.rows, '2026-03-31')
        self.assertEqual([s['removed'] for s in funnel], [[11], [7], [3, 8, 10], [2]])
        self.assertEqual([s['before'] for s in funnel], [12, 11, 10, 7])
        self.assertEqual([s['after'] for s in funnel], [11, 10, 7, 6])
        self.assertEqual([r['row_id'] for r in panel], [1, 4, 5, 6, 9, 12])

    def test_both_signals_months_and_rounding(self):
        panel, _ = asof.asof_panel(self.rows, '2026-03-31')
        expected = [(panel, {'AAA': (103/101-1, '+1.98%, February 2026'),
                             'BBB': (97/98-1, '-1.02%, February 2026'),
                             'CCC': (111/110-1, '+0.91%, February 2026')}),
                    (asof.naive_panel(self.rows),
                     {'AAA': (109/103-1, '+5.83%, March 2026'),
                      'BBB': (92/97-1, '-5.15%, March 2026'),
                      'CCC': (112/110-1, '+1.82%, February 2026')})]
        for data, answers in expected:
            actual = asof.signals(data)
            for ticker, (value, display) in answers.items():
                with self.subTest(ticker=ticker, display=display):
                    self.assertAlmostEqual(actual[ticker]['percent'], value*100, places=10)
                    self.assertEqual(asof.format_signal(actual[ticker]), display)

    def test_april_convergence(self):
        panel, funnel = asof.asof_panel(self.rows, '2026-04-15')
        self.assertEqual(funnel[2]['removed'], [])
        self.assertEqual(panel, asof.naive_panel(self.rows))
        self.assertEqual(asof.signals(panel), asof.signals(asof.naive_panel(self.rows)))

    def test_inclusive_cutoff(self):
        row = dict(self.rows[0], published_on='2026-03-31')
        self.assertEqual(asof.asof_panel([row], '2026-03-31')[0], [row])
        self.assertEqual(asof.asof_panel([row], '2026-03-30')[0], [])

    def test_missing_ticker(self):
        for ticker in (None, '', '   ', '\t'):
            self.assertEqual(asof.quarantine([dict(self.rows[0], ticker=ticker)]), [])
        row = dict(self.rows[0])
        del row['ticker']
        self.assertEqual(asof.quarantine([row]), [])

    def test_duplicate_fields_and_original_id(self):
        row = self.rows[5]
        duplicate = dict(row, row_id=100)
        changed = dict(row, row_id=101, price=96.0)
        self.assertEqual([r['row_id'] for r in asof.deduplicate([duplicate, changed, row])], [6, 101])

    def test_revision_and_tie(self):
        original, revised = self.rows[1], self.rows[3]
        tied = dict(revised, row_id=99, price=900.0)
        for rows in ([tied, original, revised], [revised, original, tied]):
            self.assertEqual(asof.latest_vintages(rows), [revised])

    def test_order_independence_and_no_mutation(self):
        saved = copy.deepcopy(self.rows)
        expected = asof.asof_panel(self.rows, '2026-03-31')
        naive = asof.naive_panel(self.rows)
        for seed in range(10):
            shuffled = copy.deepcopy(self.rows)
            random.Random(seed).shuffle(shuffled)
            before = copy.deepcopy(shuffled)
            self.assertEqual(asof.asof_panel(shuffled, '2026-03-31'), expected)
            self.assertEqual(asof.naive_panel(shuffled), naive)
            asof.signals(shuffled)
            self.assertEqual(shuffled, before)
        self.assertEqual(self.rows, saved)

    def test_insufficient_history_and_zero(self):
        self.assertIsNone(asof.signals(self.rows[:1])['AAA']['percent'])
        rows = [dict(self.rows[0], price=0), self.rows[1]]
        self.assertEqual(asof.signals(rows)['AAA']['reason'], 'previous price is zero')
        self.assertEqual(asof.signals([]), {})
        self.assertIn('no available', asof.format_signal(None))

    def test_previous_available_month(self):
        rows = [dict(self.rows[0], price=100), dict(self.rows[2], price=105)]
        self.assertAlmostEqual(asof.signals(rows)['AAA']['percent'], 5.0)
        self.assertEqual(asof.signals(rows)['AAA']['month'], '2026-03')

    def test_default_and_printed_output(self):
        self.assertEqual(asof.DECISION_DATE, '2026-03-31')
        output = io.StringIO()
        with redirect_stdout(output):
            asof.main()
        for text in ('+1.98%', '-1.02%', '+0.91%', '+5.83%', '-5.15%', '+1.82%'):
            self.assertIn(text, output.getvalue())


if __name__ == '__main__':
    unittest.main()
```

## Recorded validation output

```text
APRIL 15: independent edited source copy
COMMAND: python asof.py
Decision date: 2026-04-15
ROW FUNNEL
Rule 1: 12 -> 11; removed 1; original rows [11]
Rule 2: 11 -> 10; removed 1; original rows [7]
Rule 3: 10 -> 10; removed 0; original rows []
Rule 4: 10 -> 8; removed 2; original rows [2, 12]

AS-OF PANEL
 row  ticker  month     published    price
   1  AAA     2026-01  2026-02-10    101.0
   4  AAA     2026-02  2026-03-25    103.0
   3  AAA     2026-03  2026-04-10    109.0
   5  BBB     2026-01  2026-02-10     98.0
   6  BBB     2026-02  2026-03-10     97.0
   8  BBB     2026-03  2026-04-10     92.0
   9  CCC     2026-01  2026-02-10    110.0
  10  CCC     2026-02  2026-04-05    112.0

SIGNALS (fund returns from price changes, %)
Ticker   AS-OF                          NAIVE (includes later publications/revisions)
AAA      +5.83%, March 2026             +5.83%, March 2026
BBB      -5.15%, March 2026             -5.15%, March 2026
CCC      +1.82%, February 2026          +1.82%, February 2026

EXIT: 0

PASS: all three April 15 as-of signals equal naive signals.
PASS: delivered source retains March 31 unchanged.
COMMAND: python asof.py
Decision date: 2026-03-31
ROW FUNNEL
Rule 1: 12 -> 11; removed 1; original rows [11]
Rule 2: 11 -> 10; removed 1; original rows [7]
Rule 3: 10 -> 7; removed 3; original rows [3, 8, 10]
Rule 4: 7 -> 6; removed 1; original rows [2]

AS-OF PANEL
 row  ticker  month     published    price
   1  AAA     2026-01  2026-02-10    101.0
   4  AAA     2026-02  2026-03-25    103.0
   5  BBB     2026-01  2026-02-10     98.0
   6  BBB     2026-02  2026-03-10     97.0
   9  CCC     2026-01  2026-02-10    110.0
  12  CCC     2026-02  2026-03-10    111.0

SIGNALS (fund returns from price changes, %)
Ticker   AS-OF                          NAIVE (includes later publications/revisions)
AAA      +1.98%, February 2026          +5.83%, March 2026
BBB      -1.02%, February 2026          -5.15%, March 2026
CCC      +0.91%, February 2026          +1.82%, February 2026

EXIT: 0

COMMAND: python -m unittest -v test_asof.py
test_april_convergence (test_asof.ProvenanceTests.test_april_convergence) ... ok
test_both_signals_months_and_rounding (test_asof.ProvenanceTests.test_both_signals_months_and_rounding) ... ok
test_default_and_printed_output (test_asof.ProvenanceTests.test_default_and_printed_output) ... ok
test_duplicate_fields_and_original_id (test_asof.ProvenanceTests.test_duplicate_fields_and_original_id) ... ok
test_inclusive_cutoff (test_asof.ProvenanceTests.test_inclusive_cutoff) ... ok
test_insufficient_history_and_zero (test_asof.ProvenanceTests.test_insufficient_history_and_zero) ... ok
test_known_funnel_and_panel (test_asof.ProvenanceTests.test_known_funnel_and_panel) ... ok
test_missing_ticker (test_asof.ProvenanceTests.test_missing_ticker) ... ok
test_order_independence_and_no_mutation (test_asof.ProvenanceTests.test_order_independence_and_no_mutation) ... ok
test_previous_available_month (test_asof.ProvenanceTests.test_previous_available_month) ... ok
test_revision_and_tie (test_asof.ProvenanceTests.test_revision_and_tie) ... ok

----------------------------------------------------------------------
Ran 11 tests in 0.012s

OK

EXIT: 0
```

## References

- [Week 10 handout](https://github.com/CinderZhang/FIN43900-Fall2026/blob/main/lessons/week-10/student-handout.md)
- [Week 10 prework](https://github.com/CinderZhang/FIN43900-Fall2026/blob/main/lessons/week-10/student-prework.md)
- [ALFRED Help](https://alfred.stlouisfed.org/help)
