# StudioSoundSet Product Brief

StudioSoundSet is a multi-zone music control hub with two frontends:

1. Admin Hub
2. StudioSoundSet Player

The Admin Hub controls multiple Player accounts.
Each Player runs on a tablet/browser/PWA, connects its own Spotify Premium account, loads the Spotify Web Playback SDK and plays music itself.

The Admin does not control the normal Spotify app.
The Admin sends internal commands to the StudioSoundSet Player.

Core goals:
- create Player accounts
- login as Player
- connect Spotify account per Player
- load Spotify Web Playback SDK
- import Spotify playlists and tracks
- show Now Playing live in Admin
- control play/pause/resume/skip
- control app-internal volume
- run a real Player Test Center
- schedule playlists and volume ramps
- log every command
- never show fake success

MVP target:
One Admin creates one Player called Gym Player.
Android tablet opens /player-login.
Player connects Spotify.
Admin imports playlist with tracks.
Admin starts playlist.
Player plays.
Admin sees Now Playing.
Admin changes volume.
Player confirms success.
