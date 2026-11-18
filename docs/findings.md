# Findings

Defects I found in the Toolshop API (`https://api.practicesoftwaretesting.com`, OpenAPI v5.0.0)
while building this collection, checked on 18 November 2026.

Each one has a request in the folder **06 Known issues** with a skipped test named
`Known issue TS-NN`, so it shows in every report without failing the run. If one gets fixed,
un-skip the test and it should pass.

The Toolshop is a practice application and may contain some of these on purpose. I've written
them up the way I would for a production API.

| ID | Finding | Severity |
|---|---|---|
| TS-01 | Brands can be created without logging in | Critical |
| TS-02 | Products can be edited without logging in | Critical |
| TS-05 | A customer can mark their own order as completed | Major |
| TS-03 | Checkout rejects real addresses and only accepts the city its own postcode lookup invents | Major |
| TS-04 | The same cart can be turned into an order more than once | Major |
| TS-07 | Registration accepts an email address without an @ | Minor |
| TS-06 | Payment check approves an unknown payment method | Minor |

---

### TS-01: Brands can be created without logging in

```bash
curl -s -X POST https://api.practicesoftwaretesting.com/brands \
  -H "Content-Type: application/json" -d '{"name":"test-brand","slug":"test-brand"}'
```

Expected: 401. Deleting a brand needs a token, so creating one should too.
Actual: `201` and the brand is live in the catalogue.

The OpenAPI file doesn't mark this endpoint as secured, so this is a gap in the design rather than
just the implementation. The spec leaves the create and edit endpoints for categories and products
unsecured in the same way (I tested products, see TS-02). The collection deletes the brand it
creates straight away, using the admin account.

### TS-02: Products can be edited without logging in

`PATCH /products/{id}` with no `Authorization` header returns `{"success": true}`. Anyone could
change a product's name or price. The collection only ever sends the product's existing name,
so the shop's data isn't actually changed.

### TS-05: A customer can mark their own order as completed

After placing an order, the same customer token can call:

```
PUT /invoices/{id}/status   {"status": "COMPLETED"}
```

Expected: 403, because order status belongs to the shop's staff. Actual: `200 {"success": true}`,
and the invoice search then shows the order as `COMPLETED`.

### TS-03: Checkout rejects real addresses

Placing an order with a real address, for example Marienplatz 1, 80331 Munich, Germany, fails:

```json
{"billing_country": ["The billing_country does not match the entered address. The city does not belong to the selected country."]}
```

I tried `DE` and `Germany`, Berlin, Vienna, Amsterdam and New York, and all of them were refused.
The order only goes through with the address returned by `GET /postcode-lookup`, and that lookup
returns invented data. For `80331` (central Munich) it answers "Stutensee, Saarland", and for
`1122AB` it answers "Scherpenzeel, Groningen" while the real town is Diemen. So the server checks
the city against its own fake lookup. A real customer couldn't type their address and check out.

The collection places its order with the looked-up address so the rest of the flow can be tested.

### TS-04: The same cart can be ordered twice

`POST /invoices` with a `cart_id` that has already been turned into an invoice returns `201` again
and creates a second invoice with a new number. The cart also keeps its items after checkout.
In a real shop that's a duplicate order and a double charge.

### TS-07: Registration accepts an invalid email

`POST /users/register` with `"email": "not-an-email"` returns `201`. The account exists and
can't receive order confirmations or password resets. The collection deletes the account again.

### TS-06: Payment check approves an unknown method

```bash
curl -s -X POST https://api.practicesoftwaretesting.com/payment/check \
  -H "Content-Type: application/json" -d '{"payment_method":"bitcoin","payment_details":{}}'
```

Actual: `200 {"message":"Payment was successful"}`. `POST /invoices` does reject `bitcoin` as a
payment method, so the two endpoints disagree on what's valid.

---

## Things that worked well

Worth noting too, because they're the checks that usually fail:

- Wrong password and unknown email get exactly the same `401`, so you can't find out who has an account.
- No endpoint returns a password, including `/users/me` and the admin user list.
- A customer can't list users, read reports, edit someone else's profile or open someone else's invoice.
- Refreshing a token invalidates the old one, and logging out kills the token.
- Cart quantities are limited to 1–99 and unknown products are rejected.
- Invoice totals match the cart exactly.

## Housekeeping note

An account that has placed an order can't be deleted (`409 "Seems like this customer is used
elsewhere."`). So the collection places orders with the public demo customer, and only creates
throwaway accounts for flows that can be cleaned up afterwards. One test account from my first
exploration (with two orders) is still on the practice site for that reason.
