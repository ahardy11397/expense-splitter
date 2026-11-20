# expense-splitter

`split` — a shared-expense splitter ("Splitwise for a house") written 100% in
[Memelang](https://github.com/). Beta demonstration program: graph algorithms
(debt simplification via greedy min-cash-flow) plus ledger bookkeeping.

All money is **integer cents** internally; dollar amounts only ever appear in
output strings, rendered by integer arithmetic (`money/1` in
`split_logic.meme` — no floats anywhere in the program).

## Files

| File | Purpose |
|---|---|
| `main.meme` | Thin argv router: parses the CLI, loads/saves `data/*.json`, calls into `split_logic` |
| `split_logic.meme` | Pure logic module: records (`Person`, `Expense`, `Payment`, `Transfer`, `ExpenseRow`) and all splitting / netting / settle / ledger functions |
| `tests/meme/*.meme` | Four test suites (27 cases), run with `meme test` |
| `tests/meme/split_logic.meme` | Symlink to the root module (modules resolve relative to the importing file) |
| `data/` | JSON persistence: `people.json`, `expenses.json`, `payments.json` (created on first write) |
| `*.meme.lock` | Deterministic lockfiles (zero-LLM rebuilds) |

## Build

```bash
uv run --project /path/to/meme meme build main.meme --provider heuristic -o split_bin
./split_bin ...
```

The binary is a single static C11 build; all `use split_logic` modules are
linked in. Exit codes propagate directly (`1` on every error). Pass
`--provider heuristic` explicitly: the default provider is an LLM frontend,
while the heuristic parser is the zero-dependency offline path this project
targets.

## CLI usage

```
split add-person <name>
split add-expense <payer> <total-cents> <name1> <name2> ... [split-mode]
split pay <from> <to> <cents>        # a settlement payment
split balances                       # net balance per person, sorted by name
split settle                         # minimum transfers that clear all debts
split ledger                         # full history, sorted by entry id
```

`total-cents`, amounts, and weights are integer cents. `balances` prints
`<name> <dollars>` (positive = the person is owed money; negative = they owe).

### Split modes

- **equal** (default, also accepted as an explicit keyword) — `total / n` each;
  a non-divisible remainder is distributed one cent each to participants in
  **name-sorted order** until the total is exact. Example: 3 people, 100
  cents → 34/33/33 with the first leftover cent going to the
  alphabetically-first participant.
- **exact** `<c1> <c2> ...` — one amount per participant, in argument order;
  must sum exactly to `total` (else error).
- **shares** `<w1> <w2> ...` — proportional to positive integer weights:
  floor(`total * w / sum(w)`) each, then the same name-sorted leftover-cent
  distribution. Example: 101 cents, equal weights → 34/34/33 by name order.

`equal`, `exact`, and `shares` are reserved mode keywords: a participant
cannot be named that (an argument matching one of them starts the mode
section), and an explicit `equal` keyword accepts no trailing amounts.

### Settle algorithm

Computes net balances, then repeatedly settles the **largest creditor**
against the **largest debtor**, emitting a transfer *from the debtor to the
creditor* for `min(creditor.net, -debtor.net)`. Ties break by name
(alphabetically-first person wins, because the scan runs in sorted key order
with strict comparisons). Every transfer zeroes at least one account, so the
result is at most `n-1` transfers and is the minimum for any balance set with
no exactly-pairable debtor/creditor amounts. After printing, `settle`
**replays** the transfers through the same arithmetic as a `pay` entry and
asserts every balance is exactly zero.

### Data files

State lives in `data/people.json`, `data/expenses.json`, `data/payments.json`.
Every read is an `Optional` record reader (`read json records ... or else
nothing`): a missing, unreadable, corrupt, or schema-mismatched file degrades
to an empty list — never a traceback. Note that `expenses.json` stores
primitive-only rows (`ExpenseRow`) with CSV-encoded participant/amount lists,
because JSON record I/O requires primitive fields. Entry ids are shared across
expenses and payments (`next_entry_id` = max id + 1).

### Error handling

Every error is one line (prefixed `Error:`) and exit code 1 — unknown
command, wrong argument count, unknown person, duplicate person/participant,
non-positive or non-integer cents, exact-mode sum mismatch, share weight ≤ 0,
self-payment. Unknown commands print usage.

## Tests

```bash
uv run --project /path/to/meme meme test tests/meme --provider heuristic --target python
# and --target rust|cpp|go|c|ts for the other backends
```

27 cases across `tests/meme/split_core_test.meme`, `netting_test.meme`,
`settle_test.meme`, and `data_test.meme`: equal split exact division; equal
split with remainder (34/33/33 asserted per participant); exact mode and
sum-mismatch rejection; shares proportional math and name-sorted remainder;
multi-expense netting both directions; payment netting; balances always sum
to zero; hand-computed 4-person minimum settle (count **and** amounts);
replaying settle transfers zeroes all balances; settle of a clear ledger;
money formatting; entry-id counter; JSON round-trips; missing-file, corrupt-JSON,
and schema-mismatch tolerance. Test scratch files are unique-named and deleted
in the same block (test blocks run in an isolated temp working directory).

## Example session

```
$ ./split_bin add-person ann
Added person ann
$ ./split_bin add-expense ann 10000 ann bob carl dave
Added expense 1: ann paid 100.00 for ann, bob, carl, dave (equal)
$ ./split_bin add-expense carl 6000 dave ann carl exact 1000 2000 3000
Added expense 2: carl paid 60.00 for dave, ann, carl (exact)
$ ./split_bin add-expense bob 3301 dave bob carl shares 1 1 2
Added expense 3: bob paid 33.01 for dave, bob, carl (shares)
$ ./split_bin pay bob ann 500
Added payment 4: bob paid ann 5.00
$ ./split_bin balances
ann 50.00
bob 4.75
carl -11.50
dave -43.25
$ ./split_bin settle
dave -> ann 43.25
carl -> ann 6.75
carl -> bob 4.75
Settled with 3 transfers.
Replay proof: all balances are zero.
$ ./split_bin ledger
1 | expense | ann paid 100.00 | equal | ann owes 25.00; bob owes 25.00; carl owes 25.00; dave owes 25.00
2 | expense | carl paid 60.00 | exact | dave owes 10.00; ann owes 20.00; carl owes 30.00
3 | expense | bob paid 33.01 | shares | dave owes 8.25; bob owes 8.26; carl owes 16.50
4 | payment | bob paid ann 5.00
```
