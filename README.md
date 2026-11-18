# Toolshop API tests (Postman + Newman)

[![API tests](https://github.com/afzal-ahmed-qa/toolshop-api-tests-postman/actions/workflows/api-tests.yml/badge.svg)](https://github.com/afzal-ahmed-qa/toolshop-api-tests-postman/actions/workflows/api-tests.yml)

API tests for the [Toolshop](https://practicesoftwaretesting.com), an online hardware shop built by
Testsmith for people practising software testing. Its REST API has a documented
[OpenAPI spec](https://api.practicesoftwaretesting.com/api/documentation), JWT login, customer and
admin roles, a cart and a real checkout, which makes it much closer to a production API than most
practice targets.

The collection runs with Newman in GitHub Actions on every push and once a day. Latest report:
https://afzal-ahmed-qa.github.io/toolshop-api-tests-postman/ (data-driven run
[here](https://afzal-ahmed-qa.github.io/toolshop-api-tests-postman/data-driven.html)).

## What it covers

The folders run in order and pass IDs and tokens from one request to the next, so a run follows
what a real customer would do.

| Folder | What it checks |
|---|---|
| 01 Catalogue | Paging, product detail against a JSON schema, related products, search, sorting by price, filtering by brand and price range, the category tree |
| 02 Login and tokens | Login with the demo accounts, wrong password versus unknown email, missing and forged tokens, token refresh |
| 03 Account lifecycle | Register, the duplicate-email, weak-password and under-18 rules, profile update, editing someone else's profile, password change, logout, deletion |
| 04 Cart and checkout | Adding and removing items, the 1 to 99 quantity limit, unknown products, postcode lookup, payment check, placing the order, invoice totals |
| 05 Access control | Another customer's invoice, the user list and sales reports for customers versus the admin, favourites |
| 06 Known issues | The seven defects described below, each as a skipped test |
| 07 Search terms | One request repeated for every row in `data/search-terms.csv` |

On top of that, every response is checked for a JSON content type and a response time under 3
seconds. A full run is 77 requests and about 260 assertions, and takes around a minute in CI.

## What I found

Seven defects, written up with steps to reproduce in [docs/findings.md](docs/findings.md). The
ones I'd raise first:

- Brands can be created and products edited without logging in (TS-01, TS-02), even though
  deleting them needs a token.
- A customer can mark their own order as completed (TS-05).
- Checkout rejects real addresses. The server only accepts the city returned by its own postcode
  lookup, and that lookup makes cities up: 80331, central Munich, comes back as "Stutensee" (TS-03).
- The same cart can be ordered twice (TS-04).

I kept the correct expectation for each of these as a skipped test named `Known issue TS-NN`
rather than changing the assertion to match the bug. The run stays green, and the defects still
show up in every report.

Plenty works as it should, too. Wrong password and unknown email get the same error, no response
contains a password, customers can't reach other people's data or the admin endpoints, refreshing
a token invalidates the old one, and invoice totals match the cart.

## Things I had to think about

Everyone practising on the Toolshop shares one server, and an account that has placed an order
can't be deleted afterwards. So orders go through the public demo customer, and the only accounts
the run creates are ones it deletes again at the end. If the API wrongly accepts something, like a
brand or an account with an invalid email, the admin removes it in the same request's test script.

The server sometimes drops a connection, mostly when I ran the collection from home rather than
from GitHub Actions. Each request has a 20 second timeout and there's a 300 ms pause between
requests, which keeps the run stable and goes easier on a server other people are using.

Tokens expire after five minutes. The admin logs in at the start of folder 03 and later clean-up
steps reuse that token, which is fine because a full run takes about a minute. If you run folder 05
on its own, it logs the admin in first.

## Running it

Import `collections/toolshop.postman_collection.json` and
`environments/toolshop.postman_environment.json` into Postman and use the Collection Runner, or run
it from the command line with Node.js 20 or newer:

```bash
npm ci
npm test            # everything except the data-driven folder, HTML report in reports/index.html
npm run test:data   # search terms from data/search-terms.csv
```

The environment contains the demo logins that the Toolshop itself publishes for testers
(`customer@practicesoftwaretesting.com` / `welcome01` and so on). They aren't personal credentials.

## Layout

```
collections/    the Postman collection
environments/   base URL and the public demo accounts
data/           CSV for the data-driven run
docs/           findings
.github/        Newman in GitHub Actions, report published to GitHub Pages
```

## Related

- [playwright-typescript-e2e-framework](https://github.com/afzal-ahmed-qa/playwright-typescript-e2e-framework):
  UI and API automation with Playwright and TypeScript.
- [petstore-api-tests-postman](https://github.com/afzal-ahmed-qa/petstore-api-tests-postman):
  the same approach on the Swagger Petstore.
- [manual-test-plans-ecommerce](https://github.com/afzal-ahmed-qa/manual-test-plans-ecommerce):
  test strategy, plans and defect reports for an e-commerce site and its API.

---

Afzal Ahmed, [github.com/afzal-ahmed-qa](https://github.com/afzal-ahmed-qa)
