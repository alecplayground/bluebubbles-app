# CRITICAL FINDING: FaceTime Implementation Architecture

## Summary

After deep analysis of the BlueBubbles Android client FaceTime implementation, I have discovered something very important:

**The BlueBubbles Android app does NOT handle actual FaceTime audio/video streaming internally. Instead, it acts as a notification relay that generates FaceTime web links from the server and opens them in external applications (typically a web browser).**

## Key Finding

When a user answers a FaceTime call:
1. The Android app calls the BlueBubbles server API: `POST /api/v1/facetime/answer/{callUuid}`
2. The server responds with a `link` (FaceTime web URL)
3. The Android app launches this link externally using `launchUrl(mode: LaunchMode.externalApplication)`
4. The actual FaceTime call happens in the external application (browser), NOT in the BlueBubbles app

**Source Evidence:**
- File: `lib/services/backend/java_dart_interop/intents_service.dart`, lines 141-157
- The answerFaceTime function calls `http.answerFaceTime(callUuid)` to get a link
- Then uses `launchUrl(Uri.parse(link), mode: LaunchMode.externalApplication)` to open it externally

## Implications

**There is NO:**
- WebRTC implementation in the Android client
- Audio streaming from server to Android client
- Audio streaming from Android client to server
- Video streaming from server to Android client
- Video streaming from Android client to server
- Direct media handling within the BlueBubbles Android app

**The Android app only:**
- Receives FaceTime call notifications (via FCM or Socket.IO)
- Displays notifications with Answer/Ignore buttons
- Generates FaceTime web links from the server
- Launches external applications to handle the actual call

## What Actually Happens

The BlueBubbles **server** (running on macOS) is doing the heavy lifting:
1. It detects incoming FaceTime calls via macOS Private APIs
2. It generates FaceTime web links (using Apple's FaceTime web feature)
3. It notifies clients via FCM/Socket.IO
4. Clients can request links and open them in browsers/external apps

The Android client is essentially a **remote notification and control interface** for FaceTime calls that are managed by the macOS server.
