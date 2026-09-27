# Submission Notes

Live application:

- Vercel storefront: https://sanctum-sanctorum-gamma.vercel.app
- Render API: https://sanctum-sanctorum-9x63.onrender.com
- Public repository: https://github.com/Anchal-T/sanctum-sanctorum

The Vercel storefront proxies API requests to Render, so both URLs use the same
application data. The seeded database includes member `#1` (Wong Li, supreme
tier), which can be used in the UI. The Render free service may take a little
time to wake after inactivity.

## Completed

- Books: ISBN-13 normalization and checksum validation, duplicate detection,
  partial updates, filtering, sorting, and pagination.
- Members: normalized email validation, duplicate detection, tier access rules.
- Orders: validation, price snapshots, tier and bulk discounts, all-or-nothing
  stock reservation, payment, cancellation, and stock restoration.
- Loans: due dates, tier limits, restricted-book access, overdue checks,
  returns, late fees, computed status, and status filtering.
- Member statistics and top-books reporting.
- The full test suite passes: `202 passed`.
- The API and frontend are deployed publicly.

## Not completed

Optional assignment extras were not implemented:

- Concurrent last-copy protection for simultaneous orders.
- A paginated `GET /members` endpoint.

The deployed Render instance currently uses SQLite. This keeps the local setup
and tests simple, but Render's filesystem is ephemeral, so deployment data can
reset after a restart or redeploy. A hosted Postgres database would be the
next production-hardening step.

## Known issues found during browser verification

The Loans page renders a Return button, but clicking it does nothing.

Reproduction:

1. Sign in as member `#1`.
2. Borrow an available book.
3. Open the Loans tab.
4. Click Return.

Expected behavior: the loan is returned, the book's stock is restored, and the
loan changes to `returned`.

Actual behavior: the Return button remains visible, the loan stays active, and
no request is sent.

Root cause: `frontend/app.js` renders `data-action="loan-return"` for the
button, but the `initActions()` dispatcher has no `loan-return` case. The
backend `POST /loans/{id}/return` endpoint works when called directly.

Status: known frontend issue, not fixed in this submission.

## Architecture and trade-offs

- Routers remain thin and handle HTTP parsing and dependency injection.
  Business rules live in the service modules.
- Pydantic schemas handle request validation and normalization before service
  logic runs.
- Order stock is reserved at creation. Every book and stock check completes
  before any stock is changed, so a failed order does not partially mutate
  inventory.
- Order items store the unit price at order time, so later catalogue price
  changes do not alter existing orders.
- Loan status is computed at read time from the current clock. Exactly at the
  due time a loan is still active; late fees use the price at return time.
- The application uses the injected `get_now` dependency so time-based rules
  remain deterministic in tests.
- SQLite is retained for the local default and test suite. The Vercel frontend
  is separated from the FastAPI service because Render is a better fit for a
  long-running Python API.

## Spec decisions

- Mixed-case ordering follows the database's normal string ordering because
  `SPEC.md` explicitly leaves that behavior unspecified.
- Ambiguous behavior was documented and implemented according to the written
  check order in the specification.

## AI usage

Cursor was used as an engineering assistant, not as a replacement for
understanding the code:

- I used it first to map the repository, read the assignment contract, inspect
  the existing routers/services/models, and identify the incomplete behavior.
  The specification and tests remained the source of truth.
- I deliberately worked in small feature slices: books, members, loans,
  reports/stats, and orders. After each slice I ran its focused tests rather
  than waiting until the end to discover unrelated failures.
- I reviewed the implementation decisions before accepting them. In
  particular, I checked the order validation sequence, the all-or-nothing
  stock behavior, price snapshots, loan boundary conditions, late-fee
  rounding, and the use of the injected `get_now` dependency.
- I kept the tests unchanged and used failing responses to locate missing
  application behavior. For example, report tests initially failed because
  order creation was still a 501; I treated that as a dependency issue and
  completed the order flow before judging the report query.
- I used AI-assisted deployment setup, but still verified the public Vercel
  and Render URLs directly through `/`, `/health`, and `/books`. I also
  checked the repository status and tracked-file list for secrets and
  generated files.
- During browser verification, I used the browser tooling to exercise the
  deployed UI rather than assuming that passing backend tests proved the
  frontend worked. This exposed the loan Return issue. I reproduced it by
  clicking the button, confirmed that the loan remained active, then compared
  the rendered `loan-return` action with the `initActions()` dispatcher and
  documented the mismatch instead of claiming the flow was complete.
- I made the final scope decisions myself. Optional concurrency protection and
  the paginated members endpoint were left out intentionally and documented
  rather than adding untested complexity.

One generated checksum implementation was too compressed to be easy to review.
I replaced it with an explicit loop and named intermediate values for the ISBN
weights, remainder, expected digit, and actual digit. I also narrowed an early
broad implementation plan to the requested feature slices instead of allowing
the assistant to expand the scope.

The final verification was `uv run pytest`, which passed all 202 tests.