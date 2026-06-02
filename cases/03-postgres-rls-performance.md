# Case 3: RLS policy is correct, but the query plan does too much work

**Status:** Reproduced in a hosted Supabase project with a 100,000-row table. I compared `EXPLAIN ANALYZE` output before and after adding an index on the column used for row ownership.

## Why I picked this case

I picked this case because it is more database-focused than the first two.

Case 1 was about authorization. Case 2 was about Realtime configuration. This one is about what can happen after the policy is technically correct, but the table grows and Postgres has to evaluate more rows than expected.

The interesting part of this reproduction was that the index did not create a dramatic speedup in my small hosted test. The query plan changed, but the runtime stayed similar. That made the case more useful for me, not less, because it reminded me not to describe indexes as magic fixes.

I wanted to use this case to practice:

- reading `EXPLAIN ANALYZE`
- connecting RLS policies to query planning
- checking whether policy columns are indexed
- explaining performance issues without overpromising
- suggesting safe next steps instead of guessing

## Customer problem

A user says their app became slow after their `projects` table grew.

The client query looks simple:

```js
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
using ((select auth.uid()) = user_id);
```

At small scale, everything worked fine. After the table grew, dashboard loading became slow.

## Impact

The user's dashboard loads slowly.

From the user's perspective, Supabase or the client query may look slow. But the issue may be lower down: the database might be scanning more rows than needed while evaluating the ownership condition used by the RLS policy.

The policy can be correct from a security point of view and still deserve a performance review.

## Reproduction

| Item | Details |
|---|---|
| Environment | Hosted Supabase project |
| Table tested | `public.performance_projects` |
| Rows tested | 100,000 |
| Rows owned by test user | Around 1,000 |
| Failure pattern | Query needed to filter out most rows before the ownership column was indexed |
| Tool used | `EXPLAIN ANALYZE` |
| Fix tested | Added an index on `user_id` |
| Result after fix | Query plan changed from sequential scan to bitmap index scan |
| Important caveat | Runtime was similar in this small test, so I would not claim a guaranteed speedup |

## What I checked first

I would not start by saying “add an index” without looking at the query plan.

First, I would check:

- How many rows are in the table?
- Which query is slow?
- Is RLS enabled?
- Which policies are active on the table?
- Which columns are used in the RLS policy?
- Are those columns indexed?
- Is the query returning too many rows?
- Are there filters, joins, ordering, or limits in the real query?
- What does `EXPLAIN ANALYZE` show before and after any index change?

For this symptom, the key question is not only:

> Is the policy correct?

It is also:

> Can Postgres evaluate the policy efficiently as the table grows?

## Reproduction notes

I reproduced this with a `performance_projects` table.

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
using ((select auth.uid()) = user_id);
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

For the query plan test, I used a direct UUID instead of `auth.uid()` because `auth.uid()` depends on the authenticated request context and can be `null` in the SQL editor.

The tested query was:

```sql
explain analyze
select *
from public.performance_projects
where user_id = '<test-user-id>'::uuid;
```

This is not a perfect copy of the client request, but it is useful for checking the same ownership condition that the RLS policy depends on.

## Before adding an index

Before adding an index on `user_id`, Postgres used a sequential scan:

```txt
Seq Scan on performance_projects
Filter: (user_id = '<test-user-id>'::uuid)
Rows Removed by Filter: 99000
Returned rows: 1000
Execution time: 11.26ms
```

This means Postgres scanned the full 100,000-row table and filtered out 99,000 rows.

That does not automatically prove the query is slow in every environment, but it is a useful signal: the database did not have a better access path for the ownership condition.

## Index added

I added an index on the ownership column:

```sql
create index performance_projects_user_id_idx
on public.performance_projects (user_id);
```

For a live production table, I would be more careful before running this directly. I would consider table size, write traffic, maintenance windows, and whether `create index concurrently` is more appropriate.

## After adding the index

After adding the index, Postgres used the `performance_projects_user_id_idx` index:

```txt
Bitmap Heap Scan on performance_projects
Recheck Cond: (user_id = '<test-user-id>'::uuid)
Heap Blocks: exact=1000
Returned rows: 1000
Execution time: 11.51ms

Bitmap Index Scan on performance_projects_user_id_idx
Index Cond: (user_id = '<test-user-id>'::uuid)
```

In this small hosted test, execution time was similar:

- before index: `11.26ms`
- after index: `11.51ms`

The important difference was the plan.

Before the index, Postgres scanned the full table. After the index, Postgres used the ownership column index to find the matching rows.

This was a good result because it kept me honest. I would not tell a user, “This index will definitely make the query faster.” I would say, “The current plan scans the whole table. An index gives Postgres a better access path, and we should compare the plan before and after.”

## Investigation notes

The main thing I would explain to the user is that RLS correctness and RLS performance are related, but they are not the same thing.

A policy like this can be correct:

```sql
using ((select auth.uid()) = user_id)
```

But if the table grows and `user_id` is not indexed, Postgres may have to evaluate many rows to find the ones visible to the user.

That is why I would look at:

- the policy condition
- the query plan
- the indexes on the table
- the number of rows removed by filters
- whether the query also sorts, joins, or filters by other columns

I would also check the real query shape. For example, this query may need a different index:

```js
const { data, error } = await supabase
  .from('projects')
  .select('*')
  .order('created_at', { ascending: false })
  .limit(50)
```

For that pattern, I would test a composite index such as:

```sql
create index performance_projects_user_id_created_at_idx
on public.performance_projects (user_id, created_at desc);
```

I would only keep that index if the real query plan and workload justify it.

## Likely root cause

For this symptom, I would first look for one of these causes:

- the RLS policy checks `user_id`, but `user_id` is not indexed
- the query returns too many rows
- the query sorts or filters on columns that are not supported by an index
- the policy calls functions or joins that add work per row
- the table statistics are stale
- the real app query is more complex than the simplified example

In my reproduction, the clearest issue was that the ownership column used by the policy did not have an index.

## Fix to test

For a user-owned table, the first index I would test is:

```sql
create index performance_projects_user_id_idx
on public.performance_projects (user_id);
```

If the app commonly orders the user's rows by creation time, I would also test:

```sql
create index performance_projects_user_id_created_at_idx
on public.performance_projects (user_id, created_at desc);
```

I would not add indexes blindly.

Before recommending the final change, I would compare:

- `EXPLAIN ANALYZE` before the index
- `EXPLAIN ANALYZE` after the index
- whether the plan uses the index
- whether rows removed by filter decrease
- actual latency in the application
- write overhead and storage cost
- whether the index matches the real query pattern

## Production safety note

For my reproduction, I used a normal `create index` statement.

For a large production table, I would not suggest running that casually during peak traffic. Index creation can affect a live database, and the safer approach may be to create the index concurrently, depending on the situation:

```sql
create index concurrently performance_projects_user_id_idx
on public.performance_projects (user_id);
```

I would also mention that `create index concurrently` has tradeoffs: it can take longer, uses more work, and cannot run inside a transaction block.

So my support guidance would be:

> Test the index on a staging or development project first, review the query plan, and be careful about how and when you apply it to production.

## Support reply I would send to the user

Thanks for sharing the query and policy. Since the table has grown and the policy checks ownership with `user_id`, I would look at the RLS policy and indexes together.

The policy may be correct from an access-control point of view, but Postgres still needs to evaluate the column used in that policy. If `user_id` is not indexed, the database may need to scan many rows as the table grows.

I would first check the query plan with the user ID replaced by a test UUID:

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

Then I would run `EXPLAIN ANALYZE` again and compare the plan.

In my own test with 100,000 rows, the query changed from a sequential scan that filtered out 99,000 rows to a bitmap index scan using the `user_id` index. The runtime was similar in that small dataset, so I would not claim the index is automatically faster in every case. But it did give Postgres a better access path, which is exactly what I would check for an RLS policy based on row ownership.

If the query is still slow after indexing `user_id`, I would next look at the full query pattern: ordering, limits, joins, additional filters, row count, and whether a composite index would fit the actual access pattern.

Please share a sanitized table schema, active RLS policies, existing indexes, the slow query, and the `EXPLAIN ANALYZE` output if available. Please remove API keys, JWTs, service-role keys, email addresses, and real user or project IDs before sharing.

## Escalation note

I would not escalate this immediately if the evidence points to a normal schema or indexing issue.

I would escalate or ask for deeper database support if:

- the relevant columns are already indexed
- `EXPLAIN ANALYZE` still shows unexpectedly slow performance
- the query plan looks unusual
- the table is very large
- there are joins, functions, or complex policies involved
- performance changed suddenly without schema, traffic, or data-volume changes
- the issue can be reproduced but does not match the expected query plan behavior

For escalation, I would include:

- sanitized table schema
- active RLS policies
- existing indexes on the table
- slow query with secrets and real IDs removed
- `EXPLAIN ANALYZE` before and after index changes
- approximate row count
- expected latency
- actual latency
- whether the issue happens through the API, direct Postgres connection, or both
- relevant logs, timestamps, and request IDs if available
- a minimal reproduction if possible

## Documentation or product improvement idea

This issue is easy to miss because users often think about RLS as only a security feature.

A useful docs or dashboard note could say:

> If an RLS policy filters by a column such as `user_id`, check whether that column is indexed for tables that may grow large. The policy can be correct, but the query plan may still scan more rows than necessary.

This would help users understand that RLS correctness and RLS performance are related, but separate, concerns.

It could also be useful for the Dashboard policy editor or Performance Advisor to surface a reminder when a policy uses a non-indexed ownership column.

## What I learned

This test was a good reminder not to describe indexes as magic fixes.

The index changed the query plan, but the measured runtime in this small dataset stayed similar. That was useful because it forced me to stay precise: the index did not prove a dramatic speedup in my reproduction, but it did change the way Postgres accessed the data.

The support takeaway is:

- do not guess
- look at the query plan
- check the columns used in the RLS policy
- index ownership columns when the evidence supports it
- validate before and after with `EXPLAIN ANALYZE`
- avoid promising a performance improvement that has not been measured

If an RLS policy depends on a column like `user_id`, that column is a strong candidate for indexing, especially as the table grows or when queries add sorting, limits, joins, or additional filters.
