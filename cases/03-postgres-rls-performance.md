# Case 3: Slow queries caused by RLS policy checks on non-indexed columns

> Status: Draft. I wrote this based on a common Supabase/Postgres performance scenario and will update it after reproducing the issue with real query plans.

## Why I picked this case

I picked this case because it is more database-focused than the first two.

Support engineering is not only about fixing client-side issues. Sometimes the user problem is that the app becomes slow as data grows.

This case is interesting because the query itself can look simple, but the RLS policy can add extra work for Postgres. If the columns used in the policy are not indexed, the app may become slower as the table grows.

I wanted to use this case to practice thinking about:

- Row Level Security
- query performance
- indexes
- `EXPLAIN ANALYZE`
- safe customer communication around database changes

## Customer problem

A user says their app became slow after their `projects` table grew.

The client query is simple:

```ts
const { data, error } = await supabase
  .from('projects')
  .select('*')
```

The table uses RLS so users can only read their own projects.

Example policy:

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using (user_id = auth.uid());
```

At small scale, everything worked fine. After the table grew, dashboard loading became slow.

## Impact

The user's dashboard loads slowly.

From the user's perspective, Supabase or the client query may look slow, but the real cause may be database-level performance: the RLS policy checks `user_id`, and the database may need an index to evaluate that efficiently.

## What I would check first

1. How many rows are in the table?
2. What query is slow?
3. Is RLS enabled?
4. What policies are active on the table?
5. Which columns are used in the RLS policy?
6. Are those columns indexed?
7. What does `EXPLAIN ANALYZE` show before and after adding an index?
8. Is the query returning too many rows?
9. Are there additional filters, joins, or ordering in the real query?

## Minimal reproduction plan

I plan to reproduce this with a `projects` table that stores user-owned rows:

```sql
create table public.projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  created_at timestamptz default now()
);
```

Enable RLS:

```sql
alter table public.projects enable row level security;
```

Add a policy:

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using (user_id = auth.uid());
```

Then I would test query performance before and after adding an index on `user_id`.

Query to test:

```sql
select *
from public.projects
where user_id = auth.uid();
```

Index to test:

```sql
create index projects_user_id_idx
on public.projects (user_id);
```

## Investigation notes

My first thought would be to avoid guessing.

Instead of saying "add an index" immediately, I would first look for evidence:

```sql
explain analyze
select *
from public.projects
where user_id = auth.uid();
```

I would compare:

- query plan before index
- query plan after index
- execution time before index
- execution time after index
- whether Postgres uses a sequential scan or an index scan

If the RLS policy uses `user_id = auth.uid()`, then `user_id` is an important column for access control. If the table is large and `user_id` is not indexed, Postgres may need to scan more rows than necessary.

## Likely root cause

The likely root cause is that the RLS policy checks `user_id`, but `user_id` is not indexed.

At small scale this may not matter. As the table grows, the same query can become slower because Postgres has more rows to evaluate.

## Fix to test

Add an index on the column used in the RLS policy:

```sql
create index projects_user_id_idx
on public.projects (user_id);
```

If the app commonly orders by `created_at`, I would also consider testing a composite index:

```sql
create index projects_user_id_created_at_idx
on public.projects (user_id, created_at desc);
```

I would not add indexes blindly. I would test with `EXPLAIN ANALYZE` and compare before/after results.

## Customer-facing response draft

Thanks for the details. Since the query became slow as the table grew, I would check the RLS policy and indexes together.

Your policy uses `user_id = auth.uid()`, which means Postgres needs to evaluate `user_id` when deciding which rows the authenticated user can read.

If `projects.user_id` is not indexed, performance can become worse as the table grows. A good first test would be to compare the query plan before and after adding an index:

```sql
explain analyze
select *
from public.projects
where user_id = auth.uid();
```

Then test this index:

```sql
create index projects_user_id_idx
on public.projects (user_id);
```

After adding the index, run `EXPLAIN ANALYZE` again and compare the execution time and plan.

If your UI usually sorts recent projects first, we may also want to test a composite index like:

```sql
create index projects_user_id_created_at_idx
on public.projects (user_id, created_at desc);
```

I would start with the simple `user_id` index, measure the result, and only add more indexes if the query pattern requires it.

## Escalation note

I would not escalate this immediately because it is likely a schema/indexing issue.

I would escalate or ask for deeper database support if:

- the relevant columns are already indexed
- `EXPLAIN ANALYZE` still shows unexpectedly slow performance
- the query plan looks unusual
- the table is very large
- there are joins, functions, or complex policies involved
- performance changed suddenly without schema or traffic changes

For escalation, I would include:

- table schema
- active RLS policies
- indexes on the table
- slow query
- `EXPLAIN ANALYZE` before and after index changes
- approximate row count
- expected latency
- actual latency

## Documentation or product improvement idea

A useful docs note could say:

> If an RLS policy filters by a column such as `user_id`, make sure that column is indexed for tables that may grow large.

This would help users understand that RLS correctness and RLS performance are related but separate concerns.

## Reproduction notes

TODO after testing:

- Supabase project setup:
- Table schema:
- Number of rows:
- RLS policy:
- Indexes before:
- Query tested:
- `EXPLAIN ANALYZE` before:
- Index added:
- `EXPLAIN ANALYZE` after:
- Result before fix:
- Result after fix:
- What I learned:
