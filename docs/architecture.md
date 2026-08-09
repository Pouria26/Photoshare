# Architecture — Photoshare App Backend

## 1. Domain model

This is a **group photo-sharing app** (think: private family/friend photo
albums, shared per group, with social features layered on top).

### Core entity hierarchy
```
users (shadow of auth.users)
 └── groups (owned by a user)
      └── group_members (role: owner | admin | member)
      └── albums (belongs to a group)
           └── photos (belongs to an album AND denormalized to its group)
                ├── comments (threaded 1 level via parent_id)
                ├── likes
                ├── photo_tags (manual / ai / auto sourced)
                └── photo_permissions (per-user view/download overrides)
      └── album_permissions (per-user view overrides, for group_only albums)
      └── group_invitations (token-based, friend-gated — see below)
      └── group_join_requests (for groups with requires_approval = true)
      └── group_audit_log (role changes, ownership transfers)
```

### Social layer (cross-cutting, not group-scoped)
```
friendships (requester/addressee, status: pending|accepted|declined|blocked)
notifications (typed: like, comment, group_invite, mention, friend_request, ...)
activities (a generic activity-feed/audit table keyed by entity_type+entity_id)
user_settings (1:1 with users — theme, notification toggles, profile visibility)
```

### Notable design decisions
- **Soft deletes** everywhere that matters: `users`, `groups`, `albums`,
  `photos`, `comments` all have `deleted_at` and RLS/queries filter on it
  rather than hard-deleting.
- **Denormalized counters**: `groups.member_count/album_count/photo_count`,
  `albums.photo_count`, `photos.like_count/comment_count`,
  `comments.reply_count` are all maintained by triggers (see
  `05_triggers.sql`), not computed on read. If you add a new way to insert/
  delete rows into `group_members`, `albums`, `photos`, `likes`, or
  `comments`, you MUST keep these triggers in sync or add new ones.
- **Photos store two full URL/path pairs** (`display_url`/`display_path` and
  `thumbnail_url`/`thumbnail_path`) rather than deriving a thumbnail
  dynamically — introduced by the `photos_two_url_storage_model` migration.
  Any code still referencing a single `image_url` column (see
  `get_photo_feed` below) is stale.
- **Invitations require an existing friendship.** `create_group_invitation()`
  raises an exception if the inviter and invitee are not already
  `friendships.status = 'accepted'`. Group invites are friend-gated, not
  open — this is a deliberate anti-spam / trust design choice.
- **Membership role changes never go through direct UPDATEs.** The
  `group_members` table has `members_update` policy with `USING (false)
  WITH CHECK (false)` — i.e. **all direct UPDATEs are blocked by RLS,
  unconditionally**. Role changes MUST go through the `update_member_role()`
  SECURITY DEFINER RPC, which enforces the role hierarchy (owners can
  promote members to admin and demote admins; **admins can only manage
  regular members** — never other admins or the owner; the owner role can
  only change via `transfer_group_ownership()`), writes an audit log entry.
- **The role hierarchy is enforced at the RLS layer too, not just in the
  RPCs** — a direct DELETE/INSERT against `group_members` cannot escalate
  privileges:
  - `members_delete`: owners may remove anyone except the last owner;
    **admins may remove ONLY `role = 'member'` rows** (an admin can never
    delete another admin or the owner); self-removal is blocked while the
    remover is the last owner (no ownerless groups).
  - `members_insert`: a user may self-join only as `member`; the recorded
    `groups.owner_id` may register themselves as owner (group-creation
    flow); owners may add members/admins but never a second owner directly;
    admins may add members only. Invitation/approval flows use SECURITY
    DEFINER RPCs (`accept_group_invitation`, `approve_group_join_request`)
    and are unaffected.
- **Ownership transfer is a dedicated RPC** (`transfer_group_ownership`),
  not a role update — it demotes the old owner to admin, promotes the new
  owner, logs it, and asserts exactly one owner remains afterward.

## 2. Permission model

Almost every RLS policy on a content table (`albums`, `photos`, `comments`,
`likes`, `photo_tags`, `photo_permissions`, `album_permissions`) is built on
top of a small set of **STABLE SECURITY DEFINER** predicate functions defined
in `04_functions.sql`:

- `check_is_group_member(group_id, user_id)`
- `check_is_group_admin(group_id, user_id)` — role in (owner, admin)
- `check_is_group_owner(group_id, user_id)`
- `check_is_group_public(group_id)`
- `check_can_view_album(album_id, user_id)`
- `check_can_manage_album(album_id, user_id)` — creator or group admin
- `check_can_access_photo(photo_id, user_id)`
- `check_can_manage_photo(photo_id, user_id)` — uploader or group admin
- `check_can_download_photo(photo_id, user_id)`
- `are_friends(user1, user2)`

Because these are SECURITY DEFINER, they bypass RLS internally (so a
sub-select inside them doesn't recursively re-trigger the calling table's
own policy), while still being safe to expose to any authenticated role,
since they only ever return a boolean and take explicit user/row IDs as
arguments — they don't leak row contents.

**Visibility cascades group → album → photo**: a photo is visible if its
album is visible (public album + group member, or created by you, or
explicit `album_permissions` grant) AND the photo itself isn't individually
gated by `photo_permissions`. Album visibility itself depends on group
visibility/membership. This three-level cascade is intentional and lets a
group admin restrict a specific album or even a specific photo without
changing the group's overall visibility.

**One RESTRICTIVE policy exists**: `restrict_private_albums` on `albums`.
Postgres ANDs all RESTRICTIVE policies together with the OR of all
PERMISSIVE policies for the same command. In practice this means: no matter
what the permissive `albums_select` policy would otherwise allow, a
`private`-visibility album is NEVER visible to anyone except its creator.
This is a deliberate defense-in-depth guard — keep it if you rebuild this,
since it's easy to accidentally lose the intent of "private really means
private" while iterating on the (more complex) permissive policy alone.
(The migration history shows several iterations getting the permissive
policy right for public/group_only albums — `restrict_private_albums` was
seemingly kept unchanged throughout as the safety net.)

## 3. RPCs (business logic exposed via `/rest/v1/rpc/*`)

Grouped by purpose:

**Group membership & invitations**
- `create_group_invitation`, `accept_group_invitation`,
  `decline_group_invitation` — friend-gated, token-based invite flow
- `approve_group_join_request` — for `requires_approval` groups
- `update_member_role`, `transfer_group_ownership` — admin actions with
  hierarchy enforcement + audit logging. `update_member_role` rejects
  admin→admin and any→owner role writes; ownership changes go exclusively
  through `transfer_group_ownership`

**Photos / albums**
- `add_photo_tag` — upsert-style tagging (manual/ai/auto sourced)
- `set_album_permissions`, `set_photo_permissions` — bulk-replace the
  allow-list for a `group_only` album or a restricted photo
- `get_album_permissions` — who can see a given album and their current flag
- `get_albums_with_visible_counts` — per-group album list with a
  *permission-aware* photo count (deliberately relies on the caller's own
  RLS context rather than re-implementing the visibility logic — see the
  inline comment preserved in `04_functions.sql`)

**Feed / search / social**
- `get_photo_feed`, `get_user_groups` — personalized feed (⚠️ see Known
  Issues — this one is broken as captured)
- `search_public_groups`, `search_users_with_friendship`,
  `get_mentionable_users` — typeahead/search endpoints
- `get_user_friends`, `get_pending_requests`, `get_friendship_status`,
  `are_friends` — friend graph queries

**Notifications**
- `create_notification` — safe enum-cast wrapper (falls back to `'system'`
  type on bad input rather than erroring)

## 4. Known issues / inconsistencies to resolve (found during inspection)

These are reproduced faithfully in the SQL files (per the instruction to
document rather than silently "fix" what was found), but you should decide
what to do with each before shipping a new build:

1. **`get_photo_feed` references `p.image_url`, which does not exist.**
   The `photos` table was migrated to `display_url`/`thumbnail_url` by
   `photos_two_url_storage_model`, but this function was never updated. It
   will throw `column p.image_url does not exist` if called. Likely dead
   code — the app has probably moved to querying `photos` directly (e.g.
   paired with `get_albums_with_visible_counts`).

2. **Duplicate function overloads never cleaned up:**
   - `get_mentionable_users(uuid, text)` (old, `plpgsql`, references
     `groups.is_private` — a column that doesn't exist on the current
     `groups` table, so this overload is also broken) vs.
     `get_mentionable_users(uuid, text, uuid)` (current, `sql`, uses
     `groups.visibility`). PostgREST will pick an overload based on the
     arguments the client actually sends — if any client code still calls
     the 2-arg version, it will error.
   - `search_users_with_friendship(uuid, text)` vs.
     `(uuid, text, uuid, integer)` — both work, just redundant; the 3-arg
     version is a slightly refined superset (adds 'requested'/'received'
     framing for pending requests instead of a flat 'pending').
   Recommendation: drop the stale overloads in a fresh build once you
   confirm which one the client calls.

3. **`public_profiles` view is `SECURITY DEFINER`** (flagged as an
   ERROR-level finding by Supabase's own linter). This means the view
   ignores the querying user's RLS and always runs as the view owner. Given
   the view only exposes non-sensitive public profile fields and already
   filters `deleted_at IS NULL`, the practical risk is limited, but it's
   inconsistent with the rest of the schema's RLS-first design. Recommend
   switching to `security_invoker = true` (see commented-out alternative in
   `07_views.sql`).

4. **Storage bucket coverage is incomplete relative to the schema.** The
   schema has `*_path`/`*_url` column pairs for **avatars**
   (`users.avatar_path`), **group covers** (`groups.cover_path`), **album
   covers** (`albums.cover_path`), and **photos**
   (`photos.display_path`/`thumbnail_path`) — but only ONE storage bucket
   (`group-covers`) exists. Either the other buckets exist under a
   credential/project this inspection didn't have access to, or (more
   likely, given the very recent `photos_two_url_storage_model` migration
   date) the app is mid-migration from external URLs to Supabase Storage,
   starting with group covers. Budget for creating `avatars`, `album-covers`,
   and `photos` buckets with appropriate RLS if you want full parity.

5. **`extension_in_public` warning**: `pg_trgm` is installed directly into
   the `public` schema rather than a dedicated `extensions` schema. Harmless
   functionally, but Supabase's linter flags it because it pollutes the
   `public` schema's function/operator namespace. `01_extensions_and_types.sql`
   reproduces the source project's behavior but comments the recommended fix.

6. **`group-covers` bucket is public with an unscoped SELECT policy** —
   anyone can `list()` every file in the bucket, not just fetch a known URL.
   Fine if group cover images are meant to be fully public and enumerable;
   otherwise scope the policy (see `08_storage.sql`).

7. **~50 SECURITY DEFINER functions are callable by `anon`/`authenticated`
   with no additional gating**, per Supabase's security advisor. Most of
   these are read-only predicate helpers (safe) or already do their own
   `auth.uid()` checks inside the function body (safe), but this is worth a
   deliberate pass rather than assuming — the full advisor output with every
   affected function is in `SECURITY_ADVISORIES.md`.

8. **Friend RPCs name the friend's UUID column non-obviously.**
   `get_user_friends` returns the friend's real user UUID under the column
   name `friend_id` (there is **no** `id` column), and `get_pending_requests`
   returns it as `requester_id`. Screens navigating to `/profile/:userId`
   MUST read those keys — reading `['id']` yields `null`, producing
   `/profile/null` and a false "User not found". The Friends list was fixed
   to use `friend_id`; `friend_requests_screen.dart` still reads
   `request['id']` and hits the same bug class (not yet fixed).

## 5. Migration history & what it implies

The 10 migrations found (see README) all cluster on 2026-07-25 through
2026-07-28 — i.e. this project is very young and under active, fast
iteration, with the album-visibility RLS policy in particular being
revisited **four separate times** (`fix_album_visibility_public_group_
membership`, `remove_admin_view_override_for_private_albums`,
`fix_public_album_visibility_for_non_members`,
`fix_albums_select_policy_for_public_autojoin_groups`). This is a strong
signal that "who can see a public group's albums when they haven't
explicitly joined yet" was a genuinely tricky edge case to get right —
worth extra test coverage if you rebuild this. The final state (captured
here) is: a `public` album is visible if the viewer is a group member OR
the group itself is `public` with `requires_approval = false` (i.e.
auto-joinable groups expose their public albums to non-members too, since
joining is a formality).

The last two migrations (`fix_username_case_insensitive_uniqueness_and_
collision_safe_signup`, `sync_is_verified_on_email_confirmation`) show the
sign-up flow being hardened: case-insensitive unique usernames
(`users_username_unique_ci`) plus collision-safe retry logic in
`handle_new_user()`, and syncing `is_verified` from Supabase Auth's own
`email_confirmed_at` via a new `auth.users` trigger.
