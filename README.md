# Supabase Support Engineer Casebook

I built this casebook as proof-of-work for the Supabase Support Engineer role. I did not want to just say that I am interested in Supabase. I wanted to show it through reproducible support cases.

My background is in technical support, customer communication, internal tools, and API integrations. This repo connects that experience with Supabase debugging: PostgreSQL, Auth, RLS, Realtime, indexes, customer-facing explanations, and escalation notes.

The goal was not to create perfect textbook examples, but to show how I approach unclear support issues: reproduce the problem, isolate the layer, explain the likely cause, suggest a safe fix, and know when to escalate.

## How to review this repo

If you only have a few minutes, start with:

1. [Case 1: Auth + RLS — authenticated user sees no rows](./cases/01-auth-rls-empty-data.md)
2. [Case 2: Realtime — subscription connects but receives no events](./cases/02-realtime-subscribed-no-events.md)
3. [Case 3: Postgres/RLS performance — slow queries after enabling policies](./cases/03-rls-performance-indexing.md)

Each case is written to show both technical debugging and customer communication. The most relevant sections are:

- the reproduction notes
- the likely root cause
- the fix or workaround
- the support reply I would send to the user
- the escalation note

## Cases

### 1. Auth + RLS: authenticated user sees no rows

**Area:** Supabase Auth, PostgreSQL Row Level Security  
**Status:** Reproduced in a hosted Supabase project

A user is logged in, but their query returns:

```ts
data = []
error = null
```

The important detail is that this is not always a failed query. In my reproduction, the request succeeded, but RLS filtered out every row because no matching select policy allowed the authenticated user to read the data.

This case helped me separate two things that are easy to mix up when debugging Supabase issues:

authentication: is the user signed in?
authorization: is this user allowed to see these rows?

[Read case](cases/01-auth-rls-empty-results.md)

---

### 2. Realtime: subscription connects but receives no events

**Area:** Supabase Realtime, Postgres Changes
**Status:** Reproduced in a hosted Supabase project

A subscription reached SUBSCRIBED, and the insert succeeded, but no payload was received until the table was enabled for Postgres Changes / Realtime.

The misleading part is that SUBSCRIBED can look like the whole Realtime setup is working. In this case, it only confirmed that the client joined the channel. It did not prove that a matching database change would be delivered.

This case was useful because it forced me to debug across a few layers:

- client subscription status
- schema and table names
- event filters
- database changes happening after subscription
- Realtime table configuration
- RLS visibility

[Read case](cases/02-realtime-no-events.md)

---

### 3. Postgres/RLS performance: slow queries after enabling policies

**Area:** PostgreSQL, RLS, indexing, `EXPLAIN ANALYZE`  
**Status:** Reproduced in a hosted Supabase project

I tested a 100,000-row table with an RLS-style ownership column.

Before adding an index, Postgres used a sequential scan and filtered out 99,000 rows. After adding an index on user_id, Postgres used the index. The runtime was similar in this small test, which was actually an important result: measuring is better than assuming.

This case is about the difference between a policy being correct and a policy being efficient as data grows. If an RLS policy depends on a column like user_id, that column is worth looking at when debugging slow queries.

[Read case](cases/03-postgres-rls-performance.md)

## Case format

Each case includes:

- customer problem
- impact
- investigation steps
- my reproduction notes
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
