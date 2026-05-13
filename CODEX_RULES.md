# Codex Rules for StudioSoundSet

1. Do not create fake success states.
2. Player commands are successful only after Player acknowledgment.
3. Admin never directly assumes playback state.
4. PlayerState is the source of truth for Admin live views.
5. Playlist import is incomplete unless PlaylistTrack rows exist.
6. Spotify app remote control is not the primary strategy.
7. StudioSoundSet Player with Spotify Web Playback SDK is the primary strategy.
8. Secrets must never be exposed in frontend.
9. Every feature must include loading, success, error states.
10. Every command failure needs errorCode, humanMessage, suggestedFix.
11. Build in phases. Do not implement future phases until current acceptance criteria pass.
