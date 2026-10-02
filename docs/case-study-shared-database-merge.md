# Case study: merging two production apps onto one database with one login

*Written by Robin Hmaidan. September 2026. Kept deliberately general: no internal identifiers, schema names or data. Snippets are generic illustrations.*

## Context

The client runs two products on two subdomains:

- **esthéJob** (esthejob.fr), the marketplace I lead. Live, with real users signing up daily.
- **esthéJob Cowork** (cowork.esthejob.fr), a booking and subscription SaaS for beauty coworking spaces, built separately on its own Supabase project.

Many users belong to both worlds: a salon that rents out rooms also recruits replacements. Two projects meant two accounts, two passwords and two copies of the same person. The goal set on 24 September 2026:

> **One database and one login. Separate domains, separate products.** Only accounts are shared.

The hard constraint: **the marketplace was live and its logic was off-limits during the move.** Everything had to adapt on the Cowork side, and nothing could change for marketplace users.

## Options considered

| Option | Verdict |
|---|---|
| Keep two projects and sync accounts between them | Rejected: two sources of truth for identity, sync bugs, and passwords still diverge. |
| Merge Cowork tables into the marketplace's main schema | Rejected: both apps had tables with the same names and different meanings, and mixing them blurs ownership. |
| **Move Cowork into its own Postgres schema inside the marketplace's project, sharing Supabase Auth** | **Chosen.** One `auth.users`, clean separation of tables, policies and functions, and the Cowork code only needs to point its client at its schema. |

## Plan

### 1. Read-only inventory of both sides

Before writing anything, I inventoried both projects with read-only sessions: schemas, tables, triggers on the auth tables, storage buckets and policies, extensions, scheduled jobs, auth settings and email templates. I also searched both codebases for every place that touches auth, storage or cross-schema names.

That inventory turned up the real blockers. Generically, they were:

- **Sign-up side effects.** The marketplace creates its own profile row when an account is created. A Cowork sign-up landing in the shared Auth must not silently become a marketplace profile with the wrong role.
- **Auth emails.** One Auth project means one set of email templates and redirect URLs, all tuned for the marketplace. Changing them was off-limits.
- **Deletion paths.** Deleting an account on one product must not cascade into data the other product depends on.
- **Name clashes and search paths.** Functions and policies that assumed the default schema had to be pinned explicitly.
- **Users who already existed on both sides** under the same email.

### 2. Design decisions

- **Own schema for Cowork.** All Cowork tables, functions, triggers and policies live in a dedicated schema. Every `SECURITY DEFINER` function pins its `search_path`, so nothing resolves to the wrong table by accident. The Cowork data client is configured with its schema; the marketplace code needed no change.
- **Cowork sends its own auth emails.** Instead of relying on the shared Auth templates, Cowork generates sign-up, recovery and email-change links server-side and verifies them on its own callback. Marketplace emails stay exactly as they were.
- **Same account, separate sessions.** A user has one identity and one password, but signs in on each domain separately. That avoided cross-domain cookie changes on the live marketplace.
- **Foreign keys as safety rails.** Cowork records reference their owning account with a restricting foreign key, so deleting that account from the marketplace side fails loudly instead of cascading into Cowork data.
- **Namespaced storage.** Cowork files moved into their own buckets with their own policies.

A generic sketch of the schema-pinning pattern:

```sql
create schema if not exists app_two;

create function app_two.do_something(p_id uuid)
returns void
language plpgsql
security definer
set search_path = app_two, pg_temp   -- never resolve names via the caller's path
as $$
begin
  -- ...
end;
$$;

revoke all on function app_two.do_something(uuid) from public;
grant execute on function app_two.do_something(uuid) to authenticated;
```

### 3. Rehearsal before cutover

A cloud staging project was not available for this, so I built the staging environment locally: a Postgres instance rebuilt from the marketplace's migrations in deployment order, then aligned with the production catalog using read-only facts until a fingerprint of the schema matched production. On top of it I replayed every Cowork migration into the new schema.

Then I rehearsed the full move on that copy:

- the schema migration;
- a **data-copy script** with a dry-run mode, which reported exactly which accounts would be created, which would be matched to an existing account by email, and how many rows and files would move;
- a **rollback script**, run for real on the rehearsal database, to prove the marketplace could be returned to its previous state;
- a set of automated checks: every table present, every function pinned, every policy in place, deletion paths behaving as designed.

The copy and rollback scripts refused to write anything unless an explicit "writes authorized" flag was set, so a mistyped command could not touch production.

Rehearsal surfaced issues that would have hurt in production, for example a storage protection mechanism that blocked the rollback path, and a pending-account flow that had to change before accounts became shared. All were fixed before the real run.

### 4. Cutover

The production move happened on 29 September 2026 in a quiet window, after a fresh verified backup and a final pre-flight against both live projects. The steps followed the rehearsed runbook: apply the schema, copy the data and files, expose the new schema to the API, switch the Cowork deployment's environment, deploy.

### 5. Verification

"It loads" is not verification. After the switch:

- **Content hashes of every Cowork table**, old project versus new, matched, apart from the columns that were intentionally remapped to shared account IDs.
- Users signed in on Cowork with their copied accounts.
- All scheduled jobs ran on the new database over the following days.
- Cowork's production smoke suite passed.
- **New marketplace sign-ups kept arriving and confirming normally** the same morning, which showed the marketplace was unaffected.

## After the merge

Sharing accounts creates new cross-product rules, and I handled them as follow-ups:

- **Email changes stay consistent.** Changing the email of the shared account now updates the matching marketplace profile automatically, through a narrowly scoped database trigger.
- **Account linking is a trust boundary.** Any "link a profile to an account by email" path now requires a verified email and only links profiles that are not already linked.
- **Deletion guards.** Deleting an account that still owns data in the other product is blocked rather than cascaded.

## Lessons

1. **Inventory before design.** The blockers were all visible in a read-only pass of both databases and codebases. None were visible from the architecture diagram.
2. **If you can't get a staging project, build one that you can prove matches production.** A schema fingerprint against the live catalog turned "close enough" into "identical".
3. **Script the rollback and run it.** A rollback that has never run is a hope, not a plan.
4. **Make dangerous scripts refuse by default.** Dry run first, explicit flag to write.
5. **Verify with data, not screenshots.** Table hashes, real sign-ins, scheduled jobs and the other product's sign-ups told me the move worked.
6. **Shared identity changes your threat model.** Every place one product trusts an email or an account ID from the other deserves a second look.
