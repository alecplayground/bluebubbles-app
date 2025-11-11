# FaceTime Research Notes - Android Client

## Initial Discovery

### File Structure
Found 41 Kotlin files in the Android app located at `android/app/src/main/kotlin/`

### Key FaceTime-Related Files Identified
1. `CreateIncomingFaceTimeNotification.kt` - Handles incoming FaceTime notification display
2. `MethodCallHandler.kt` - Routes method calls between Flutter and Android native code
3. `Constants.kt` - Defines constants including FaceTime notification tags

## Component Analysis

### 1. CreateIncomingFaceTimeNotification.kt

**Location**: `android/app/src/main/kotlin/com/bluebubbles/messaging/services/notifications/CreateIncomingFaceTimeNotification.kt`

**Purpose**: Creates and displays Android notifications for incoming FaceTime calls

**Key Findings**:
- This is a MethodCallHandler implementation that receives calls from Flutter/Dart
- Tag: `"create-incoming-facetime-notification"`
- Invoked via Flutter's MethodChannel: `com.bluebubbles.messaging`

**Parameters Received from Flutter**:
- `channel_id` (String) - Android notification channel ID
- `notification_id` (Int) - Unique ID for this notification
- `call_uuid` (String?) - UUID identifying the call
- `title` (String) - Notification title
- `body` (String) - Notification body text
- `caller` (String) - Name of the caller
- `caller_avatar` (ByteArray?) - Caller's avatar image as byte array

**Notification Actions**:
1. **Answer Action** (line 67-79):
   - Creates PendingIntent with type `"AnswerFaceTime"`
   - Opens MainActivity
   - Includes extras: `callUuid`, `answer=true`, `caller` name
   - Uses pending intent ID: `notificationId + Constants.pendingIntentAnswerFaceTimeOffset` (-100000)

2. **Decline/Ignore Action** (line 82-92):
   - Creates PendingIntent with type `"DeleteNotification"`
   - Sends broadcast to InternalIntentReceiver
   - Uses pending intent ID: `notificationId + Constants.pendingIntentDeclineFaceTimeOffset` (-200000)

3. **Open Summary Action** (line 55-64):
   - Tapping the notification body
   - Opens MainActivity with `answer=false`

**Notification Properties**:
- Category: `CATEGORY_CALL`
- Priority: `PRIORITY_MAX`
- Auto-cancel: true
- Ongoing: true (prevents user from dismissing)
- Timeout: 30 seconds (line 114) - automatically clears if no server response
- Notification Tag: `Constants.newFaceTimeNotificationTag` = `"com.bluebubbles.messaging.NEW_FACETIME_NOTIFICATION"`
- Color: 4888294 (blue color)
- Wearable support: Actions available on Android Wear devices

**Important Details**:
- The notification is purely for alerting - it does NOT handle the actual call media
- When "Answer" is tapped, it opens MainActivity which presumably launches the Flutter UI
- The Flutter layer must handle the actual call connection and media streaming

### 2. MethodCallHandler.kt

**Location**: `android/app/src/main/kotlin/com/bluebubbles/messaging/services/backend_ui_interop/MethodCallHandler.kt`

**Purpose**: Central dispatcher for Flutter-to-Android method calls

**Key Findings**:
- Routes method call `CreateIncomingFaceTimeNotification.tag` to CreateIncomingFaceTimeNotification handler (line 68)
- Uses Flutter's MethodChannel: `com.bluebubbles.messaging`
- Provides `invokeMethod()` companion function to send calls from Android back to Flutter (lines 37-41)

**Method Channel Communication**:
- Android → Flutter: `MethodCallHandler.invokeMethod(method, arguments)`
- Flutter → Android: Routed through `methodCallHandler()` function

### 3. Constants.kt

**FaceTime-Related Constants**:
- `newFaceTimeNotificationTag`: `"com.bluebubbles.messaging.NEW_FACETIME_NOTIFICATION"`
- `pendingIntentAnswerFaceTimeOffset`: -100000
- `pendingIntentDeclineFaceTimeOffset`: -200000

## Architecture Pattern Identified

The Android native code appears to handle ONLY the notification UI layer for FaceTime. The actual call handling (WebRTC, media streaming, signaling) likely occurs at the Flutter/Dart layer.

**Flow Pattern**:
```
Server → Flutter/Dart → Android Native (notification) → User Interaction → Flutter/Dart (call handling)
```

## Research Complete

### Major Findings

**CRITICAL DISCOVERY**: The BlueBubbles Android app does NOT handle FaceTime audio/video streaming directly. Instead:

1. **No WebRTC in Client**: No WebRTC implementation found in Android client
2. **Notification Relay Pattern**: Client acts as notification relay only
3. **External Browser Handling**: Actual calls happen in external browsers via FaceTime web links
4. **Server-Side Private API**: All Private API usage is on the macOS server

### Flow Summary

**Incoming Call Flow**:
```
macOS Server (Private API detects call)
  ↓ (FCM/Socket.IO)
Android Client (receives notification)
  ↓ (displays notification)
User (taps Answer)
  ↓ (requests link from server)
Server (generates FaceTime web link via Private API)
  ↓ (returns link)
Android Client (launches link externally)
  ↓
Browser (handles actual WebRTC call)
```

**What the Android Client Does**:
- Receives FaceTime call notifications via FCM or Socket.IO
- Displays Android notifications with Answer/Ignore buttons
- Requests FaceTime web links from server API
- Opens links in external applications (browsers)
- Handles notification lifecycle (create, clear, timeout)

**What the Android Client Does NOT Do**:
- ❌ Handle audio streaming
- ❌ Handle video streaming
- ❌ Implement WebRTC
- ❌ Use Private APIs directly
- ❌ Manage call media

### All Files Analyzed

**Android Native (Kotlin)**:
- ✅ BlueBubblesFirebaseMessagingService.kt
- ✅ DartWorkManager.kt
- ✅ DartWorker.kt
- ✅ CreateIncomingFaceTimeNotification.kt
- ✅ MethodCallHandler.kt
- ✅ MainActivity.kt
- ✅ Constants.kt

**Flutter/Dart**:
- ✅ intents_service.dart
- ✅ action_handler.dart
- ✅ notifications_service.dart
- ✅ facetime_helpers.dart
- ✅ http_service.dart
- ✅ socket_service.dart
- ✅ method_channel_service.dart
- ✅ private_api_panel.dart

### Documentation Created

1. **facetime-research-notes.md** - Initial research notes
2. **critical-finding.md** - Key architectural discovery
3. **facetime-flow-detailed.md** - Complete technical documentation with diagrams
