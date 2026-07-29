# Architecture

PhotoShare is a Flutter application organized as **feature-first Clean Architecture**, backed entirely by Supabase (no custom backend server).

## Layers

Each feature under `lib/features/<feature>/` is split into three layers:

```
feature/
├── data/
│   ├── datasources/      # Direct Supabase/Cloudinary calls (local + remote)
│   └── repositories/     # Implements the domain repository interface
├── domain/
│   ├── entities/         # Plain Dart data classes
│   ├── repositories/     # Abstract repository interfaces
│   └── usecases/         # One class per user action, wrapping a repository call
└── presentation/
    ├── providers/        # Riverpod providers / StateNotifiers
    ├── screens/          # Full-page widgets, wired to providers
    └── widgets/          # Feature-local reusable widgets
```

The **domain** layer has no Flutter or Supabase imports — it only depends on plain Dart. The **data** layer implements domain repository interfaces using `supabase_flutter` and direct Cloudinary HTTP calls. The **presentation** layer depends on domain use cases (through Riverpod providers), never directly on data-layer classes.

```
┌─────────────────────────────────────────────────────┐
│                    PRESENTATION                       │
│  Screens → Riverpod Providers (StateNotifier/etc.)    │
├─────────────────────────────────────────────────────┤
│                      DOMAIN                           │
│  Entities → Use Cases → Repository Interfaces         │
├─────────────────────────────────────────────────────┤
│                       DATA                            │
│  DataSources (local + remote) → Repository Impls      │
├─────────────────────────────────────────────────────┤
│                     EXTERNAL                          │
│  Supabase (Postgres/Auth/Realtime) │ Cloudinary       │
└─────────────────────────────────────────────────────┘
```

## Features

| Feature | Responsibility |
|---|---|
| `auth` | Sign-up/sign-in, session handling, password reset, deep-link-driven email confirmation and password recovery |
| `groups` | Group CRUD, membership, roles (Owner/Admin/Member), invitations, join requests |
| `albums` | Albums within a group, including per-member album visibility |
| `photos` | Photo upload/detail, per-member photo visibility, shareable photo links |
| `social` | Comments and likes on photos |
| `friends` | Friend search, requests, accept/decline |
| `notifications` | In-app notification feed |
| `explore` | Public group and public photo discovery |
| `profile` | Own profile management and public profile viewing |

## Top-level structure

```
lib/
├── main.dart      # Supabase.initialize(), top-level auth-event logging, runApp()
├── app/           # MaterialApp + go_router configuration
├── core/          # Constants, theming, network layer, logging, reusable widgets
├── features/      # One folder per feature, structured as above
└── shared/
    └── services/  # Cross-feature services: Cloudinary uploads, Supabase Realtime
                    # channel management, local notifications, local file storage
```

## State management

Riverpod (`flutter_riverpod`) is used throughout with hand-written providers — `Provider`, `StateNotifierProvider`, `StreamProvider`, and `.family` variants where state is keyed by an id (e.g. a specific group or album). Code generation (`riverpod_generator`) is listed as a dev dependency but is not currently used by any provider in the app.

Each feature typically exposes:
- Provider(s) for its data sources and repository (`Provider`)
- Provider(s) for its use cases (`Provider`)
- One or more `StateNotifierProvider`s holding the feature's UI state (loading/loaded/error + data)

## Routing

Navigation uses `go_router` with a single `GoRouter` instance (`routerProvider`). Auth-gated redirect logic reads the current authentication state on every navigation attempt, redirecting unauthenticated users to the login screen (preserving their intended destination) and redirecting authenticated users away from auth-only screens. Password-recovery and email-confirmation deep links (`photoshare://...`) are handled by dedicated routes so the app can react to them without relying on ad-hoc imperative navigation.

## Realtime updates

`shared/services/realtime_service.dart` manages Supabase Realtime channel subscriptions (Postgres changes) on behalf of feature providers. A provider subscribes to the relevant table(s) for its screen, updates its own state when a change event arrives, and the channel is torn down when no longer needed (and unconditionally on logout, to avoid leaking subscriptions or data across accounts on the same device).

## Backend

There is no custom backend server. Supabase provides:
- **Auth** (email/password, session management)
- **PostgreSQL** (all application data, with Row Level Security enforcing access — including the per-member album/photo visibility rules)
- **Realtime** (Postgres change subscriptions)

Image assets are hosted on **Cloudinary**, uploaded directly from the client via HTTP (no Cloudinary SDK dependency).
