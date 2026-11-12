# NoticeClock

**Category:** Intelligent Contracts
**Network:** GenLayer Studionet

## The problem

Business subscriptions renew by default, and the only way out is a notice period
buried in a public page of legal text. Thirty days here, two months there, counted
in business days somewhere else. Miss the window and another full term is owed.
The clause is public, so the answer is public, but nobody turns it into a date.

## What it does

You pin a terms URL and the day your current term ends. The contract reads the
page inside consensus, finds the clause that lets a party leave, extracts the
stated period, and stores the last day to give notice together with the exact
sentence it relied on. A caller can then ask `is_notice_open(id, today)` and
`days_left(id, today)` and act on the result.

A live run against the Hetzner terms, term ending 2027-03-31:

```
state         RESOLVED
notice_days   30
unit          CALENDAR_DAYS
applies_to    BOTH
deadline      2027-03-01
quote         "The contract may be terminated by either party with 30 days'
              notice to the end of the month, without specifying any reasons."
```

## Why the consensus layer is load bearing, not decoration

* The model is asked for a number and the sentence that prints it. It is never
  asked for a date and never computes one. The deadline comes from integer civil
  date arithmetic inside the contract, so every validator arrives at the same day.
* The model has no way to be vague and slip through. A missing clause must come
  back as `notice_days: 0`, which the contract treats as a failure with a stored
  reason, not as an answer.
* Every validator re fetches the page itself and runs three checks on the leader's
  claim: the sentence must exist on the validator's own copy, it must contain the
  number the leader stored, and it must talk about ending the agreement. A real
  sentence about arbitration fails. A sentence that prints 10 while the leader
  stored 30 fails. A sentence the leader invented fails.
* The validator then re runs the extraction and the arithmetic and requires an
  identical verdict: same state, same deadline, same unit, same number, and for a
  failure, the same reason. Any difference is a no vote, which becomes
  disagreement instead of a wrong deadline settling on chain.
* Untrusted page text is passed inside `<source>` tags with an explicit
  instruction that it is data, never instructions. A test plants an injection in
  the page and asserts it is not obeyed.

## Reading pages that are too long for a prompt

Real terms pages rendered to 47 KB and up. Sending the opening chunk gives the
model cookie banners. Sending everything makes it skim. So the contract slides a
4000 character window, scores each position with termination and renewal words
weighted above the bare word "notice", and sends that contiguous stretch. The
quote is then verified against the whole page, not the window, so a clause just
outside it is not punished.

## State and failure modes

One record per `url|anchor`, ids `N1`, `N2`, cap 2000, opening twice returns the
same id. States are `OPEN`, `RESOLVED`, `UNRESOLVED`. Every failure is stored with
a reason a reader can act on: `NO_NOTICE_CLAUSE`, `UNIT_NOT_READABLE`,
`QUOTE_NOT_GROUNDED`, `QUOTE_NOT_SUPPORTING`, `EXTRACTION_FAILED`,
`DEADLINE_UNCOMPUTABLE`, plus `SOURCE_EMPTY_OR_UNREADABLE` which reverts rather
than storing a guess. Input errors revert before any network use:
`URL_MUST_BE_HTTPS`, `URL_TOO_LONG`, `ANCHOR_DATE_INVALID`, `TODAY_DATE_INVALID`,
`UNKNOWN_RECORD`, `CAPACITY_REACHED`.

The determinism rules, including "the anchor day is not counted", "business days
are Monday to Friday with no holidays" and "month subtraction clamps to a real
day", are published on chain by `rules()`.

`resolve` is open to any caller: the fact is public and pages change, so anyone may
refresh, and each run bumps `version`. A caller can trigger a read but cannot pick
the outcome.

## Tests

25 direct mode tests, all passing, `genvm-lint check` clean, 9 methods (2 write,
7 view). They cover the date engine including a leap day and month end clamping, a
borrowed quote, a quote that prints the wrong number, a quote that exists but is
off topic, a prompt injection in the page, a page with no clause, a validator that
disagrees with the leader, input validation, idempotent opening, the notice window
either side of the deadline, and a clause buried past 20000 characters of notice
heavy text where the mocked model only answers if the clause actually reached the
prompt.

## Proof on chain

```
contract      0xe9C1212D7299f1E57610A3F1384ED89a0dc1615A
deploy tx     0xd51be71cf1d6a4ace8ea0057a69a0106ede84faa87643f55e32c67cb3ed3cf5f
resolve tx    0xc08a2d37f99942dc6198e297ab502028bd5207b65187151f244df92b5c0f5fab  FINALIZED, MAJORITY_AGREE
fail closed   0x390b5695336a78774167f54d6e2e558c396bcde0f2889d5b6cd79180f6f25e04  UNRESOLVED, NO_NOTICE_CLAUSE
input guard   0x0fdf526cfffd150927c5bd6f4d7c0f31d94f57f1ae204311d222612ea4c1ce8e  rollback, URL_MUST_BE_HTTPS
```

The resolved record stores `notice_days 30`, `unit CALENDAR_DAYS`, `applies_to
BOTH`, `deadline_iso 2027-03-01`, and the sentence from the page. Views on it:
`is_notice_open(id, "2026-12-01")` is `true`, `is_notice_open(id, "2027-03-02")`
is `false`, `days_left(id, "2026-12-01")` is `90`.

## Limits, stated honestly

The contract reads the general clause. If the page then says other periods may
apply by product, the reader should check the plan terms. Business days ignore
public holidays. Pages behind a JavaScript app or a bot wall revert as unreadable
instead of producing a number. A record is a claim about the page at `checked_at`,
so re resolve before acting on an old one.
