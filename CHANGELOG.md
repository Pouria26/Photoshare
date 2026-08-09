# Changelog

All notable changes to this project are documented in this file.

## [0.0.5-alpha] — 2026-08-09

The **offline-first release**. PhotoShare now caches groups, albums, photos, notifications, and your profile locally — everything opens instantly, works without a connection, and re-syncs automatically when you're back online. This release also hardens the group permission hierarchy (enforced in both the Supabase backend and the app), reworks the Friends and group-members experience, and fixes a batch of stability issues.

### Group members — permission hierarchy hardening (backend + frontend)
- **Critical fix**: an admin could remove another admin. The `members_delete` RLS policy allowed admins to delete any row where `role <> 'owner'` — it now restricts admins to `role = 'member'` rows only. Owners may still remove anyone (except the last remaining owner), and self-removal is also blocked while the remover is the last owner, preventing ownerless groups.
- **`members_insert` RLS** now enforces the role hierarchy on direct inserts: users may self-join only as `member`, the recorded `groups.owner_id` may register themselves as owner (group creation flow), owners may add members/admins (never a second owner directly), and admins may add members only. Privilege escalation via a direct API insert is no longer possible.
- **`update_member_role` RPC** now blocks admins from managing anything but regular members (no demoting other admins, no touching owners) and blocks any direct change to the owner role — ownership changes only via `transfer_group_ownership()`. Existing owner→member promote, owner→admin demote, and admin→member remove flows are unaffected.
- The members screen overflow menu now derives its options from a **permission matrix** that mirrors the backend exactly — an action that would fail server-side is never shown (e.g. no menu at all for an admin viewing another admin or the owner).
- Material 3 polish on the members screen: premium member cards (larger avatar with ring, improved spacing/typography, subtle press-scale animation), role sections with member counts ("Owners (3)"), redesigned role chips (Owner = warm gold, Admin = blue, Member = neutral), an animated search field with clear button, an outlined "You" badge instead of "(You)", and a tonal Invite button. Includes accessibility semantics and a spinner that only shows on first load (background refreshes keep the list visible and preserve scroll position).

### Group detail — banner state model
- Fixed incorrect guest banner on the group detail screen: the "You're viewing this public group as a guest" banner was shown for *any* non-member, including users invited to a **private** group. Banner logic is now driven by a single resolved `_GroupRelation` state (member / invited / requested / notMember / checking) instead of a pile of independent booleans, so invalid visibility × membership combinations can't render:
  - Guest banner shows ONLY for public auto-join groups where the viewer is not a member and has no pending invitation.
  - Private groups with a pending invitation now show a visibility-aware invitation banner ("You've been invited to join this private group.") with the existing Accept/Decline actions.
  - Invited users are no longer blocked by the approval wall — the invitation takes precedence (they were already approved by being invited).

### Stability & layout
- Fixed the Flutter assertion `'child != this': Tried to make a child into a parent of itself.` on the group members screen: the search field previously attached the same `FocusNode` to both an outer `Focus` wrapper and the `TextField`'s internal `Focus`. The wrapper was removed and the node is now attached once and observed via a listener.
- Fixed bottom overflow on the Friend Search and Friend Requests screens: empty states are now scrollable (`LayoutBuilder` + `SingleChildScrollView` + `ConstrainedBox`), so they center normally when there's room and scroll instead of overflowing when the keyboard is up or on small screens.

### Friends
- Fixed "User not found" when opening a friend's public profile: the Friends screen passed a nonexistent `id` key from the `get_user_friends` RPC result — it now passes `friend_id`, the friend's actual user UUID.
- The entire friend row (avatar, name, username) now opens the friend's public profile; the three-dot menu remains independent and does not trigger navigation.
- Removing a friend now asks for confirmation ("Remove friend?"), removes the user from the list immediately on success, and shows success/error SnackBars.
- Added pull-to-refresh to the Friends list.
- Added a proper empty state ("No friends yet") with a Find Friends action.
- Avatars now always fall back to the initials placeholder when the URL is null, empty, invalid, or fails to load — including a fix for the `CircleAvatar` assertion (`foregroundImage != null || onForegroundImageError == null`) that crashed the list for friends without an avatar.

### Public profile
- The public profile screen now distinguishes a genuinely missing/deleted user ("User not found") from loading and from real failures ("Couldn't load profile" with a Retry action) instead of showing "User not found" for every error.

### Profile & navigation
- The current user's avatar now appears in the Home screen header.
- The Groups / Friends / Photos stat cards on the Profile screen are now clickable and open the existing Groups (Home), Friends, and My Photos screens respectively, with Material ripple feedback.

### Offline-first architecture (major)
- The app is now **offline-first, stale-while-revalidate**: reads return cached data instantly, fresh data re-syncs in the background, and mutations (create/update/delete) require connectivity and fail fast instead of fabricating local-only data. See [`docs/OFFLINE_FIRST.md`](docs/OFFLINE_FIRST.md) for the full design.
- **Hive metadata cache** (`lib/core/storage/hive_service.dart`) — five boxes (`userProfile`, `groups`, `albums`, `photos`, `notifications`), all opened concurrently before `runApp()`, with synchronous reads for instant paint. Albums, photos, and notifications were rewritten from in-memory maps / hardcoded demo data to Hive-backed caches; Profile gained a dedicated `ProfileLocalDataSource`.
- **Image-byte cache** (`lib/core/cache/photoshare_cache_managers.dart`) — `HomeImageCacheManager` (30d/200 files), `ExploreImageCacheManager` (1d/40 files, reuses Home-cache hits, emptied every launch), and `PhotoImageCacheManager` (60d/500 files, the only persisted photo store). Album covers, collage grids, grid thumbnails, and full photo views all use `CachedNetworkImage` with these managers and stable `photo_{id}_thumbnail`/`_normalized` cache keys.
- **Connectivity** (`lib/core/network/connectivity_provider.dart`) — `isOnlineProvider` (link-level check) drives the new animated `OfflineBanner` on the Home screen, and `ConnectivityStatus` gives repositories a synchronous snapshot so offline reads return cache *immediately* instead of waiting out a doomed Supabase call. `ensureInitialized()` runs before `runApp()` so the first frame already knows the truth.
- **Auto-recovery** (`lib/core/network/connectivity_recovery.dart`) — on an offline→online transition, all user-scoped list providers (groups, group detail, albums, photos, notifications, friends) are invalidated; open screens rebuild, re-sync, and re-subscribe their Realtime channels automatically — no restart or manual refresh. Realtime channels torn down by `channelError`/`timedOut` are now also removed from the channel map so resubscribes are clean.
- **Startup speed** — the four independent startup steps (SharedPreferences, Hive init, connectivity check, dotenv) run in parallel; auth routes instantly from the local session (no blocking profile fetch) and refreshes the profile in the background.
- **Per-user cache isolation** — sign-out/sign-in now clears Hive boxes *and* empties the Home/Photo image caches, so a second account never sees the previous account's cached groups, albums, photos, notifications, profile, or ~3 MB of image bytes.

### Offline bug fixes
- Fixed **offline auto-logout**: `getCurrentUser()` used to swallow any failure (including a plain network outage) into `null`, which the session-refresh path treated as "session invalid" and forced the user to `/login`. Failures now rethrow into the existing handler that keeps the user signed in; only a genuinely invalid session signs out.
- Fixed the **album cover regression**: `Album.toJson()`/`fromJson()` never round-tripped `recent_photo_urls`, so once reads started coming from the Hive cache, 1/2/4-photo collage covers collapsed to a bare placeholder even online.
- Added **generation counters** to Album/Photo list notifiers so an older in-flight fetch can no longer overwrite a newer result.
- **My Photos is now offline-first** (was a bare Supabase call that errored offline), and **Photo Detail** distinguishes "not available offline yet" (`offline`) from "photo removed" (`notFound`), with cache-first metadata lookups for album, My Photos, and Profile-grid entry points.
- **Profile is offline-first**: instant cached identity/stats/previews, connectivity-gated refresh chain, deep-cast fix for Hive's untyped nested maps (the "My Groups" crash), and Edit-Profile fails fast offline (no fake success) with a bounded 8-second timeout.
- **Friends** explicitly show a connection-error state with Retry when offline (previously an offline read silently produced a misleading "No friends yet" empty state).

### Storage Dashboard
- New Profile → Storage screen (`/storage-dashboard`): total cached size, Home/Photos breakdown bar, per-file list with thumbnails, single-file delete, multi-select delete, and "Clear All" (through `emptyCache()` so cache-manager bookkeeping stays in sync).
- Reads the correct directory (`getTemporaryDirectory()` — previously it scanned `getApplicationSupportDirectory()`, where the cache never writes, so it always reported zero).
- Legend uses `Wrap` so it no longer overflows on narrow phones; Explore's cache is intentionally excluded from accounting (it never persists across launches).

## [0.0.1-alpha] — First public alpha

Initial public release of PhotoShare, distributed as a signed Android APK.

### Authentication
- Email/password sign-up and sign-in via Supabase Auth.
- Email confirmation and password-reset flows handled through a custom `photoshare://` deep-link scheme.
- Session restore on app start, with logout fully clearing local session state (Realtime subscriptions, image cache, and cached provider data) so a second account never sees a previous account's data on the same device.
- Password reset does not reveal whether an email address has an associated account, in line with standard anti-enumeration practice.

### Groups
- Create private (invite-only) or public (discoverable) groups.
- Group cover image, description, and up to 5 topic tags.
- Member roles: Owner, Admin, Member.
- Invite friends directly to a group, or request to join a public group that requires approval.

### Albums
- Create albums inside a group.
- Per-album visibility: choose exactly which group members can see a given album.

### Photos
- Upload photos into an album with an optional caption.
- Per-photo visibility: further restrict which members can see an individual photo, independent of the album's own visibility.
- Photo detail view with comments and likes.
- Shareable per-photo links.

### Explore
- Discover public groups, sorted by recent activity ("Hot").
- Browse trending public photos across groups.

### Friends
- Search users by username.
- Send, accept, and decline friend requests.
- View mutual groups on another user's public profile.

### Notifications
- In-app notifications for likes, comments, group invitations, and other membership activity.
- Filterable by type (Likes / Comments / Activity).

### Profile
- View and edit your own profile (display name, username, bio, avatar).
- Profile stats: groups, friends, photos.
- Light and dark theme support.

### Known issues at this release
- Android-only; no iOS, web, or desktop build.
- Alpha-quality software — see the "Known Limitations" section in `README.md`.
