# SplitIt

A Venmo-style payment app with a buy-now-pay-later option, written in Go.

You can send money to a friend, request money back, split a bill across a group, or take a purchase and break it into installments you pay off over a few weeks. Everything runs in one page in the browser.

This is a personal project and it was my first time writing Go, so a big part of the point was figuring out how to lay out a backend myself instead of having a framework decide for me. There's no real money anywhere in it. Balances are just integers in a SQLite file, and "depositing" is a button that adds to that number.

## What you can do in it

Send a payment to another user, with a note if you want. Request money from someone, and they get to accept or decline it. Add friends, make a group, and split a payment evenly across everyone in it. Or use the BNPL flow: pick a purchase amount and a number of installments, and the app builds you a payment schedule with due dates two weeks apart.

Every account has a credit score from 0 to 100 that starts at 50. Paying an installment on time earns you a point, paying it late costs you five, and letting it sit past due costs you ten. Your score then decides the fee on your next loan: 1% if you're above 90, up to 7% if you're below 50. Every change gets written to a log table so you can see the whole history.

There's also a wallet view with monthly spending, who you pay the most, and how much of your credit limit you're using, plus in-app notifications and a combined activity feed.

## Running it

You need Go 1.25 or newer. Nothing else to install.

```bash
git clone https://github.com/connorpodea/splitit
cd splitit
go run ./cmd/server
```

Then open http://localhost:8080. The SQLite file (`app.db`) gets created on first run with all the tables, so there's no migration step. Tests:

```bash
go test ./...
```

There are 86 of them, mostly around money movement and making sure users can't touch each other's stuff.

## How it's laid out

Three layers, and I tried to keep them from leaking into each other. `store` is the only package that touches SQL, `handlers` is the only package that touches HTTP, and `models` is just the shared structs.

```
cmd/server/main.go     starts the DB, the scheduler, and the HTTP server
internal/models/       all the data types
internal/store/        every SQL query, one file per area (payments, installments,
                       groups, social, wallet, sessions, settings, users)
internal/handlers/     the 68 HTTP endpoints, same file split as store,
                       plus middleware and the HTML rendering
```

The frontend is HTMX and Tailwind loaded from a CDN, so there's no build step and no JavaScript bundle. Handlers return HTML fragments and HTMX swaps them into the page. The page shells are Go string literals, and anything containing user-entered text goes through `html/template` so it gets escaped properly.

## Database

14 tables:

`users`, `sessions`, `payments`, `payment_requests`, `installments`, `friends`, `friend_requests`, `groups`, `group_members`, `group_invitations`, `notifications`, `user_settings`, `wallet_transactions`, `credit_score_log`

Foreign keys are turned on at the connection level, and there's an index on every foreign key column so lookups don't end up scanning whole tables. All money is stored as integer cents, never floats, so nothing drifts from rounding.

Anything that moves money runs inside a single transaction. Creating a BNPL loan, for example, does five writes: the loan record, the credit limit deduction, funding the seller, collecting the down payment, and generating the schedule rows. If any one of those fails the whole thing rolls back, because a loan that pays out the seller but never records a payment schedule would just be lost money.

## Auth and security

Passwords are hashed with bcrypt before they're stored. Logging in creates a random 32-byte token, saves it in the `sessions` table, and sends it back as an HttpOnly cookie, so JavaScript on the page can't read it. Tokens expire after 30 days and get deleted the next time they're used. Deactivating an account wipes all of its sessions so old cookies stop working.

Handlers never trust a user ID sent by the client. Whoever you are comes from the session lookup, which means you can't pay off someone else's installment by editing a form field. There are 17 tests specifically for this, each one an authenticated user trying to act on a third party's records and getting rejected.

There's also CSRF protection using the double-submit cookie pattern, and a token bucket rate limiter on login and registration that caps you at 5 attempts a minute per IP.

## The background job

A goroutine on a 24-hour ticker looks for unpaid installments past their due date and docks the borrower's credit score. Each user's penalty runs in its own transaction, and once an installment has been penalized it gets flagged so the next day's pass doesn't hit it again.

## Stuff I know is rough

- SQLite is capped at one open connection here, since it only allows one writer at a time and more connections just produced "database is locked" errors. That's fine for one person clicking around, but it means writes are serialized and it wouldn't hold up under real traffic.
- The HTML got long. `dashboard_views.go` is around 1,900 lines of Go strings building markup, which works but is not how I'd do it again.
- There's an email notifications toggle in settings that saves your preference but doesn't send anything, because there's no mail setup.
- It only runs locally. Nothing is deployed, and the treasury account that funds BNPL purchases is just another row in the users table.

Built by Connor Podea to learn Go, relational database design, and how web auth actually works.
