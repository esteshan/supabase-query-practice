# Supabase Best Practices

> **Purpose**: This document provides comprehensive best practices for Supabase development.
> It is designed for AI review to ensure adherence to standards and can be periodically referenced during development.

---

## Table of Contents

1. [Database Best Practices](#1-database-best-practices)
2. [Row Level Security (RLS)](#2-row-level-security-rls)
3. [Authentication Best Practices](#3-authentication-best-practices)
4. [Storage Best Practices](#4-storage-best-practices)
5. [Realtime Best Practices](#5-realtime-best-practices)
6. [API Best Practices](#6-api-best-practices)
7. [Edge Functions Best Practices](#7-edge-functions-best-practices)
8. [Performance Optimization](#8-performance-optimization)
9. [Security Best Practices](#9-security-best-practices)
10. [Production Readiness Checklist](#10-production-readiness-checklist)

---

## 1. Database Best Practices

### 1.1 Schema Design

- **Use `uuid` for Primary Keys**: Provides better distribution and avoids sequential IDs that can be guessed.

  ```sql
  create table profiles (
    id uuid primary key default gen_random_uuid(),
    -- other columns
  );
  ```

- **Use Foreign Keys**: Enforce referential integrity between tables.

  ```sql
  create table posts (
    id uuid primary key default gen_random_uuid(),
    user_id uuid references auth.users(id) on delete cascade,
    -- other columns
  );
  ```

- **Use `NOT NULL` Constraints**: Ensure data completeness for essential columns.

- **Choose Appropriate Data Types**:
  - Use `text` instead of `varchar` unless you need a specific length limit
  - Use `jsonb` instead of `json` for better performance and indexing
  - Use `timestamptz` for timestamps (timezone-aware)
  - Use `bigint` for auto-incrementing IDs that may grow large

- **Use `generated always as identity`** for auto-incrementing columns:

  ```sql
  create table items (
    id bigint primary key generated always as identity,
    -- other columns
  );
  ```

### 1.2 Naming Conventions

- Use `snake_case` for table and column names
- Use plural names for tables (e.g., `users`, `posts`, `comments`)
- Prefix private/internal schemas with `private` or `internal`

### 1.3 Indexing

- **Index Frequently Queried Columns**: Columns used in `WHERE`, `JOIN`, and `ORDER BY` clauses
- **Primary keys are automatically indexed**
- **Foreign keys should be indexed** for better join performance

  ```sql
  create index idx_posts_user_id on posts (user_id);
  ```

- **Use partial indexes** for frequently filtered subsets:

  ```sql
  create index idx_active_users on users (email)
  where is_active = true;
  ```

- **Use composite indexes** for multi-column filters:

  ```sql
  create index idx_posts_user_date on posts (user_id, created_at desc);
  ```

- **Avoid over-indexing**: Indexes slow down writes (`INSERT`, `UPDATE`, `DELETE`)

### 1.4 Query Best Practices

- **Avoid `SELECT *`**: Only select columns you need

  ```sql
  -- Bad
  select * from users;

  -- Good
  select id, name, email from users;
  ```

- **Use `LIMIT`** for pagination and limiting result sets

- **Batch operations** to reduce network round trips:

  ```sql
  insert into items (name, value) values
    ('item1', 100),
    ('item2', 200),
    ('item3', 300);
  ```

---

## 2. Row Level Security (RLS)

### 2.1 Core Requirements

- **ALWAYS enable RLS on tables in the `public` schema**:

  ```sql
  alter table public.profiles enable row level security;
  ```

- **Create policies for all CRUD operations** as needed (SELECT, INSERT, UPDATE, DELETE)

### 2.2 Policy Best Practices

- **Wrap `auth.uid()` and `auth.jwt()` in `select` statements** for better performance:

  ```sql
  -- Bad (slower)
  create policy "Users can view own data" on profiles
  for select using (auth.uid() = user_id);

  -- Good (faster - uses initPlan optimization)
  create policy "Users can view own data" on profiles
  for select using ((select auth.uid()) = user_id);
  ```

- **Specify the role in policies using `TO`**:

  ```sql
  create policy "Authenticated users can view profiles" on profiles
  for select
  to authenticated
  using (true);
  ```

- **Index columns used in RLS policies**:

  ```sql
  create index idx_profiles_user_id on profiles (user_id);
  ```

### 2.3 Policy Examples

**SELECT Policy**:

```sql
create policy "Users can view their own profile"
on profiles for select
to authenticated
using ((select auth.uid()) = user_id);
```

**INSERT Policy**:

```sql
create policy "Users can create their own profile"
on profiles for insert
to authenticated
with check ((select auth.uid()) = user_id);
```

**UPDATE Policy**:

```sql
create policy "Users can update their own profile"
on profiles for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);
```

**DELETE Policy**:

```sql
create policy "Users can delete their own profile"
on profiles for delete
to authenticated
using ((select auth.uid()) = user_id);
```

### 2.4 Advanced RLS Patterns

- **Use security definer functions** to bypass RLS on join tables:

  ```sql
  create function private.get_user_role(p_user_id uuid)
  returns text
  language sql
  security definer
  set search_path = ''
  as $$
    select role from public.user_roles where user_id = p_user_id;
  $$;
  ```

- **Avoid joining on row data in policies** - rewrite to filter on fixed values:

  ```sql
  -- Bad (slower)
  create policy "Team access" on documents
  using (auth.uid() in (
    select user_id from team_members
    where team_members.team_id = documents.team_id
  ));

  -- Good (faster)
  create policy "Team access" on documents
  using (team_id in (
    select team_id from team_members
    where user_id = (select auth.uid())
  ));
  ```

### 2.5 RLS Performance Tips

1. Add indexes on columns used in policies
2. Use `(select auth.uid())` instead of `auth.uid()`
3. Add explicit filters in queries that match policy conditions
4. Use security definer functions to avoid RLS on lookup tables
5. Minimize joins in policies
6. Always specify the role with `TO`

---

## 3. Authentication Best Practices

### 3.1 Security

- **Never expose the `service_role` key** in client-side code
- **Use the `anon` key** for client-side operations
- **Store API keys in environment variables** - never commit to source control
- **Use HTTPS** for all communications
- **Implement proper redirect URLs** for OAuth providers
- **Consider enabling MFA** for sensitive applications

### 3.2 Session Management

- Store JWTs securely using HTTP-only cookies for web apps (using `@supabase/ssr`)
- Handle token refresh properly
- Implement proper sign-out that clears all session data

### 3.3 User Management

- **Verify email addresses** before granting full access
- **Implement robust password policies**
- **Use `raw_app_meta_data`** for authorization data (cannot be modified by users)
- **Use `raw_user_meta_data`** only for non-sensitive user preferences

### 3.4 Auth Code Examples

**Server-side client creation (Next.js)**:

```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          );
        },
      },
    }
  );
}
```

---

## 4. Storage Best Practices

### 4.1 Bucket Security

- **Use private buckets** for sensitive data
- **Implement storage policies** similar to RLS:

  ```sql
  create policy "Users can upload to own folder"
  on storage.objects for insert
  to authenticated
  with check (
    bucket_id = 'avatars' and
    (storage.foldername(name))[1] = (select auth.uid())::text
  );
  ```

### 4.2 File Management

- **Validate file types and sizes** before upload
- **Use signed URLs** for temporary access to private files
- **Organize files logically** using folder structures
- **Implement file cleanup** for orphaned files

### 4.3 Performance

- **Optimize images/files** before uploading (compress, resize)
- **Use CDN** for frequently accessed public assets
- **Set appropriate cache headers**

---

## 5. Realtime Best Practices

### 5.1 Channel Management

- **Use private channels** in production:

  ```typescript
  const channel = supabase.channel('room:123:messages', {
    config: { private: true },
  });
  ```

- **Follow naming conventions** - use pattern `scope:id:entity`:
  - `room:123:messages` - Messages in room 123
  - `user:456:notifications` - Notifications for user 456

### 5.2 Subscription Management

- **Always unsubscribe** when done:

  ```typescript
  // React example
  useEffect(() => {
    const channel = supabase.channel('room:123:messages');
    // ... setup listeners
    return () => {
      supabase.removeChannel(channel);
    };
  }, []);
  ```

- **Subscribe only to necessary changes** - avoid subscribing to entire tables
- **Use filters** to limit the data received

### 5.3 RLS for Realtime

```sql
-- Allow authenticated users to receive broadcasts
create policy "authenticated_users_can_receive" on realtime.messages
for select to authenticated using (true);

-- Allow authenticated users to send broadcasts
create policy "authenticated_users_can_send" on realtime.messages
for insert to authenticated with check (true);
```

---

## 6. API Best Practices

### 6.1 Security

- **Always enable RLS** on tables exposed via the API
- **Use the `anon` key** for public/unauthenticated requests
- **Never expose `service_role` key** in client code
- **Implement rate limiting** using pre-request functions if needed

### 6.2 Query Patterns

- **Add filters** to every query (don't rely solely on RLS):

  ```typescript
  // Bad
  const { data } = await supabase.from('posts').select();

  // Good
  const { data } = await supabase
    .from('posts')
    .select()
    .eq('user_id', userId);
  ```

- **Select only needed columns**:

  ```typescript
  const { data } = await supabase
    .from('profiles')
    .select('id, name, avatar_url')
    .eq('id', userId)
    .single();
  ```

### 6.3 Error Handling

- Always check for errors in responses
- Handle specific error codes appropriately
- Log errors for debugging (without exposing sensitive data)

```typescript
const { data, error } = await supabase
  .from('profiles')
  .select()
  .eq('id', userId)
  .single();

if (error) {
  console.error('Error fetching profile:', error.message);
  // Handle error appropriately
}
```

---

## 7. Edge Functions Best Practices

### 7.1 Security

- **Verify JWTs** in functions that require authentication
- **Use environment variables** for secrets
- **Never hardcode secrets** in function code

### 7.2 Performance

- **Keep functions small and focused**
- **Use connection pooling** for database connections
- **Implement proper error handling**
- **Set appropriate timeouts**

### 7.3 Example Function Structure

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  try {
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_ANON_KEY')!,
      {
        global: {
          headers: { Authorization: req.headers.get('Authorization')! },
        },
      }
    );

    // Get user from JWT
    const {
      data: { user },
      error: authError,
    } = await supabase.auth.getUser();

    if (authError || !user) {
      return new Response(JSON.stringify({ error: 'Unauthorized' }), {
        status: 401,
        headers: { 'Content-Type': 'application/json' },
      });
    }

    // Your function logic here

    return new Response(JSON.stringify({ success: true }), {
      headers: { 'Content-Type': 'application/json' },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { 'Content-Type': 'application/json' },
    });
  }
});
```

---

## 8. Performance Optimization

### 8.1 Query Analysis

- **Use `EXPLAIN ANALYZE`** to understand query performance:

  ```sql
  explain analyze select * from users where email = 'test@example.com';
  ```

- **Monitor cache hit rates** - aim for 99%+:

  ```sql
  select
    'index hit rate' as name,
    (sum(idx_blks_hit)) / nullif(sum(idx_blks_hit + idx_blks_read), 0) as ratio
  from pg_statio_user_indexes
  union all
  select
    'table hit rate' as name,
    sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) as ratio
  from pg_statio_user_tables;
  ```

### 8.2 Index Optimization

- **Check unused indexes** and remove them:

  ```bash
  supabase inspect db unused-indexes
  ```

- **Check index usage**:

  ```sql
  select
    relname,
    100 * idx_scan / (seq_scan + idx_scan) as percent_of_times_index_used,
    n_live_tup as rows_in_table
  from pg_stat_user_tables
  where seq_scan + idx_scan > 0
  order by n_live_tup desc;
  ```

### 8.3 Connection Management

- **Use connection pooling** via Supavisor
- **Use `transaction` mode** for serverless environments
- **Close connections properly** in your application

### 8.4 Vacuuming

- **Monitor vacuum stats** regularly:

  ```bash
  supabase inspect db vacuum-stats
  ```

- **Run `ANALYZE`** after large data changes:

  ```sql
  analyze table_name;
  ```

---

## 9. Security Best Practices

### 9.1 API Keys

- **Never expose `service_role` key** in client-side code
- **Use environment variables** for all keys
- **Rotate keys periodically** (when compromised or as policy)

### 9.2 Data Security

- **Enable RLS on all public schema tables**
- **Use `private` schema** for internal functions and data
- **Sanitize user input** - never use `dangerouslySetInnerHTML` without sanitization
- **Encrypt sensitive data** at the application layer if needed

### 9.3 Access Control

- **Implement least privilege** - grant only necessary permissions
- **Review RLS policies regularly**
- **Monitor Security Advisor** recommendations in the dashboard

### 9.4 SSL/TLS

- **Enforce SSL** for all database connections
- **Use HTTPS** for all API communications

---

## 10. Production Readiness Checklist

### 10.1 Security Checklist

- [ ] RLS enabled on all tables in `public` schema
- [ ] RLS policies tested and verified
- [ ] `service_role` key not exposed in client code
- [ ] API keys stored in environment variables
- [ ] SSL enforcement enabled
- [ ] Security Advisor recommendations addressed
- [ ] MFA enabled for dashboard access (recommended)

### 10.2 Performance Checklist

- [ ] Indexes created for frequently queried columns
- [ ] Indexes created for foreign key columns
- [ ] Indexes created for columns used in RLS policies
- [ ] Unused indexes removed
- [ ] Query performance analyzed with `EXPLAIN ANALYZE`
- [ ] Cache hit rate above 99%
- [ ] Connection pooling configured

### 10.3 Reliability Checklist

- [ ] Backups enabled and tested
- [ ] Point-in-Time Recovery enabled (if needed)
- [ ] Monitoring and alerting set up
- [ ] Error handling implemented throughout application
- [ ] Rate limiting configured (if needed)

### 10.4 Development Workflow

- [ ] Database migrations managed via CLI
- [ ] Separate environments for development/staging/production
- [ ] TypeScript types generated from database schema
- [ ] Testing implemented for RLS policies

---

## Quick Reference Commands

### Supabase CLI Commands

```bash
# Link to project
supabase link --project-ref <project-id>

# Generate TypeScript types
supabase gen types typescript --linked > database.types.ts

# Run database migrations
supabase db push

# Inspect database
supabase inspect db bloat
supabase inspect db unused-indexes
supabase inspect db cache-hit
supabase inspect db index-usage
```

### Common SQL Commands

```sql
-- Enable RLS
alter table public.table_name enable row level security;

-- Create index
create index idx_name on table_name (column_name);

-- Analyze table
analyze table_name;

-- Check table sizes
select pg_size_pretty(pg_total_relation_size('table_name'));

-- Reset pg_stat_statements
select pg_stat_statements_reset();
```

---

## Resources

- [Supabase Documentation](https://supabase.com/docs)
- [Row Level Security Guide](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [RLS Performance Discussion](https://github.com/orgs/supabase/discussions/14576)
- [Production Checklist](https://supabase.com/docs/guides/platform/going-into-prod)
- [Shared Responsibility Model](https://supabase.com/docs/guides/deployment/shared-responsibility-model)

---

*Last updated: November 2024*
*Based on Supabase official documentation*

