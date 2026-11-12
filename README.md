# NoticeClock

A GenLayer Intelligent Contract that answers one question about a public contract
page: by which date do I have to give notice so I am out before my term ends, and
which sentence says so?

Deployed on Studionet: `0xb6119F10f7De149a74A5f680b85680d637751f45`

## The problem

Most business subscriptions renew on their own. The way out is a notice period
buried in a public page of legal text, written differently at every vendor. Miss
the window and another full term is owed. The clause is public, so the answer is
public, but nothing turns it into a date.

## What the contract does

1. `open_record(url, anchor)` pins a terms page and the day the current term ends.
2. `resolve(id)` reads that page inside consensus, finds the clause that lets a
   party leave, extracts the stated period, and stores the last day for notice
   together with the sentence it relied on.
3. `is_notice_open(id, today)` and `days_left(id, today)` let other contracts and
   people act on the stored date.

## The design line that matters

The model is asked for a number and the sentence that prints it. It is never asked
for a date and never computes one. The date comes from integer arithmetic in the
contract, so every validator arrives at the same day.

A record reaches `RESOLVED` only when a validator has independently done four
things against its own fetch of the page:

1. Fetched the page itself.
2. Confirmed the stored sentence is really on that page, compared after collapsing
   whitespace.
3. Confirmed the sentence both talks about ending the agreement and prints the
   number the leader stored. A real sentence about arbitration, or a sentence that
   prints 10 while the leader stored 30, is rejected.
4. Re ran the extraction and the arithmetic and required an identical verdict:
   same state, same deadline, same unit, same number, and for a failure, the same
   reason.

Any mismatch is a no vote, which becomes disagreement instead of a wrong deadline
settling on chain. A clause that cannot be proven from the page produces a stored
failure, never a guessed date.

## Reading a page that is too long for a prompt

Real terms pages rendered to 47 KB and up. Sending the opening chunk gives the
model cookie banners. Sending the whole page makes it skim. So the contract slides
a 4000 character window, scores each position with termination and renewal words
weighted above the bare word "notice", and sends that contiguous stretch. The
quote is then verified against the whole page, not the window, so a clause just
outside it is not punished.

## State

| Field | Meaning |
| --- | --- |
| `owner` | address that opened the record, for routing only |
| `terms_url` | the page, https only, 300 characters max |
| `anchor_iso` | the day the current term ends |
| `state` | `OPEN`, `RESOLVED`, `UNRESOLVED` |
| `reason` | why a record is unresolved, empty when resolved |
| `notice_days` | the integer printed in the clause |
| `unit` | `CALENDAR_DAYS`, `BUSINESS_DAYS`, `MONTHS` |
| `applies_to` | `CUSTOMER`, `PROVIDER`, `BOTH`, `UNKNOWN` |
| `quote` | the verbatim sentence, 12 to 400 characters |
| `deadline_iso` | the last day to give notice, empty when unresolved |
| `checked_at` | block time of the read that produced these values |
| `version` | how many times the record has been resolved |

Failure reasons, all stored, all fail closed: `NO_NOTICE_CLAUSE`,
`UNIT_NOT_READABLE`, `QUOTE_NOT_GROUNDED`, `QUOTE_NOT_SUPPORTING`,
`EXTRACTION_FAILED`, `DEADLINE_UNCOMPUTABLE`, and `SOURCE_EMPTY_OR_UNREADABLE`
which reverts the write instead of storing a guess.

Input errors revert before any network use: `URL_MUST_BE_HTTPS`, `URL_TOO_LONG`,
`ANCHOR_DATE_INVALID`, `TODAY_DATE_INVALID`, `UNKNOWN_RECORD`, `CAPACITY_REACHED`.

## Determinism rules

Published on chain by `rules()`:

* the anchor day itself is not counted
* `CALENDAR_DAYS` counts every day
* `BUSINESS_DAYS` counts Monday to Friday, no holiday calendars
* `MONTHS` subtracts whole months and clamps to the last real day of the month
* bounds: 1 to 730 days, 1 to 60 months
* invalid calendar dates such as `2026-02-30` are rejected, not rolled over

## API

```python
open_record(terms_url: str, anchor_iso: str) -> str   # returns the record id
resolve(record_id: str) -> str                        # returns the new state

get_record(record_id) -> dict
deadline(record_id) -> str
is_notice_open(record_id, today_iso) -> bool
days_left(record_id, today_iso) -> int
list_records() -> list
count() -> int
rules() -> dict
```

## Tests

25 direct mode tests, all passing. `genvm-lint check` clean, 9 methods (2 write,
7 view).

Covered: the date engine including business days, month end clamping and a leap
day; a quote that exists but prints the wrong number; a quote that is not on the
page; a quote that is on the page but about arbitration; a prompt injection
planted in the page; a page with no clause; an unreadable page; a validator
rejecting a borrowed quote; a validator rejecting a different day; a validator
whose own reading differs; idempotent opening; input validation; version bumps;
the notice window either side of the deadline; and a clause buried past 20000
characters of notice heavy text, where the mocked model only answers if the clause
actually reached the prompt.

## Running the tests

```
pip install -r requirements.txt
pytest tests/direct -q
genvm-lint check contracts/noticeclock.py
```

## Live proof on Studionet

| Case | Transaction | Result |
| --- | --- | --- |
| deploy | `0x4b7292d0ffa01200e339f9377e5a16f5c3bf1394fcce6215e9039fec4902477a` | FINALIZED, MAJORITY_AGREE |
| open Hetzner terms, term ends 2027-03-31 | `0xec1758f285b83840e084d4909c4e85c9562265fa2994e85161273e91371096f2` | record N1, MAJORITY_AGREE |
| resolve that record | `0x4565b278c106ae6014dd7f80ff0189af771b36f41ece0e5ec9e1eaca247a8769` | FINALIZED, MAJORITY_AGREE, RESOLVED |
| resolve a page with no clause | `0x9ab3669456a195614d0224234c11d2d3b17da6240143b2feda8e7fbf65939ed7` | UNRESOLVED, `NO_NOTICE_CLAUSE` |
| open with a plain `http://` URL | `0x24d00e1a71304c238796165dbb83d4e9628fae664214fa1c49b18d828b8e13b9` | FINALIZED, rollback, `URL_MUST_BE_HTTPS` |

The stored record:

```json
{
  "state": "RESOLVED",
  "notice_days": 30,
  "unit": "CALENDAR_DAYS",
  "applies_to": "BOTH",
  "anchor_iso": "2027-03-31",
  "deadline_iso": "2027-03-01",
  "quote": "The contract may be terminated by either party with 30 days' notice to the end of the month, without specifying any reasons."
}
```

Views: `is_notice_open(id, "2026-12-01")` is `true`, `is_notice_open(id,
"2027-03-02")` is `false`, `days_left(id, "2026-12-01")` is `90`.

## Known limits

* The contract reads the general clause. If the page then says other periods may
  apply by product, the returned figure is the general one.
* Business days ignore public holidays.
* Pages behind a JavaScript app or a bot wall revert as unreadable rather than
  producing a number.
* The prompt is English, so a non English sentence is more likely to end as
  `QUOTE_NOT_SUPPORTING`.
* A record is a claim about the page at `checked_at`. Re resolve before acting on
  an old one.

## Layout

```
contracts/noticeclock.py     the contract
tests/direct/                25 direct mode tests
gltest.config.yaml           test runner config
requirements.txt             pinned toolchain
```
