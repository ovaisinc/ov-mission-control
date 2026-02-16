# Supabase Database Setup (Production Hardening)

This project now scopes task data by `user_id` and is structured for future Supabase Auth.

## 1) Tasks table requirements

> Replace with your exact schema if needed, but these columns are required by app queries.

```sql
alter table public.tasks
  add column if not exists user_id text not null default 'ov';

-- Optional but recommended indexes
create index if not exists idx_tasks_user_done_type on public.tasks (user_id, done, type);
create index if not exists idx_tasks_user_completed_at on public.tasks (user_id, completed_at desc);
```

> Note: `user_id` is currently a text identifier (`ov` fallback). Replace with `auth.uid()` UUID flow when auth is fully enabled.

## 2) Enable RLS

```sql
alter table public.tasks enable row level security;
```

## 3) Required RLS policies

```sql
-- Example RLS policies needed:
CREATE POLICY "Users can only see their own tasks"
ON public.tasks
FOR SELECT
USING (auth.uid()::text = user_id);

CREATE POLICY "Users can only insert their own tasks"
ON public.tasks
FOR INSERT
WITH CHECK (auth.uid()::text = user_id);

CREATE POLICY "Users can only update their own tasks"
ON public.tasks
FOR UPDATE
USING (auth.uid()::text = user_id)
WITH CHECK (auth.uid()::text = user_id);

CREATE POLICY "Users can only delete their own tasks"
ON public.tasks
FOR DELETE
USING (auth.uid()::text = user_id);
```

If you are still running with anon/testing access, keep policy rollout staged and test carefully.

## 4) App auth prep

The frontend currently uses:

- `DEFAULT_USER_ID = 'ov'`
- `sb.auth.getSession()` + `onAuthStateChange`

When auth is implemented:

1. remove hardcoded fallback where appropriate
2. map `currentUserId` directly to `session.user.id`
3. keep `.eq('user_id', currentUserId)` query scoping

## 5) Migration behavior

The migration tool now:

- supports **dry run** (preview, no writes)
- uses **upsert** on `id` (idempotent, no duplicate inserts on re-run)
- tracks **success/failure per record**
- blocks accidental duplicate run while in progress

## 6) Validation checklist

1. Run migration in dry-run mode, then live mode.
2. Re-run live mode: verify no duplicate rows.
3. Complete/restore/edit a task in UI and confirm DB updates.
4. Simulate network error and confirm UI rollback + visible error banner.
5. Test with mismatched user scope under RLS and verify access is denied.
