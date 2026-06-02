# Supabase Support Engineer Casebook

I built this casebook for my Supabase Support Engineer application.

Instead of only saying that I am interested in Supabase, I wanted to work through a few problems that felt close to real support cases: a signed-in user seeing no rows, a Realtime channel that connects but receives no events, and an RLS policy that is correct but worth checking from a query-planning perspective.

I wanted to show how I would handle a support issue when the first symptom is ambiguous: reproduce it, check each layer, explain what the evidence points to, and be clear about what I would escalate.

## How to review this repo

If you only have a few minutes, start with:

1. [Case 1: Auth + RLS authenticated user sees no rows](cases/01-auth-rls-empty-results.md)
2. [Case 2: Realtime — subscription connects but receives no events](./cases/02-realtime-subscribed-no-events.md)
3. [Case 3: Postgres/RLS performance — slow queries after enabling policies](./cases/03-rls-performance-indexing.md)

Each case is written to show both technical debugging and customer communication. The most relevant sections are:

- reproduction notes
- likely root cause
- fix or workaround
- support reply I would send to the user
- escalation note

## Cases

### 1. Auth + RLS: authenticated user sees no rows

**Area:** Supabase Auth, PostgreSQL Row Level Security  
**Status:** Reproduced in a hosted Supabase project

A user is logged in, but their query returns:

```js
data = []
error = null
```

The important detail is that this is not always a failed query. In my reproduction, the request succeeded, but RLS filtered out every row because no matching `SELECT` policy allowed the authenticated user to read the data.

This case helped me separate two layers that are easy to mix up in Supabase support:

- **Authentication:** is the user signed in?
- **Authorization:** is this user allowed to read these rows?

The key detail in this reproduction was that the request did not fail. The client received `error = null`, but RLS still removed all rows from the result.

[Read case →](cases/01-auth-rls-empty-results.md)

### 2. Realtime: subscription connects but receives no events

**Area:** Supabase Realtime, Postgres Changes  
**Status:** Reproduced in a hosted Supabase project

A subscription reached `SUBSCRIBED`, and the insert succeeded, but no payload was received until the table was enabled for Postgres Changes / Realtime.

The misleading part is that `SUBSCRIBED` can look like the whole Realtime setup is working. In this case, it only confirmed that the client joined the channel. It did not prove that a matching database change would be delivered.

This case was useful because it forced me to debug across a few layers:

- client subscription status
- schema and table names
- event filters
- database changes happening after subscription
- Realtime table configuration
- RLS visibility

My first instinct was to treat `SUBSCRIBED` as proof that Realtime was working, but the reproduction showed that it only proved the channel connection.

[Read case →](./cases/02-realtime-subscribed-no-events.md)

### 3. Postgres/RLS performance: slow queries after enabling policies

**Area:** PostgreSQL, RLS, indexing, `EXPLAIN ANALYZE`  
**Status:** Reproduced in a hosted Supabase project

I tested a 100,000-row table with an RLS-style ownership column.

Before adding an index, Postgres used a sequential scan and filtered out 99,000 rows. After adding an index on `user_id`, Postgres used the index. The runtime was similar in this small hosted test, which was actually an important result: measuring is better than assuming.

This case is about the difference between a policy being correct and a policy being efficient as data grows. If an RLS policy depends on a column like `user_id`, that column is worth looking at when debugging slow queries.

I expected the index to make the query obviously faster, but the small hosted test was more useful than that: it changed the plan, while reminding me not to claim performance improvements without measuring them.

[Read case →](./cases/03-rls-performance-indexing.md)

## Case format

Each case includes:

- customer problem
- impact
- investigation steps
- reproduction notes
- likely root cause
- fix or workaround
- support reply I would send to the user
- escalation note
- documentation or product improvement idea
- what I learned while reproducing and debugging the issue

## What I wanted to show

This casebook is meant to show how I would approach support work at Supabase:

- reproduce issues instead of guessing
- separate client, auth, database, policy, and configuration problems
- explain technical behavior in a way that is useful to the user
- avoid overpromising when the evidence is not there yet
- use logs, minimal examples, SQL, and query plans to narrow down the issue
- know when a case can be solved through support and when it needs escalation
