# Spotify Integration

Use Spotify only inside the StudioSoundSet Player system.
Do not control the normal Spotify app as primary strategy.

Each Player connects its own Spotify Premium account.

Spotify scopes:
- streaming
- user-read-email
- user-read-private
- user-read-playback-state
- user-modify-playback-state
- user-read-currently-playing
- playlist-read-private
- playlist-read-collaborative

Player Frontend:
- loads Spotify Web Playback SDK
- initializes player
- receives ready event with device_id
- uses getCurrentState
- listens to player_state_changed
- executes pause, resume, nextTrack, previousTrack, setVolume
- sends PlayerState heartbeat

Backend:
- handles OAuth
- refreshes tokens
- stores tokens securely
- imports playlists
- imports playlist tracks paginated
- starts playlist playback on SDK device_id using Spotify Web API

Important:
The admin never marks commands as successful.
Only the Player can confirm execution.
