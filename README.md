# Breaking Supabase on purpose: a small support lab

I'm an IT infrastructure officer at a public-health programme in Ghana. Most of my support work is for internal staff and field teams, not external developers. I wanted to find out what common Supabase support problems look like from the inside, so I made a free Supabase project, broke it on purpose, and wrote down what each error looked like and how to fix it.

This is my lab notebook from 12 October 2026, cleaned up. Everything was done in the Supabase dashboard (mostly the SQL Editor) and the browser's developer tools. I used an AI assistant as a guide while working through the lab; every query was run by me and the results below are what my project returned. Where I got something wrong, I've left it in.

No project URL, project ref or API keys are included here.

## Setup

- Supabase free plan, one project in Central EU (Frankfurt), smallest compute size
- Three small test tables: `firstlab`, `notes` (one row per user) and `orders` (200,000 rows)
- No app: I used the SQL Editor and simulated signed-in users, and called the REST API from the browser console

## Results at a glance

| # | Problem a customer might report | What I saw | Cause |
|---|---|---|---|
| 1 | "My query returns an empty array" | 0 rows, **no error** | RLS on, no policy |
| 2 | (Nobody reports this one) | An anonymous visitor could **write** to a table | Table created in SQL had RLS off |
| 3 | "Users can see each other's data" / "my data disappeared" | User 2 saw 0 rows; the row was there | Per-user policy working as designed |
| 4 | "My insert fails" | `ERROR 42501: new row violates row-level security policy` | `with check` rejected the row |
| 5 | "This query is slow" | 36.9 ms → 0.129 ms on 200,000 rows | Missing index |
| 6 | "Invalid API key" | HTTP 401, and a different message for a missing key | Wrong or missing `apikey` header |
| 7 | "Failed to fetch" | Request never reached Supabase | Browser Content Security Policy |
| 8 | "Confirmation email goes to localhost" | Site URL default is `http://localhost:3000` | Site URL not set |
| 9 | "Can I make restrictive policies on storage?" | Worked from the SQL Editor in my project | (see below) |

## What I learned

### 1. "Empty array, no error" usually means RLS

I made a table `firstlab` in the dashboard's table editor and added three rows. Then I read it as the `anon` role, the role an anonymous visitor using the public API key gets:

```sql
set role anon;
select * from public.firstlab;
```

I expected 3 rows, because I thought Row Level Security (RLS) was off. I got **0 rows and no error**. The dashboard had already turned RLS on when it created the table, and with RLS on and no policy, a role sees nothing.

You can check with:

```sql
select relrowsecurity from pg_class where oid = 'public.firstlab'::regclass;
```

Adding one policy made the rows appear:

```sql
create policy "anon can read firstlab" on public.firstlab
  for select to anon using (true);
```

**Lesson:** an empty result with no error is the classic sign of RLS with a missing or wrong policy. My guess was wrong, and the empty result is what told me.

### 2. A table made in SQL had RLS off, and anyone could write to it

I made a second table, `openlab`, with SQL instead of the dashboard. That one had RLS **off** (`relrowsecurity = false`). As `anon`, I could read it, and this worked too:

```sql
insert into public.openlab (name) values ('written by a stranger');
```

So on that day, in my project, a table created in SQL in the `public` schema could be changed by anyone with the public key. Defaults can change, so check this on your own project rather than relying on my result.

**Lesson:** turn on RLS for every table in a schema the API can see:

```sql
alter table public.openlab enable row level security;
```

### 3. Per-user data: user 2 can't see user 1's note

I made a `notes` table where each row belongs to a user, with RLS and two policies:

```sql
create table public.notes (
  id bigint generated always as identity primary key,
  user_id uuid not null default auth.uid(),
  body text,
  created_at timestamptz default now()
);
alter table public.notes enable row level security;

create policy "read own notes" on public.notes
  for select to authenticated using (auth.uid() = user_id);
create policy "insert own notes" on public.notes
  for insert to authenticated with check (auth.uid() = user_id);
```

To test without building an app, I pretended to be a signed-in user in the SQL Editor by setting the role and the JWT claims that `auth.uid()` reads:

```sql
set role authenticated;
select set_config('request.jwt.claims',
  '{"sub":"22222222-2222-2222-2222-222222222222","role":"authenticated"}', false);
select count(*) from public.notes;
```

(This is a testing shortcut. A real sign-in goes through Supabase Auth and gives the app a real token.)

| Who | Rows seen |
|---|---|
| User 1 (`1111...`), after inserting a note | 1, with `user_id` filled in automatically by `default auth.uid()` |
| User 2 (`2222...`) | **0** |
| Admin (`reset role`) | 1 |

The row exists; user 2 just can't see it. From user 2's side that looks exactly like problem 1.

### 4. The insert error: 42501

Still as user 2, I tried to insert a note that claimed to belong to user 1:

```
ERROR: 42501: new row violates row-level security policy for table "notes"
```

The `with check` part of the insert policy rejected it.

**Lesson:** RLS fails in two different ways, and customers describe them differently:

| Blocked by | What the customer sees | What they usually say |
|---|---|---|
| `using` (reads) | Empty result, no error | "My data disappeared" |
| `with check` (writes) | Error 42501 | "My insert fails" |

### 5. Slow query: reading `EXPLAIN ANALYZE`

I made an `orders` table with 200,000 rows and random customer ids, then looked up one customer:

```sql
explain analyze
select * from public.orders where customer_id = 1234;
```

| | Plan | Rows read to find 38 | Execution time |
|---|---|---|---|
| No index | `Seq Scan` | all 200,000 (`Rows Removed by Filter: 199962`) | **36.9 ms** |
| After `create index orders_customer_id_idx on public.orders (customer_id);` | `Bitmap Index Scan` | 38 heap blocks | **0.129 ms** |

**Lesson:** a `Seq Scan` with a big "Rows Removed by Filter" on a large table means Postgres read everything to find a few rows. An index on the filtered column fixed it. On my small table the saving was milliseconds; on millions of rows the same plan would cost seconds. Indexes also take space and slow down writes, so they're for columns you filter or join on often.

### 6. API key errors: missing vs wrong

I called the REST API for `firstlab` from the browser.

| Request | Response |
|---|---|
| No key at all | `` {"message":"No API key found in request","hint":"No `apikey` request header or url param was found."} `` |
| `apikey: wrong` | **HTTP 401** `` {"message":"Invalid API key","hint":"Double check your Supabase `anon` or `service_role` API key."} `` |

Things I'd check for a customer: the key copied with a space or line break, a key from a different project, an old key after rotating keys, or the client not sending the header at all.

**Lesson:** a 401 "Invalid API key" is about the key, not RLS. With a valid key, a blocked policy gives an empty result or 42501, never this message.

### 7. "Failed to fetch" that wasn't Supabase's fault

My first attempt at problem 6 failed with `TypeError: Failed to fetch`. The console said the page "violates the following Content Security Policy directive: connect-src ...". I had run the code on a Chrome internal page, and that page isn't allowed to connect anywhere. The request never left the browser.

**Lesson:** "Failed to fetch" can come from the page itself (Content Security Policy or CORS). Read the full console message before blaming the API.

### 8. Confirmation emails that go to localhost

In Authentication → URL Configuration, a new project had:

- **Site URL:** `http://localhost:3000`
- **Redirect URLs:** empty

Auth emails (confirm sign-up, reset password) send people to the Site URL unless the app asks for another allowed address. So after a deploy, users can click the email link and land on a dead localhost page. The fix is to set the Site URL to the real app address and add any other addresses (preview, local dev) to Redirect URLs.

I only looked at the defaults. I didn't send a real auth email, so I haven't reproduced the redirect myself.

### 9. Restrictive policies on `storage.objects`

While reading a community question, I saw people disagreeing about whether you can create an `AS RESTRICTIVE` policy on `storage.objects` from the SQL Editor, since that table belongs to an internal Storage role. I tested it in my project:

```sql
select current_user;   -- postgres

create policy "test restrictive" on storage.objects
  as restrictive for select to authenticated using (true);

select policyname, permissive, roles, cmd
from pg_policies
where schemaname = 'storage' and tablename = 'objects';
-- test restrictive | RESTRICTIVE | {authenticated} | SELECT

drop policy "test restrictive" on storage.objects;
```

It worked. That's one free project on one date; I didn't test the dashboard's policy editor, CLI migrations, or whether the policy was enforced on real uploads.

## My own mistakes (also useful)

- **`ERROR 42P01: table "openlab" does not exist`.** My script started with `drop table public.openlab;`, but I'd already dropped it. The SQL Editor stopped at the first error, so **nothing after that line ran**, including the `create table` I cared about. `drop table if exists` fixed it. A failing first line can hide that the rest never ran.
- **`ERROR 42704: policy ... does not exist`.** I thought my first storage policy test had worked, but when I tried to drop the policy it wasn't there. I'd run the create and the drop together without noticing. Re-running one statement at a time, and checking `pg_policies` in between, gave a real answer.
- **The SQL Editor only shows one result per run.** When I ran several `select`s together, I only saw one of them and missed the one I needed. Now I run checks on their own, or put the one I need last.

## If someone came to me with one of these

1. Find out which layer the error comes from first: the browser, the API key, RLS, or the query itself. The error text usually says.
2. Empty result, no error → check RLS and policies for that role.
3. 42501 → check the `with check` part of the insert/update policy.
4. 401 "Invalid API key" → check the key, not the policies.
5. Slow → ask for `explain analyze` and look for a `Seq Scan` on a big table.
6. Email link goes to localhost → Site URL and Redirect URLs.
7. After any change, check with a query instead of trusting "Success".

## What I didn't cover

- A real app with real sign-ins (I simulated users in SQL)
- Edge Functions, Realtime, and Storage uploads
- Supabase CLI and migrations
- Sending real auth emails

## Clean-up

This was a free test project with only made-up data. I deleted it on 12 October 2026, after publishing this write-up.
