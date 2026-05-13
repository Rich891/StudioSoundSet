# User Flows

## Owner Setup
1. Owner creates main login.
2. Owner creates API Credential Set.
3. Owner creates Zone.
4. Owner creates Player Account.
5. Player logs in on tablet.
6. Player connects Spotify.
7. Player SDK becomes ready.
8. Admin starts Player Test Center.
9. Admin imports Playlist including Tracks.
10. Admin starts Playlist.
11. Admin sees Now Playing.

## Player Creation
1. Admin selects Zone.
2. Admin enters Player name.
3. Admin selects API Credential Set.
4. System creates Player user login.
5. Admin gets Player login credentials and QR/player URL.
6. Tablet logs in as Player.
7. Player UI opens automatically.

## Player Command
1. Admin clicks action.
2. Backend creates PlayerCommand.
3. Player polls pending commands.
4. Player marks picked_up.
5. Player executes command.
6. Player writes success or failed.
7. Admin UI updates only after confirmation.

## Playlist Import
1. Admin selects Player.
2. Admin loads Spotify playlists.
3. Admin imports playlist metadata.
4. Backend loads all tracks paginated.
5. PlaylistTrack rows are created.
6. Playlist detail page displays songs.

## Live Now Playing
1. Player reads SDK state.
2. Player sends PlayerState heartbeat.
3. Admin displays track, artist, cover, position, duration, volume.
4. Admin progress bar advances locally based on last update.
