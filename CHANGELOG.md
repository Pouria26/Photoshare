# Changelog

All notable changes to this project are documented in this file.

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
