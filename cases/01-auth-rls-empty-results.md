# Case 1: Authenticated user sees no rows because of RLS

**Status:** Reproduced in a hosted Supabase project using a test user, an RLS-enabled table, and a small Node.js client script.

## Why I chose this case

I chose this case because it can look like a client-side bug at first. The user is signed in, the query does not throw an error, and the table has rows. But the app still receives an empty array.

When I reproduced it, the important part was not the `select()` call. The request was working. The missing piece was authorization: the user was authenticated, but no RLS policy allowed that user to read the row.

That made this a useful support case because the first instinct can easily be to debug the wrong layer.

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

The users dashboard appears empty. From the users point of view, the app looks broken even though authentication is working. The confusing part is that the response shape looks successful - there is no error, just no visible data.

## Reproduction

| Item | Details |
|---|---|
| Environment | Hosted Supabase project |
| Client | Small Node.js script using `@supabase/supabase-js` |
| Table | `public.projects` |
| Failure reproduced | Signed-in user received `data = []`, `error = null` |
| Fix tested | Added a `SELECT` policy for the `authenticated` role |
| Result after fix | The same query returned the project row owned by the test user |

## What I checked first

I wanted to separate authentication from authorization before changing anything.

The checks I would make are:

1. Is Row Level Security enabled on the table?
2. Is there a `select` policy for the `authenticated` role?
3. Does the table store ownership in a column such as user_id or owner_id?
4. Does that value match the logged-in users ID?
5. Is the client actually using the authenticated users session?
6. Does the same query return rows when tested with the service role key?

The service role check is not a fix for the client app. I would only use it to confirm whether the data exists and whether RLS is the likely reason the user cannot see it.

## Reproduction notes

I reproduced this with a small projects table.

### Setup

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

The script signs in as the test user and then runs:

```ts
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

This was the key part of the reproduction. The query did not fail. It returned exactly what the current RLS setup allowed the user to see: nothing.

### Policy added

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using ((select auth.uid()) = user_id);
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

## Investigation notes

The main thing I would explain to the user is that signing in and being allowed to read rows are separate steps.

Authentication answers:

> Who is the user?

Authorization answers:

> Which rows is this user allowed to access?

In this case, authentication was working. The user had a valid session. But once the query reached Postgres, RLS still decided whether any rows were visible to that user. That is why `data = []` and `error = null` is an important clue. It can mean the request succeeded, but the current policies allowed zero rows to be returned.

## Likely root cause

The most likely root cause is one of these:

1. RLS is enabled but no `select` policy exists.
2. A policy exists, but it does not match the authenticated users ID.
3. The table stores ownership in a different column than the policy expects.
4. The client is not sending the authenticated users session/JWT.

In my reproduction, the missing piece was the `select` policy.

## Fix

For a table where each project belongs to one user, a common fix is to add a `SELECT` policy that allows authenticated users to read only their own rows:

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using ((select auth.uid()) = user_id);

If users also create projects from the client, an INSERT policy may be needed as well:

create policy "Users can create their own projects"
on public.projects
for insert
to authenticated
with check ((select auth.uid()) = user_id);```

For a real application, I would first check the data model before suggesting this exact policy. If projects belong to teams or organizations, a simple user_id = auth.uid() ownership policy may be too narrow.

## Support reply I would send to the user

Thanks for sharing the query and result. Since the query returns `data = []` with `error = null`, I would first check the RLS setup on the `projects` table.

This usually means the request itself is not failing. Instead, Postgres is returning zero rows that are visible to the current user. In Supabase, signing in confirms who the user is, but RLS policies still decide which rows that user can read.

I would check these four things first:

1. Is RLS enabled on `projects`?
2. Is there a `SELECT` policy for the `authenticated` role?
3. Does `projects.user_id` store the same UUID as the logged-in user's `auth.uid()`?
4. Is the client request sending the authenticated user's JWT?

If each project belongs to one user, please test this policy:

```sql
create policy "Users can read their own projects"
on public.projects
for select
to authenticated
using ((select auth.uid()) = user_id);

Then run the same client query again:

const { data, error } = await supabase
  .from('projects')
  .select('*')

If the row appears after adding the policy, the issue was not the select() call itself. The query was working, but RLS was filtering out the row because no policy allowed this user to read it.

If it still returns an empty array, please share a sanitized table schema, the active RLS policies, and the client code used for the query. Please remove API keys, JWTs, service-role keys, email addresses, and real user or project IDs before sharing.

## Escalation note

I would not escalate this immediately because the behavior matches expected RLS behavior.

I would escalate only if:

- the RLS policy exists and looks correct
- `auth.uid()` matches the row's `user_id`
- the client is sending a valid authenticated session
- the issue can be reproduced with a minimal example
- the behavior differs from what the docs describe

For escalation, I would include:

- sanitized table schema
- active RLS policies
- example client query with secrets removed
- expected result
- actual result
- authenticated user ID used for testing, replaced with a placeholder
- whether `auth.uid()` matches the row's ownership column
- whether the same query works from a trusted server-side environment with the service-role key
- relevant logs, timestamps, and request IDs if available
- a minimal reproduction if possible

## Documentation or product improvement idea

Supabase already documents that `data = []` with existing table rows can be caused by RLS policies not matching the current user. I think this could be surfaced even closer to the RLS policy examples or in the Dashboard policy editor, because this symptom can look like a client-side bug when users first encounter it.

A short note could say:

If a client query returns an empty array with no error, check whether RLS is enabled and whether any policy matches the current authenticated user. Also confirm that the request includes a valid user JWT and that `auth.uid()` matches the ownership column used in the policy.

## What I learned

The query did not fail. The request succeeded with error = null, but RLS filtered out every row because no policy allowed the authenticated user to read them. 

The useful debugging split was:

- first confirm the user is authenticated
- then confirm the user is authorized to read the expected rows

That distinction is easy to miss when the app only shows an empty dashboard.
