# Supabase Support Engineer Casebook

I created this casebook as proof-of-work for the open Supabase Support Engineer role (EMEA).

The goal is to show how I investigate developer support issues: reproducing problems, isolating root causes, explaining fixes clearly, and identifying when something should be escalated to engineering or improved in documentation.

## Cases

### 1. Auth + RLS: authenticated user sees no rows

Area: Supabase Auth, Postgres Row Level Security  
Problem: A user is logged in, but their query returns an empty array  
Shows: RLS debugging, `auth.uid()`, policy reasoning, customer-facing explanation

[Read case](cases/01-auth-rls-empty-results.md)

### 2. Realtime: subscription connects but receives no events

Area: Supabase Realtime, Postgres changes, RLS  
Problem: A subscription appears connected, but no INSERT/UPDATE/DELETE events arrive  
Shows: structured debugging, publication/RLS checks, client-side troubleshooting

[Read case](cases/02-realtime-no-events.md)

### 3. Postgres/RLS performance: slow queries after enabling policies

Area: Postgres, RLS, indexing, query performance  
Problem: Queries become slow when RLS policies check non-indexed columns  
Shows: performance diagnosis, index reasoning, safe customer guidance

[Read case](cases/03-postgres-rls-performance.md)

## Case format

Each case includes:

- customer problem
- impact
- initial questions
- reproduction
- investigation
- root cause
- solution or workaround
- customer-facing response
- escalation note
- documentation/product improvement idea
