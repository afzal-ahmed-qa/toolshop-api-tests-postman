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

The folders run in order and pass IDs and tokens along, so each one reads like a real session:

1. **Catalogue (no login).** Paging, product detail with a JSON schema check, related products,
   search, sorting by price, filtering by brand and price range, and the category tree.
2. **Login and tokens.** Logging in with the public demo accounts, wrong password versus unknown
   email (they must look identical), missing and forged tokens, and token refresh.
3. **Account lifecycle.** A new customer registers, hits the duplicate-email, weak-password and
   under-18 rules, edits their profile, tries to edit someone else's, changes their password, logs
   out, and is deleted again by the admin.
4. **Cart and checkout.** Cart create, add, quantity limits (1 to 99), unknown products, removing an
   item, postcode lookup, payment check, placing the order and checking the invoice total line by line.
5. **Access control.** Another customer can't see the invoice, customers can't list users or read
   sales reports, the admin can. Favourites, including the duplicate rule.
6. **Known issues.** Seven defects I found, each kept as a skipped test (see below).
7. **Search terms (data-driven).** One request repeated for every row in `data/search-terms.csv`.

Every request also gets two collection-level checks: response time under 3 seconds and a JSON
content type.

## What I found

The full write-up with steps to reproduce is in [docs/findings.md](docs/findings.md). The ones I'd
raise first:

- **Brands can be created and products edited without logging in** (TS-01, TS-02). Deleting needs
  a token, creating and editing don't.
- **A customer can mark their own order as completed** (TS-05).
- **Checkout rejects real addresses.** The server only accepts the city its own postcode lookup
  returns, and that lookup invents cities: 80331, central Munich, comes back as "Stutensee" (TS-03).
- **The same cart can be ordered twice** (TS-04).

I didn't change the assertions to match what the API does. The correct expectation stays in the
collection as a skipped test called `Known issue TS-NN`, so the run stays green for everything
else and the defects stay visible in every report.

A lot also works well, and the tests prove it: identical errors for wrong password and unknown
email, no passwords in any response, customers properly locked out of other people's data and
admin endpoints, token refresh invalidating the old token, and invoice totals that match the cart.

## Things I had to think about

**Shared data.** Everyone practising on the Toolshop uses the same server, and any account that has
placed an order can't be deleted afterwards. So orders are placed with the public demo customer,
and the only new accounts are ones the run deletes again. Anything the API wrongly accepts (a
brand, an invalid account) is removed by the admin in the same request's test script.

**Network hiccups.** The practice server occasionally drops a connection or stalls. Each request has
a 20 second timeout and there's a short pause between requests, which keeps the run stable and is
kinder to a server other people are using.

**Short-lived tokens.** Tokens expire after five minutes. The admin logs in at the start of the
account folder and the clean-up steps later in the run reuse that token, which works because a
full run finishes in well under five minutes. Running folder 05 on its own logs the admin in first.

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
