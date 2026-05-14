# Supabase Support Engineer Casebook

I built this casebook while learning Supabase more deeply. I wanted to test myself against the kind of work the role requires, instead of just reading docs and sending over a CV.

My background is in technical support, customer communication, internal tools, and API integrations. This repo is my way of connecting that experience with Supabase, PostgreSQL, Auth, RLS, and Realtime.

## Cases

### 1. Auth + RLS: authenticated user sees no rows

**Area:** Supabase Auth, PostgreSQL Row Level Security  
**Status:** Reproduced in a hosted Supabase project

A user is logged in, but their query returns:

```ts
data = []
error = null

In my reproduction, the query was not failing. RLS was enabled, but no matching select policy allowed the authenticated user to read the row.

[Read case](cases/01-auth-rls-empty-results.md)

### 2. Realtime: subscription connects but receives no events

Area: Supabase Realtime, Postgres Changes
Status: Reproduced in a hosted Supabase project

A subscription reached SUBSCRIBED, and the insert succeeded, but no payload was received until the table was enabled for Postgres Changes / Realtime.
Takeaway: SUBSCRIBED means the client joined the channel. It does not always mean a matching database change will be delivered.

[Read case](cases/02-realtime-no-events.md)

### 3. Postgres/RLS performance: slow queries after enabling policies

Area: PostgreSQL, RLS, indexing, EXPLAIN ANALYZE
Status: Reproduced in a hosted Supabase project

I tested a 100,000-row table with an RLS-style ownership column.

Before adding an index, Postgres used a sequential scan and filtered out 99,000 rows. After adding an index on user_id, Postgres used the index. The runtime was similar in this small test, which was a useful reminder to measure instead of assuming.

[Read case](cases/03-postgres-rls-performance.md)

## Case format

Each case includes:

customer problem
impact
investigation steps
reproduction notes
likely root cause
fix or workaround
reply I would send to the customer
escalation note
documentation or product improvement idea
what I learned


### Notes

The examples use test projects, test users, and anonymized IDs. No private customer data or production credentials are included. I intentionally kept the cases small. The point is to show the debugging process clearly, not to build a large demo application.
