# Case 2: Realtime subscription connects but receives no events

> Status: Draft. I wrote this based on a common Supabase Realtime support scenario and will update it after reproducing the issue in a local or hosted Supabase project.

## Why I picked this case

I picked this case because Realtime issues can be confusing for users.

A subscription can appear to connect successfully, but the user still receives no database change events. This makes the problem harder to debug because there may not be an obvious error message.

I wanted to write this up because it tests a support engineer's ability to debug across several layers:

- client code
- authentication state
- database changes
- Realtime configuration
- Row Level Security
- Postgres publication settings

## Customer problem

A user says their Realtime subscription connects, but they do not receive any events when rows are inserted or updated.

Example client code:

```ts
const channel = supabase
  .channel('projects-changes')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'projects',
    },
    (payload) => {
      console.log('Change received:', payload)
    }
  )
  .subscribe((status) => {
    console.log('Subscription status:', status)
  })
```

Observed result:

```ts
Subscription status: SUBSCRIBED
```

But when a row is inserted or updated, the callback does not log any payload.

## Impact

The user's app depends on live updates, but the UI does not update when database rows change.

From the user's perspective, the subscription looks connected, but Realtime appears broken.

## What I would check first

1. Is the client actually reaching `SUBSCRIBED` status?
2. Is the table included in Realtime/Postgres changes?
3. Is the user listening to the correct schema and table?
4. Is the event filter correct?
5. Is Row Level Security enabled on the table?
6. If RLS is enabled, does the user have permission to read the changed row?
7. Is the database change happening after the subscription is active?
8. Is the app running in a backgrounded tab or environment where the connection may silently disconnect?
9. Are there any client-side errors or network disconnects?

## Minimal reproduction plan

I plan to reproduce this with a small `projects` table:

```sql
create table public.projects (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz default now()
);
```

Then I would enable Realtime/Postgres changes for the table and subscribe from a small JavaScript client.

Test subscription:

```ts
const channel = supabase
  .channel('projects-changes')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'projects',
    },
    (payload) => {
      console.log('INSERT received:', payload)
    }
  )
  .subscribe((status) => {
    console.log('status:', status)
  })
```

Then insert a row:

```sql
insert into public.projects (name)
values ('Realtime test project');
```

Expected working result:

```ts
status: SUBSCRIBED
INSERT received: { ...payload }
```

Expected failing result:

```ts
status: SUBSCRIBED
// no payload received after insert
```

## Investigation notes

My first step would be to confirm whether the issue is connection-related or event-related.

If the client never reaches `SUBSCRIBED`, I would focus on connection, keys, URL, or network problems.

If the client reaches `SUBSCRIBED` but receives no events, I would focus on:

- table configuration
- event filter
- schema/table mismatch
- RLS visibility
- whether the database change is actually happening

I would also simplify the filter.

Instead of starting with a specific event or condition, I would test:

```ts
{
  event: '*',
  schema: 'public',
  table: 'projects'
}
```

If that works, then I would narrow the filter again.

## Possible root causes

The most likely root causes are:

1. The table is not configured for Realtime/Postgres changes.
2. The client is subscribed to the wrong table or schema.
3. The event filter does not match the actual database change.
4. RLS prevents the authenticated user from seeing the changed row.
5. The database change happened before the subscription was active.
6. The connection was silently disconnected, especially in a backgrounded app or long-running session.

## Fixes to test

### 1. Confirm the subscription status

I would first log the subscription status:

```ts
.subscribe((status) => {
  console.log('Realtime status:', status)
})
```

If the status is not `SUBSCRIBED`, I would debug the connection before debugging the table.

### 2. Test with the simplest possible filter

```ts
.on(
  'postgres_changes',
  {
    event: '*',
    schema: 'public',
    table: 'projects',
  },
  (payload) => {
    console.log(payload)
  }
)
```

### 3. Confirm that the database change happens after subscription

I would subscribe first, wait until `SUBSCRIBED`, and only then insert or update a row.

### 4. Check RLS behavior

If RLS is enabled, I would confirm whether the authenticated user can select the changed row.

Example check:

```ts
const { data, error } = await supabase
  .from('projects')
  .select('*')
```

If the user cannot read the row with a normal `select`, I would not expect them to reliably receive Realtime changes for that row either.

### 5. Add or adjust a select policy

Example policy:

```sql
create policy "Authenticated users can read projects"
on public.projects
for select
to authenticated
using (true);
```

For a real app, I would avoid `using (true)` unless the table is meant to be readable by all authenticated users. For user-owned rows, I would use an ownership check instead.

## Customer-facing response draft

Thanks for the details. Since your subscription reaches `SUBSCRIBED`, I would separate this into two questions:

1. Is the Realtime connection active?
2. Are database changes matching the subscription and visible to this user?

The first thing I would try is simplifying the subscription filter:

```ts
const channel = supabase
  .channel('projects-changes')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'projects',
    },
    (payload) => {
      console.log('Change received:', payload)
    }
  )
  .subscribe((status) => {
    console.log('Realtime status:', status)
  })
```

After the status logs `SUBSCRIBED`, insert a new row into `projects`.

If you still do not receive a payload, I would next check whether the table is enabled for Realtime/Postgres changes and whether Row Level Security is preventing the authenticated user from reading the row.

A useful test is to run a normal `select()` from the same client session:

```ts
const { data, error } = await supabase
  .from('projects')
  .select('*')
```

If that returns an empty array because of RLS, then the issue may be authorization rather than the Realtime client code.

## Escalation note

I would escalate this only after confirming:

- the client reaches `SUBSCRIBED`
- the table is configured for Realtime/Postgres changes
- the schema/table/event filter is correct
- the database change happens after subscription
- RLS policies allow the user to read the changed row
- the issue is reproducible with a minimal project

For escalation, I would include:

- client code
- subscription status logs
- table schema
- RLS policies
- Realtime table configuration
- exact insert/update SQL used for testing
- expected payload
- actual behavior
- browser/runtime environment

## Documentation or product improvement idea

Realtime issues are difficult because `SUBSCRIBED` can make users think everything is fully working.

A useful troubleshooting checklist could explain:

> `SUBSCRIBED` means the client joined the channel. If no events arrive, check table configuration, event filters, timing of the database change, and RLS visibility.

This would help users debug the difference between connection success and event delivery.

## Reproduction notes

TODO after testing:

- Supabase project setup:
- Table schema:
- Realtime enabled:
- RLS enabled or disabled:
- Policies before:
- Subscription code:
- Subscription status:
- Insert/update SQL:
- Result before fix:
- Fix applied:
- Result after fix:
- What I learned:
