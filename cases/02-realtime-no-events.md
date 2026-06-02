# Case 2: Realtime subscription connects but receives no events

**Status:** Reproduced in a hosted Supabase project. The subscription reached `SUBSCRIBED`, and the insert succeeded, but no payload was received until the table was enabled for Postgres Changes / Realtime.

## Why I picked this case

I picked this case because it can look successful at the connection layer while still failing at the event-delivery layer.

When I reproduced it, the misleading part was that the client printed `SUBSCRIBED` and the insert returned successfully. My first instinct would have been to keep checking the JavaScript subscription code, but the missing piece was lower down: the table was not publishing Postgres change events to Realtime.

That made this a useful support case because it forced me to separate a few things that are easy to mix together:

- did the client join the channel?
- is the table publishing database changes?
- does the event filter match the change?
- did the change happen after the subscription was active?
- can the authenticated user read the changed row under RLS?

## Customer problem

A user says their Realtime subscription connects, but they do not receive any events when rows are inserted or updated.

Example client code:

```js
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

```txt
Subscription status: SUBSCRIBED
```

But when a row is inserted or updated, the callback does not log any payload.

## Impact

The user's app depends on live updates, but the UI does not update when database rows change.

From the user's point of view, the subscription looks connected, but Realtime appears broken. The confusing part is that there may be no obvious error message.

## Reproduction

| Item | Details |
|---|---|
| Environment | Hosted Supabase project |
| Client | Small Node.js script using `@supabase/supabase-js` |
| Table tested | `public.realtime_projects` |
| Failure reproduced | Channel reached `SUBSCRIBED`, insert succeeded, but no payload was received |
| Fix tested | Enabled Postgres Changes / Realtime for the table |
| Result after fix | The same script received the `INSERT` payload |

## What I checked first

I wanted to split the problem into connection, configuration, event matching, and permissions.

The checks I would make are:

- Does the client reach `SUBSCRIBED`?
- Is the table included in Postgres Changes / Realtime?
- Is the client listening to the correct schema and table?
- Does the event filter match the database change?
- Does the database change happen after the subscription is active?
- Is Row Level Security enabled on the table?
- If RLS is enabled, can the authenticated user read the changed row?
- Are there client-side errors, network disconnects, or background-tab issues?

## Reproduction notes

I reproduced this with a small `realtime_projects` table.

For this test, I kept RLS out of scope at first so I could isolate the Realtime configuration.

### Setup

```sql
create table public.realtime_projects (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz default now()
);
```

### Client test

I used a small Node.js script with `@supabase/supabase-js`, the project URL, and the anon public key.

The script subscribes to `INSERT` events on `public.realtime_projects`:

```js
const channel = supabase
  .channel('realtime-projects-test')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'realtime_projects',
    },
    (payload) => {
      console.log('INSERT received:', payload)
    }
  )
  .subscribe((status) => {
    console.log('Subscription status:', status)
  })
```

After the subscription reached `SUBSCRIBED`, the script inserted a test row:

```js
const { data, error } = await supabase
  .from('realtime_projects')
  .insert({ name: 'Realtime test project' })
  .select()

console.log('Insert result:', { data, error })
```

## Before enabling Postgres Changes for the table

The client joined the channel, and the insert succeeded, but no Realtime payload was received:

```txt
Subscription status: SUBSCRIBED
Inserting test row...
Insert result: {
  data: [
    {
      id: '<project-id>',
      name: 'Realtime test project',
      created_at: '<timestamp>'
    }
  ],
  error: null
}
Subscription status: CLOSED
Test finished.
```

This was the important part of the reproduction. The socket/channel connection worked, and the database insert worked, but the table was not publishing the database change to Realtime.

## Fix applied

I enabled Postgres Changes / Realtime for the `public.realtime_projects` table.

This can be done in the Supabase Dashboard by adding the table to the `supabase_realtime` publication, or with SQL:

```sql
alter publication supabase_realtime
add table public.realtime_projects;
```

## After enabling Postgres Changes for the table

Running the same script again returned the Realtime payload:

```txt
Subscription status: SUBSCRIBED
Inserting test row...
Insert result: {
  data: [
    {
      id: '<project-id>',
      name: 'Realtime test project',
      created_at: '<timestamp>'
    }
  ],
  error: null
}
INSERT received: {
  schema: 'public',
  table: 'realtime_projects',
  commit_timestamp: '<timestamp>',
  eventType: 'INSERT',
  new: {
    id: '<project-id>',
    name: 'Realtime test project',
    created_at: '<timestamp>'
  },
  old: {},
  errors: null
}
Subscription status: CLOSED
Test finished.
```

## Investigation notes

The first split I would make is:

- If the client never reaches `SUBSCRIBED`, debug the connection first.
- If the client reaches `SUBSCRIBED` but receives no payload, debug event delivery.

In this reproduction, `SUBSCRIBED` only proved that the client joined the Realtime channel. It did not prove that a matching database change would be delivered.

Once the connection was confirmed, I focused on:

- table Realtime configuration
- schema and table name
- event type
- timing of the insert
- whether RLS could hide the changed row from the subscribed user

I would also simplify the filter while debugging. Instead of starting with a specific event or condition, I would test the broadest useful subscription first:

```js
.on(
  'postgres_changes',
  {
    event: '*',
    schema: 'public',
    table: 'realtime_projects',
  },
  (payload) => {
    console.log('Change received:', payload)
  }
)
```

If that works, I would narrow the filter again.

## Likely root cause

In my reproduction, the root cause was:

> The client subscribed successfully, but the table was not enabled for Postgres Changes / Realtime.

Other causes I would check for the same symptom are:

- the client is subscribed to the wrong schema or table
- the event filter does not match the actual database change
- the database change happened before the subscription was active
- RLS prevents the authenticated user from reading the changed row
- the client disconnected after subscribing
- the app is running in an environment where the connection is paused or backgrounded

## Fixes to test

### 1. Confirm the subscription status

```js
.subscribe((status) => {
  console.log('Realtime status:', status)
})
```

If the status is not `SUBSCRIBED`, I would debug the connection before debugging the table.

### 2. Confirm that the table is enabled for Postgres Changes

In the Supabase Dashboard, check whether the table is enabled under the `supabase_realtime` publication.

The SQL equivalent is:

```sql
alter publication supabase_realtime
add table public.realtime_projects;
```

### 3. Test with the simplest useful filter

```js
const channel = supabase
  .channel('realtime-projects-debug')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'realtime_projects',
    },
    (payload) => {
      console.log('Change received:', payload)
    }
  )
  .subscribe((status) => {
    console.log('Realtime status:', status)
  })
```

### 4. Insert only after the subscription is active

I would subscribe first, wait until the status logs `SUBSCRIBED`, and only then insert or update a row.

### 5. Check RLS behavior

If RLS is enabled, I would confirm whether the authenticated user can read the changed row with a normal query:

```js
const { data, error } = await supabase
  .from('realtime_projects')
  .select('*')
```

If the user cannot read the row with a normal `select()`, I would not expect that user to receive Realtime changes for that row either.

I would not start by relaxing RLS. I would first confirm whether RLS is actually part of the failure. If it is, I would adjust the policy based on the real data model.

## Support reply I would send to the user

Thanks for sharing the code and logs. Since your subscription reaches `SUBSCRIBED`, I would separate this into two questions:

1. Did the client join the Realtime channel?
2. Is a matching database change being delivered to that channel?

`SUBSCRIBED` tells us the first part is working. It does not necessarily prove that the table is configured to publish Postgres change events.

I would first check whether the table is enabled for Postgres Changes / Realtime. In Supabase, the table needs to be part of the `supabase_realtime` publication for database change events to be delivered.

As a quick test, I would keep the subscription simple:

```js
const channel = supabase
  .channel('projects-debug')
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
    console.log('Realtime status:', status)
  })
```

Then wait until the status logs `SUBSCRIBED`, insert a new row into the same table, and check whether the callback runs.

If the insert succeeds but no payload is received, I would check:

- whether the table is enabled for Postgres Changes / Realtime
- whether the subscription uses the correct schema and table name
- whether the event filter matches the change being made
- whether the insert happens after the subscription is active
- whether RLS is enabled and the authenticated user can read the changed row
- whether the client disconnects after subscribing

If enabling Postgres Changes for the table fixes it, then the Realtime client code was connecting correctly. The missing piece was that the table was not publishing database changes to Realtime.

If it still does not work, please share the subscription code, sanitized table schema, active RLS policies, Realtime table configuration, and the exact insert/update used for testing. Please remove API keys, JWTs, service-role keys, email addresses, and real user or project IDs before sharing.

## Escalation note

I would not escalate this immediately if the issue is explained by table configuration, event filters, timing, or RLS visibility.

I would escalate only after confirming:

- the client reaches `SUBSCRIBED`
- the table is enabled for Postgres Changes / Realtime
- the schema, table, and event filter are correct
- the database change happens after subscription
- RLS policies allow the subscribed user to read the changed row
- the issue is reproducible with a minimal project or script

For escalation, I would include:

- sanitized client code
- subscription status logs
- sanitized table schema
- active RLS policies
- Realtime table configuration
- exact insert/update SQL used for testing
- expected payload
- actual behavior
- browser/runtime environment
- `@supabase/supabase-js` version
- relevant timestamps and request IDs if available
- a minimal reproduction if possible

## Documentation or product improvement idea

Realtime issues are confusing because `SUBSCRIBED` can make users think the whole setup is working.

A useful note near Realtime/Postgres Changes setup could say:

> `SUBSCRIBED` means the client joined the Realtime channel. If no database events arrive, check whether the table is enabled for Postgres Changes, whether the event filter matches, whether the change happened after subscription, and whether RLS allows the current user to read the changed row.

This would help users debug the difference between connection success and event delivery.

## What I learned

The important detail is that `SUBSCRIBED` only confirms the client joined the Realtime channel. It does not prove that a matching database change will be delivered.

In this reproduction, the client connection worked and the insert succeeded, but no payload was delivered until the table was enabled for Postgres Changes / Realtime.

The useful debugging split was:

- If the client never reaches `SUBSCRIBED`, debug the connection first.
- If the client reaches `SUBSCRIBED` but receives no payload, check table configuration, event filters, timing of the database change, and RLS visibility.
