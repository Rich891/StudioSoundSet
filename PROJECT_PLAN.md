# StudioSoundSet Project Plan

## Product Understanding

StudioSoundSet is a multi-zone music control hub with two separate frontends:

- Admin Hub: used by owners/admins/staff to create zones, create player accounts, import playlists, schedule playback, monitor state, and issue commands.
- StudioSoundSet Player Frontend: a kiosk-style browser/PWA used by each physical zone device. It logs in as a Player user, connects its own Spotify Premium account, loads the Spotify Web Playback SDK, and performs playback locally.

The Admin Hub must never control the normal Spotify app as the primary playback target. Admin actions create internal `PlayerCommand` records. The Player polls, acknowledges, executes via the Spotify Web Playback SDK or Web API targeting its SDK `device_id`, writes the command result, and updates `PlayerState`. Admin UI may show pending/running states immediately, but success is valid only after Player acknowledgement.

## Key Constraints

- Playback must happen in StudioSoundSet Player Frontend through Spotify Web Playback SDK.
- The app must not depend on controlling the normal Spotify app.
- No fake success states: every command needs a Player-confirmed terminal result.
- Every command lifecycle must be visible: `pending`, `picked_up`, `running`, `success`, `failed`, or `timeout`.
- Player Test Center must run real commands and validations, not mocked or simulated pass states.
- Each Player needs its own Spotify Premium account and OAuth token set.

## Contradictions, Gaps, And Risks

### Contradictions / Tensions

- `PLAY_PLAYLIST` is described as a Player command, but Spotify playlist playback start is assigned to the Backend using Spotify Web API against the SDK `device_id`. Resolution: Admin creates `PLAY_PLAYLIST`; Player must acknowledge, request/receive a backend-assisted playback start for its own device, then verify SDK state before marking success.
- Admin imports playlists for a Player, but Spotify OAuth belongs to the Player account. Resolution: playlist browsing/import must use the selected Player's Spotify token, refreshed server-side.
- App-internal volume is required, while Spotify SDK `setVolume` controls the Web Playback SDK volume. Resolution: treat `currentVolume` as SDK volume, and instruct device system volume to remain at 100%.
- MVP acceptance includes owner account creation, API credentials, zones, player account, OAuth, playback, command acknowledgement, test center, and track import. That is a large MVP. Resolution: split MVP into milestone gates, but do not call the MVP complete until all acceptance criteria pass.

### Missing Requirements

- Authentication provider decision and session model.
- Player login credential format, rotation, reset, and QR-token expiry rules.
- Spotify OAuth redirect URI strategy for multiple Players and environments.
- Token encryption approach and key management.
- Command timeout duration, retry policy, and idempotency behavior.
- Exact command payload schemas for all command types.
- Scheduler execution semantics around overlapping blocks, timezone, missed jobs, and offline Players.
- Multi-tenant model: whether one owner can own many organizations/venues or only one workspace.
- Browser/device support matrix for Android tablet/PWA and Spotify Web Playback SDK behavior.
- How Player stays awake, handles autoplay restrictions, and recovers after refresh/reboot.
- Audit log retention, privacy handling, and operational logs.

### Technical Risks

- Spotify Web Playback SDK requires Spotify Premium and may be sensitive to browser/device restrictions.
- Browser autoplay/user-gesture rules can block first playback until Player UI has been activated.
- SDK device IDs are ephemeral and must be refreshed on every Player boot/reconnect.
- Polling can produce duplicate or stale command execution without locking and idempotency.
- Token refresh failure can cause misleading command failures unless surfaced precisely.
- Admin progress bars can drift; they must be derived from last confirmed Player state and labeled as live-estimated.
- Scheduler commands can pile up if Player is offline; offline behavior must be explicit.
- Secure storage of Spotify client secrets and refresh tokens is mandatory before real users.

## Recommended Stack

The repository currently contains docs only, so the stack is not present. Recommended stack:

- Framework: Next.js App Router with TypeScript.
- UI: React, Tailwind CSS, shadcn/ui-style primitives where useful, lucide-react icons.
- Backend: Next.js route handlers/server actions for MVP; split to a worker service later if scheduler load grows.
- Database: PostgreSQL.
- ORM: Prisma.
- Auth: Auth.js or Supabase Auth. If using Supabase, still keep Prisma-compatible schema discipline.
- Jobs: pg-boss or BullMQ for scheduler, command timeout sweeps, playlist sync jobs, and token refresh tasks.
- Realtime MVP: polling exactly as specified, Player every 2 seconds for commands and every 3 seconds for heartbeat.
- Realtime later: WebSocket/Socket.IO or hosted realtime channels after command semantics are proven.
- Secrets: encrypted columns for Spotify client secrets and refresh tokens, using an environment-provided encryption key.
- Testing: Playwright for end-to-end Admin/Player flows, Vitest for command lifecycle and scheduler logic.

## Technical Architecture

### Runtime Components

- Admin Hub frontend
  - Owner/admin/staff UI.
  - Shows setup flow, zones, players, playlists, now playing, command history, test center, and schedule calendar.
  - Creates commands through Backend API.
  - Displays success only from `PlayerCommand.status = success` written by Player-confirmed execution.

- Player Frontend
  - Kiosk/PWA UI for Player role only.
  - Loads Spotify Web Playback SDK.
  - Connects Spotify account through OAuth.
  - Maintains SDK readiness and `spotifyDeviceId`.
  - Polls commands every 2 seconds.
  - Sends heartbeat/state every 3 seconds.
  - Executes SDK commands locally and verifies via SDK state before terminal success.

- Backend API
  - Auth/session handling.
  - Role and ownership enforcement.
  - Player account creation and credential reset.
  - Spotify OAuth callback and token refresh.
  - Playlist metadata and paginated track import.
  - Command creation, claiming, result recording, timeout handling.
  - PlayerState ingestion.
  - Player Test Center orchestration.

- Database
  - Source of truth for users, zones, players, Spotify accounts, commands, states, playlists, tracks, schedules, and test runs.

- Worker / Scheduler
  - Executes schedule blocks by creating PlayerCommands.
  - Marks stale commands as `timeout`.
  - Runs token refresh and playlist sync jobs.
  - Can start inside Next.js for development, but should be a separate process for production.

### Command Acknowledgement Contract

1. Admin action calls Backend API.
2. Backend validates permissions and creates `PlayerCommand(status='pending')`.
3. Admin displays command as pending/running, never as successful.
4. Player polls `/api/player/commands/next`.
5. Backend atomically assigns one command and sets `picked_up`.
6. Player marks `running` when execution starts.
7. Player executes the command locally using SDK where possible.
8. For playlist start, Player coordinates backend Web API start against its current SDK `spotifyDeviceId`, then verifies SDK state.
9. Player writes `success` only after verified execution, or `failed` with structured error details.
10. Admin updates UI from command result and latest `PlayerState`.
11. Timeout worker marks commands with no timely Player acknowledgement as `timeout`.

### Core Command Types

- `GET_STATE`
- `PLAY_PLAYLIST`
- `PAUSE`
- `RESUME`
- `SKIP_NEXT`
- `SKIP_PREVIOUS`
- `SET_VOLUME`
- `SYNC_STATE`
- `RESTORE_VOLUME`

Each command payload must include an idempotency key or command id, target player id, expected preconditions, and command-specific parameters.

## Route Structure

### Public / Auth

- `/` redirects authenticated users by role.
- `/login` owner/admin/staff login.
- `/signup` owner setup for first account.
- `/player-login` Player login for tablets.
- `/auth/spotify/connect` starts Spotify OAuth for selected Player context.
- `/auth/spotify/callback` handles Spotify OAuth callback.

### Admin Hub

- `/admin` dashboard and setup progress.
- `/admin/api-credentials` list/create/test credential sets.
- `/admin/zones` zone list.
- `/admin/zones/new` create zone.
- `/admin/zones/[zoneId]` zone detail.
- `/admin/players` player list.
- `/admin/players/new` create player wizard.
- `/admin/players/[playerId]` player detail/status/commands.
- `/admin/players/[playerId]/credentials` view/reset player credentials and QR URL.
- `/admin/players/[playerId]/test` Player Test Center.
- `/admin/playlists` imported playlist list.
- `/admin/playlists/import` load Spotify playlists for a selected Player.
- `/admin/playlists/[playlistId]` playlist detail with full track list.
- `/admin/now-playing` multi-zone now playing overview.
- `/admin/schedule` weekly calendar view.
- `/admin/schedule/new` create schedule block.
- `/admin/commands` command log and filters.

### Player Frontend

- `/player` Player kiosk UI.
- `/player/connect-spotify` Player-scoped Spotify connect screen.
- `/player/status` diagnostics/status screen.

### API Routes

- `POST /api/auth/player-login`
- `POST /api/api-credentials`
- `POST /api/api-credentials/[id]/test`
- `POST /api/zones`
- `POST /api/players`
- `POST /api/players/[id]/reset-password`
- `GET /api/players/[id]/state`
- `GET /api/players/[id]/commands`
- `POST /api/players/[id]/commands`
- `POST /api/player/heartbeat`
- `POST /api/player/state`
- `POST /api/player/commands/next`
- `POST /api/player/commands/[commandId]/running`
- `POST /api/player/commands/[commandId]/result`
- `GET /api/spotify/playlists?playerId=...`
- `POST /api/spotify/playlists/import`
- `POST /api/spotify/playlists/[playlistId]/sync-tracks`
- `POST /api/spotify/playback/start`
- `POST /api/player-tests`
- `POST /api/player-tests/[testRunId]/steps/[stepId]/result`
- `POST /api/schedule-blocks`
- `GET /api/schedule-blocks`

## Database Schema

Use PostgreSQL UUID primary keys, `created_at`, and `updated_at` on all main tables. Add foreign keys and indexes for every lookup path used by polling, dashboards, and scheduler jobs.

### `user_profiles`

- `id uuid primary key`
- `auth_user_id text unique not null`
- `email text not null`
- `display_name text`
- `role text not null check in ('owner','admin','staff','player')`
- `status text not null default 'active'`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

### `api_credential_sets`

- `id uuid primary key`
- `name text not null`
- `provider_type text not null default 'spotify'`
- `environment text not null`
- `client_id text not null`
- `client_secret_encrypted text not null`
- `redirect_uri text not null`
- `scopes text[] not null`
- `status text not null default 'untested'`
- `last_test_at timestamptz`
- `last_error text`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

### `zones`

- `id uuid primary key`
- `name text not null`
- `color text`
- `player_id uuid unique`
- `default_volume numeric(4,3) not null default 0.5`
- `min_volume numeric(4,3) not null default 0`
- `max_volume numeric(4,3) not null default 1`
- `status text not null default 'active'`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

### `players`

- `id uuid primary key`
- `name text not null`
- `zone_id uuid references zones(id)`
- `player_user_id uuid references user_profiles(id)`
- `api_credential_set_id uuid references api_credential_sets(id)`
- `spotify_account_id uuid`
- `device_type text`
- `supports_local_volume boolean not null default true`
- `status text not null default 'created'`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

### `spotify_player_accounts`

- `id uuid primary key`
- `player_id uuid not null references players(id)`
- `zone_id uuid references zones(id)`
- `api_credential_set_id uuid not null references api_credential_sets(id)`
- `spotify_user_id text`
- `display_name text`
- `auth_status text not null default 'not_connected'`
- `token_status text not null default 'missing'`
- `access_token_encrypted text`
- `refresh_token_encrypted text`
- `scopes text[] not null default '{}'`
- `token_expires_at timestamptz`
- `premium_status text not null default 'unknown'`
- `last_error text`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

### `player_states`

- `id uuid primary key`
- `player_id uuid unique not null references players(id)`
- `zone_id uuid references zones(id)`
- `online boolean not null default false`
- `last_heartbeat_at timestamptz`
- `sdk_loaded boolean not null default false`
- `sdk_ready boolean not null default false`
- `spotify_connected boolean not null default false`
- `spotify_device_id text`
- `current_track text`
- `current_artist text`
- `current_album text`
- `current_cover_url text`
- `current_playlist text`
- `current_track_uri text`
- `current_context_uri text`
- `position_ms integer`
- `duration_ms integer`
- `is_playing boolean not null default false`
- `current_volume numeric(4,3)`
- `last_command_id uuid`
- `last_command_status text`
- `last_error text`
- `updated_at timestamptz not null`

### `player_commands`

- `id uuid primary key`
- `target_player_id uuid not null references players(id)`
- `zone_id uuid references zones(id)`
- `type text not null`
- `payload_json jsonb not null default '{}'`
- `status text not null check in ('pending','picked_up','running','success','failed','timeout')`
- `created_by uuid references user_profiles(id)`
- `created_at timestamptz not null`
- `picked_up_at timestamptz`
- `running_at timestamptz`
- `completed_at timestamptz`
- `timeout_at timestamptz`
- `error_code text`
- `technical_message text`
- `human_message text`
- `suggested_fix text`
- `result_json jsonb not null default '{}'`

Indexes:

- `(target_player_id, status, created_at)` for Player polling.
- `(zone_id, created_at)` for Admin command log.
- `(status, created_at)` for timeout worker.

### `playlists`

- `id uuid primary key`
- `player_id uuid not null references players(id)`
- `zone_id uuid references zones(id)`
- `spotify_account_id uuid references spotify_player_accounts(id)`
- `provider_playlist_id text not null`
- `provider_playlist_uri text not null`
- `name text not null`
- `description text`
- `owner text`
- `cover_url text`
- `total_tracks integer not null default 0`
- `imported_tracks integer not null default 0`
- `metadata_sync_status text not null default 'pending'`
- `track_sync_status text not null default 'pending'`
- `last_sync_at timestamptz`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

Unique index: `(spotify_account_id, provider_playlist_id)`.

### `playlist_tracks`

- `id uuid primary key`
- `playlist_id uuid not null references playlists(id) on delete cascade`
- `provider_track_id text`
- `provider_track_uri text not null`
- `name text not null`
- `artist text`
- `album text`
- `duration_ms integer`
- `cover_url text`
- `explicit boolean not null default false`
- `sort_order integer not null`
- `is_playable boolean not null default true`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

Unique index: `(playlist_id, sort_order)`.

### `schedule_blocks`

- `id uuid primary key`
- `zone_id uuid not null references zones(id)`
- `player_id uuid not null references players(id)`
- `playlist_id uuid not null references playlists(id)`
- `title text not null`
- `day_of_week integer not null check between 1 and 7`
- `start_time time not null`
- `end_time time not null`
- `timezone text not null default 'Europe/Berlin'`
- `base_volume numeric(4,3)`
- `ramp_enabled boolean not null default false`
- `start_volume numeric(4,3)`
- `end_volume numeric(4,3)`
- `ramp_mode text`
- `repeat_weekly boolean not null default true`
- `active boolean not null default true`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

### `player_test_runs`

- `id uuid primary key`
- `player_id uuid not null references players(id)`
- `zone_id uuid references zones(id)`
- `started_by uuid references user_profiles(id)`
- `status text not null default 'running'`
- `mode text not null default 'full'`
- `started_at timestamptz not null`
- `completed_at timestamptz`
- `summary text`

### `player_test_steps`

- `id uuid primary key`
- `test_run_id uuid not null references player_test_runs(id) on delete cascade`
- `player_id uuid not null references players(id)`
- `step_key text not null`
- `step_name text not null`
- `status text not null default 'pending'`
- `command_id uuid references player_commands(id)`
- `expected_result text`
- `actual_result text`
- `technical_message text`
- `human_message text`
- `suggested_fix text`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`

## MVP Milestones

### Milestone 1: Foundation

- Create Next.js/TypeScript app, PostgreSQL, Prisma, Tailwind, auth, role guards.
- Implement owner signup and role-based redirects.
- Add schema migrations for core tables.

Acceptance:

- Owner can create account and log in.
- Player role cannot access Admin routes.
- Admin/staff cannot access Player kiosk route as a Player session.

### Milestone 2: Setup Entities

- API Credential Set CRUD and test action.
- Zone CRUD.
- Player creation wizard.
- Generated Player login credentials and QR/player URL.

Acceptance:

- Owner can create API credentials, zone, and Player account.
- Tablet can log in through `/player-login` and lands on `/player`.

### Milestone 3: Player Spotify Connection

- Player-scoped Spotify OAuth.
- Secure token storage and refresh.
- Web Playback SDK loading in Player Frontend.
- SDK ready/device_id capture.
- Heartbeat and PlayerState updates.

Acceptance:

- Player connects Spotify Premium account.
- Player reports `sdkLoaded`, `sdkReady`, `spotifyConnected`, and `spotifyDeviceId`.
- Admin sees Player online and connected from real heartbeat.

### Milestone 4: Command System

- Command creation from Admin.
- Player polling, atomic claim, running/result endpoints.
- SDK execution for pause, resume, skip, get state, and set volume.
- Timeout worker.

Acceptance:

- Admin actions create commands.
- Player acknowledges and executes commands.
- Admin success appears only after Player writes success.
- Failed commands show structured error details.

### Milestone 5: Playlist Import And Playback

- Load Spotify playlists for selected Player account.
- Import playlist metadata and paginated tracks.
- Playlist detail page with full track list.
- `PLAY_PLAYLIST` command targets Player SDK device and verifies playback.

Acceptance:

- Imported playlists include `PlaylistTrack` rows.
- Admin starts a playlist.
- Player plays through StudioSoundSet Player Frontend.
- Admin sees confirmed track, artist, cover, duration, position, and volume.

### Milestone 6: Player Test Center

- Full test run and step creation.
- Required error codes.
- Real command-backed validation for playback, pause, resume, volume, skip, and playlist switch.

Acceptance:

- Test Center runs every documented step.
- Success only appears when Player confirms and verification passes.
- Failures include `errorCode`, `technicalMessage`, `humanMessage`, and `suggestedFix`.

### Milestone 7: Scheduler And Calendar

- Weekly calendar UI Monday-Sunday with 15-minute slots.
- ScheduleBlock CRUD.
- Worker creates playback and volume commands at scheduled times.
- Define overlap/offline behavior before enabling production scheduling.

Acceptance:

- Admin can create schedule blocks visually.
- Active schedule creates PlayerCommands at the expected local time.
- Offline Player schedules fail or queue according to documented policy, with no fake success.

### Milestone 8: UX Polish And Hardening

- Dark neon control center UI.
- Strong loading/warning/error states.
- Command logs and diagnostics.
- Playwright coverage for core Admin/Player journey.

Acceptance:

- MVP acceptance checklist in `docs/08-acceptance-tests.md` passes end to end.
- No code path marks command success without Player result.

## Acceptance Criteria

The MVP is complete only when:

- Owner can create account.
- Owner can create API Credential Set.
- Owner can create Zone.
- Owner can create Player Account.
- Player can login and only see Player UI.
- Player can connect Spotify.
- Player loads Spotify Web Playback SDK.
- Player becomes SDK ready.
- Player sends heartbeat.
- Admin sees Player online.
- Admin imports a Spotify playlist.
- App imports PlaylistTrack rows.
- Playlist detail page shows songs.
- Admin starts test playlist.
- Player plays music in StudioSoundSet Player Frontend.
- Admin sees track, artist, cover, duration, position.
- Admin sets volume.
- Player confirms actual volume.
- Admin pauses and resumes.
- Admin skips.
- Player Test Center runs all steps and reports real success/failure.
- No fake success states exist.
- The normal Spotify app is not the controlled playback target.

## Open Questions

- Which auth provider should be used: Auth.js, Supabase Auth, Clerk, or another provider?
- Is StudioSoundSet single-tenant for one owner, or should the schema include organizations/venues from day one?
- What is the command timeout threshold for each command type?
- Should commands be retried automatically, or should retries always require an Admin action/test step?
- What is the exact Player credential delivery policy and reset flow?
- What devices and browsers must be supported for the Player tablet MVP?
- How should first-play user activation be handled on Player devices before scheduled playback?
- What should happen to scheduled blocks when the Player is offline at start time?
- Can schedule blocks overlap, and if so, what priority rules apply?
- How long should command logs, PlayerState history, and test runs be retained?
- Should playlist sync be manual only for MVP or periodically refreshed?
- What production hosting target is preferred?
