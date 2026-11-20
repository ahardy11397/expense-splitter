# PLAN — expense-splitter: a Memelang graduate-level demo

Build a shared-expense splitter CLI entirely in Memelang (a "Splitwise for
a house"). Beta demonstration program: graph algorithms (debt
simplification via min-cash-flow) plus ledger bookkeeping.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang; ALL money in integer cents; floats only via
  `format <x> with 2 decimals` at print time.
- JSON persistence in `data/` via Optional record readers/writers; missing/
  corrupt file = clean Optional handling, never a traceback.
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Domain and CLI (thin argv-router main + logic modules; unknown command →
usage + exit 1; every error = one stderr-style line + exit 1)

    split add-person <name>
    split add-expense <payer> <total-cents> <name1> <name2> ... [split-mode]
              # split-mode: equal (default) | exact <c1> <c2> ... (must sum
              # to total) | shares <w1> <w2> ... (integer weights)
    split pay <from> <to> <cents>      # a settlement payment
    split balances                     # net balance per person, sorted by name
    split settle                       # MINIMUM number of transfers that clear
                                       # all debts (debt simplification)
    split ledger                       # full expense + payment history, sorted
                                       # by entry id

Splitting rules: equal split with a non-divisible remainder — distribute
the leftover cents one each to participants in name-sorted order until the
total is exact; exact mode must sum exactly (else error); shares mode
proportional with the same name-sorted remainder distribution.
`settle` is the grad-algorithm core: compute net balances, then produce the
minimum set of transfers (greedy min-cash-flow: repeatedly settle the
largest creditor against the largest debtor — deterministic tie-break by
name). A settle result must always clear every balance to zero when replayed.

## Tests — tests/meme/*.meme, `meme test`

At minimum: equal split exact division; equal split with remainder
(3 people, 100 cents → 34/33/33 assigned by name order — assert who pays
what); exact mode + rejection when sums mismatch; shares mode proportional
math; multi-expense netting (A owes B net after both directions); settle
produces minimum transfer count on a known 4-person scenario (hand-computed
assert on both the count and the amounts); replaying settle transfers zeroes
all balances; unknown person; negative/zero expense rejection; corrupt data
tolerance. Fixed data, unique scratch files, cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o split_bin` — end-to-end demo: 4 people, 3
   expenses (one per mode), balances, settle, replay proof. Transcript in
   your final summary.
4. README.md: CLI usage, split rules, settle algorithm, data files, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.
