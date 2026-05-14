# Case 3: Slow queries caused by RLS policy checks on non-indexed columns

> Status: Reproduced in a hosted Supabase project with a 100,000-row table. I compared `EXPLAIN ANALYZE` output before and after adding an index on the column used for row ownership.
> 
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

## Reply I would send

Thanks for sharing the query and policy. Since the table has grown and the policy checks `user_id = auth.uid()`, I would look at the RLS policy and indexes together.

The policy may be correct from an access-control point of view, but Postgres still needs to evaluate the column used in that policy. If `user_id` is not indexed, the database may need to scan many more rows as the table grows.

I would first check the query plan:

```sql
explain analyze
select *
from public.projects
where user_id = '<user-id>'::uuid;
```

If the plan shows a sequential scan and many rows removed by the filter, I would test an index on the ownership column:

```sql
create index projects_user_id_idx
on public.projects (user_id);
```

Then run `EXPLAIN ANALYZE` again and compare the plan.

In my own test with 100,000 rows, the query changed from a sequential scan that filtered out 99,000 rows to a bitmap index scan using the `user_id` index. The runtime was similar in that small dataset, so I would not claim the index is automatically faster in every case. But it did give Postgres a better access path and is the first thing I would check for an RLS policy based on row ownership.

If the query is still slow after indexing `user_id`, I would next look at the full query pattern: ordering, limits, joins, additional filters, row count, and whether a composite index would fit the actual access pattern.
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

I reproduced this in a hosted Supabase project with a `performance_projects` table containing 100,000 rows.

### Setup

```sql
create table public.performance_projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  created_at timestamptz default now()
);

alter table public.performance_projects enable row level security;

create policy "Users can read their own performance projects"
on public.performance_projects
for select
to authenticated
using (user_id = auth.uid());
```

I inserted 100,000 test rows. Around 1,000 rows belonged to the test user, and the rest belonged to random users:

```sql
insert into public.performance_projects (user_id, name)
select
  case
    when i % 100 = 0 then '<test-user-id>'::uuid
    else gen_random_uuid()
  end as user_id,
  'Project ' || i as name
from generate_series(1, 100000) as i;
```

For the query plan test, I used a direct UUID instead of `auth.uid()` because `auth.uid()` depends on the request context and can be `null` in the SQL editor.

### Query tested

```sql
explain analyze
select *
from public.performance_projects
where user_id = '<test-user-id>'::uuid;
```

### Before adding an index

Before adding an index on `user_id`, Postgres used a sequential scan:

```text
Seq Scan on performance_projects
Filter: (user_id = '<test-user-id>'::uuid)
Rows Removed by Filter: 99000
Returned rows: 1000
Execution time: 11.26ms
```

This means Postgres scanned the full 100,000-row table and filtered out 99,000 rows.

### Index added

```sql
create index performance_projects_user_id_idx
on public.performance_projects (user_id);
```

### After adding the index

After adding the index, Postgres used the `performance_projects_user_id_idx` index:

```text
Bitmap Heap Scan on performance_projects
Recheck Cond: (user_id = '<test-user-id>'::uuid)
Heap Blocks: exact=1000
Returned rows: 1000
Execution time: 11.51ms

Bitmap Index Scan on performance_projects_user_id_idx
Index Cond: (user_id = '<test-user-id>'::uuid)
```

In this small test, execution time was similar. The important difference was the query plan: before the index, Postgres scanned the full table; after the index, it used the ownership column index.

### What I learned

This test was a good reminder not to describe indexes as magic fixes. The index changed the query plan, but the measured runtime in this small dataset stayed similar.

The support takeaway is that if an RLS policy depends on a column like `user_id`, that column is a strong candidate for indexing, especially as the table grows or when queries add sorting, limits, joins, or additional filters.

I would still validate the change with `EXPLAIN ANALYZE` instead of assuming the index improves every query.

