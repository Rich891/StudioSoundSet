# Player Test Center

Build a real test center. Do not simulate success.

Full test pipeline:
1. Check Player heartbeat
2. Check SDK loaded
3. Check SDK ready
4. Check Spotify connected
5. GET_STATE
6. PLAY_PLAYLIST with selected test playlist
7. Verify playing
8. PAUSE
9. Verify paused
10. RESUME
11. Verify playing
12. SET_VOLUME low, e.g. 0.25
13. Verify volume low
14. SET_VOLUME high, e.g. 0.65
15. Verify volume high
16. SKIP_NEXT
17. Verify track changed
18. PLAY_PLAYLIST with second playlist
19. Verify playlist switch
20. Restore original volume
21. Final state sync

Every step creates PlayerTestStep.
Every command creates PlayerCommand.
Success only after Player confirms.
Failures must include errorCode, technicalMessage, humanMessage, suggestedFix.

Required error codes:
- PLAYER_OFFLINE
- SDK_NOT_LOADED
- SDK_NOT_READY
- SPOTIFY_NOT_CONNECTED
- TOKEN_EXPIRED
- PREMIUM_REQUIRED
- NO_DEVICE_ID
- NO_TEST_PLAYLIST
- PLAYBACK_START_FAILED
- STATE_NOT_AVAILABLE
- PAUSE_FAILED
- RESUME_FAILED
- VOLUME_SET_FAILED
- VOLUME_NOT_CONFIRMED
- SKIP_FAILED
- PLAYLIST_SWITCH_FAILED
- COMMAND_TIMEOUT
- PLAYER_DID_NOT_ACK
