# Architecture

StudioSoundSet has:

- Admin Hub
- Player Frontend
- Backend API
- Database
- Spotify API integration
- Spotify Web Playback SDK inside Player frontend
- Command queue
- Player state heartbeat
- Player test center
- Playlist track sync pipeline

Communication:

Admin creates PlayerCommand.
Player polls pending commands.
Player executes command locally.
Player writes result.
Player updates PlayerState.
Admin displays confirmed state.

Command success must only be shown after Player confirms execution.
No fake success states.

MVP uses polling.
Player polls every 2 seconds for commands.
Player sends heartbeat every 3 seconds.

Later upgrade:
WebSocket / Socket.IO.
