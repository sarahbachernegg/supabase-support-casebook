# Case 1: Authenticated user sees no rows because of RLS

> Status: Reproduced in a hosted Supabase project using a test user, RLS-enabled table, and a small Node.js client script.

## Why I picked this case

I picked this because it seems like a common Supabase support issue: a user is logged in, the table has data, but the client query returns an empty array.

At first glance this can look like an auth or client-side bug. But in Supabase, authentication and authorization are separate. A user can be logged in successfully and still not be allowed to see any rows because of Row Level Security.

I wanted to write this up because it is exactly the kind of issue where clear support communication matters.

## Customer problem

A user says their login works, but this query returns no rows:

```ts
const { data, error } = await supabase
  .from('projects')
  .select('*')
```

Observed result:

```ts
data = []
error = null
```

The user can see rows in the Supabase dashboard, so they expect the app to return those rows too.

## Impact

The user's dashboard appears empty.

From the user's perspective, the app looks broken even though authentication works. This can be confusing because there is no obvious error message.

## What I would check first

1. Is Row Level Security enabled on the table?
2. Is there a `select` policy for the `authenticated` role?
3. Does the table have a `user_id` or `owner_id` column?
4. Does that column store the same UUID as `auth.uid()`?
5. Is the client using the authenticated user's session?
6. Does the same query work with the service role key?

## Minimal reproduction plan

I plan to reproduce this with a small `projects` table:

```sql
create table public.projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null
);
```

Then enable RLS:

```sql
alter table public.projects enable row level security;
```

The expected failing state is:

- user is authenticated
- table has rows
- RLS is enabled
- no matching `select` policy exists
- client query returns `data = []` and `error = null`

Client query:

```ts
const { data, error } = await supabase
  .from('projects')
  .select('*')
```

## Investigation notes

My first thought would be to separate authentication from authorization.

Authentication answers:

> Is the user logged in?

Authorization answers:

> Is this user allowed to read these rows?

In this case, the user may be authenticated correctly, but Postgres RLS can still filter out every row.

After reproducing this, I want to compare:

```sql
select auth.uid();
```

with the table's ownership column:

```sql
select id, user_id, name
from public.projects;
```

If `projects.user_id` does not match `auth.uid()`, the policy will not return rows for that user.

## Likely root cause

The most likely root cause is one of these:

1. RLS is enabled but no `select` policy exists.
2. A policy exists, but it does not match the authenticated user's ID.
3. The table stores ownership in a different column than the policy expects.
4. The client is not sending the authenticated user's session/JWT.

The important detail is that an empty array is not always a failed query. It can also mean the query succeeded, but RLS allowed zero rows to be returned.

## Fix to test

A common fix is to create a `select` policy that allows each authenticated user to read their own rows:

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using (user_id = auth.uid());
```

If users also create projects from the client, an insert policy may be needed too:

```sql
create policy "Users can create their own projects"
on public.projects
for insert
to authenticated
with check (user_id = auth.uid());
```

## Customer-facing response draft

Thanks for the details. Since authentication works but the query returns an empty array, I would first check the Row Level Security policy on the `projects` table.

In Supabase, logging in successfully does not automatically mean the user can read rows from a table. If RLS is enabled, Postgres only returns rows that match an active policy.

A good first check is whether the value in `projects.user_id` matches the logged-in user's `auth.uid()`.

For example, if each project belongs to one user, this policy would allow authenticated users to read only their own projects:

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using (user_id = auth.uid());
```

If this fixes the empty result, the issue was likely authorization/RLS rather than the client query itself.

If it still returns an empty array, I would next check whether the client request includes the authenticated session and whether the stored `user_id` values actually match the user's auth ID.

## Escalation note

I would not escalate this immediately because this is most likely expected RLS behavior.

I would escalate only if:

- RLS policy exists and looks correct
- `auth.uid()` matches the row's `user_id`
- the client is sending a valid authenticated session
- the issue can be reproduced with a minimal example
- the behavior differs from what the docs describe

For escalation, I would include:

- table schema
- active RLS policies
- example client query
- expected result
- actual result
- whether the same query works with the service role key
- a minimal reproduction if possible

## Documentation or product improvement idea

This issue is confusing because `data = []` and `error = null` can look like the table is empty.

A useful docs or dashboard improvement could be a troubleshooting note near RLS examples:

> If a client query returns an empty array with no error, check whether RLS is enabled and whether any policy matches the current authenticated user.

This would help users understand that the query may be working correctly, but returning zero visible rows because of authorization rules.

## Reproduction notes

I reproduced this in a hosted Supabase project with a small `test` table.

### Setup

Table schema:

```sql
create table public.projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  created_at timestamptz default now()
);

alter table public.projects enable row level security;
```

I created one test user through Supabase Auth and inserted one project row owned by that user:

```sql
insert into public.projects (user_id, name)
values
  ('<test-user-id>', 'Demo project owned by test user');
```

### Client test

I used a small Node.js script with `@supabase/supabase-js`, the project URL, and the anon public key.

The script signs in as the test user and runs:

```js
const { data, error } = await supabase
  .from('projects')
  .select('*')
```

### Before adding a SELECT policy

With RLS enabled and no matching `select` policy, the signed-in user could authenticate successfully, but the query returned no rows:

```text
Signed in user: <test-user-id>
Query result: { data: [], error: null }
```

### Policy added

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using (user_id = auth.uid());
```

### After adding the SELECT policy

Running the same script again returned the project row owned by the authenticated user:

```text
Signed in user: <test-user-id>
Query result: {
  data: [
    {
      id: '<project-id>',
      user_id: '<test-user-id>',
      name: 'Demo project owned by test user',
      created_at: '<timestamp>'
    }
  ],
  error: null
}
```

### What I learned

The important part is that the first query did not fail. The request succeeded with `error = null`, but RLS filtered out every row because no policy allowed the authenticated user to read them.

Authentication confirmed who the user was. The RLS policy decided which rows that user could access.
- Policy after:
- Query result after fix:
- What I learned:
