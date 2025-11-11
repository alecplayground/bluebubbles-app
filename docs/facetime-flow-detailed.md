# BlueBubbles Android FaceTime Flow - Detailed Technical Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Incoming FaceTime Call Flow](#incoming-facetime-call-flow)
3. [Answering FaceTime Call Flow](#answering-facetime-call-flow)
4. [Call Status Change Flow](#call-status-change-flow)
5. [Component Details](#component-details)
6. [Private API Integration](#private-api-integration)

---

## Architecture Overview

### System Components

The BlueBubbles FaceTime system consists of three primary layers:

1. **BlueBubbles Server (macOS)**
   - Runs on macOS with access to FaceTime Private APIs
   - Detects incoming FaceTime calls
   - Generates FaceTime web links for answering calls
   - Sends notifications to clients via FCM/Socket.IO

2. **BlueBubbles Android Client**
   - Receives notifications via Firebase Cloud Messaging (FCM) or Socket.IO
   - Displays FaceTime call notifications
   - Requests FaceTime links from server
   - Launches external applications to handle calls

3. **External Application (Browser/FaceTime Handler)**
   - Handles actual audio/video streaming
   - Manages WebRTC connections
   - Displays call interface

### Critical Architectural Finding

**The BlueBubbles Android client does NOT handle FaceTime audio/video streaming directly.**

Instead, it:
- Acts as a notification relay
- Generates FaceTime web links via server API
- Opens links in external applications (browsers)
- The actual call happens OUTSIDE the BlueBubbles app

### Component Communication Diagram

```mermaid
graph TB
    subgraph "macOS Server"
        PS[Private API Monitor]
        API[REST API Server]
        FCM_SERVER[FCM Sender]
        SOCKET_SERVER[Socket.IO Server]
    end

    subgraph "Android Client"
        FCM_CLIENT[FCM Receiver]
        SOCKET_CLIENT[Socket.IO Client]
        WORKER[DartWorker]
        FLUTTER[Flutter App]
        NOTIF[Notification Service]
        HTTP[HTTP Service]
    end

    subgraph "External"
        BROWSER[Web Browser]
        FACETIME_WEB[FaceTime Web]
    end

    PS -->|Detects Call| FCM_SERVER
    PS -->|Detects Call| SOCKET_SERVER
    PS -->|Provides Link| API

    FCM_SERVER -->|Push Notification| FCM_CLIENT
    SOCKET_SERVER -->|WebSocket Event| SOCKET_CLIENT

    FCM_CLIENT -->|Creates Work| WORKER
    WORKER -->|Method Call| FLUTTER
    SOCKET_CLIENT -->|Event| FLUTTER

    FLUTTER -->|Create Notification| NOTIF
    FLUTTER -->|Request Link| HTTP
    HTTP -->|POST /facetime/answer| API
    API -->|Returns Link| HTTP
    HTTP -->|Opens Link| BROWSER
    BROWSER -->|Loads| FACETIME_WEB
```

---

## Incoming FaceTime Call Flow

### Overview
When someone initiates a FaceTime call to a BlueBubbles user, the server detects it via Private APIs and notifies the Android client.

### Sequence Diagram - FCM Path (Android Background)

```mermaid
sequenceDiagram
    participant MAC as macOS Server<br/>(Private API)
    participant FCM as Firebase Cloud<br/>Messaging
    participant ANDROID as Android OS<br/>(BlueBubbles)
    participant FIREBASE_SVC as BlueBubbles<br/>FirebaseMessagingService
    participant WORK_MGR as DartWorkManager
    participant DART_WORKER as DartWorker
    participant FLUTTER as Flutter Engine
    participant ACTION_HANDLER as ActionHandler
    participant NOTIF_SVC as NotificationsService
    participant METHOD_CHANNEL as MethodChannelService
    participant INTENT_SVC as IntentsService
    participant ANDROID_NOTIF as Android Notification<br/>System

    Note over MAC: User receives<br/>FaceTime call
    MAC->>MAC: Private API detects<br/>incoming call
    MAC->>MAC: Extract call data:<br/>- UUID<br/>- Caller address<br/>- isAudio flag
    MAC->>FCM: Send push notification<br/>type: "ft-call-status-changed"<br/>data: {uuid, status_id: 4, ...}

    FCM->>ANDROID: Deliver FCM message
    ANDROID->>FIREBASE_SVC: onMessageReceived(RemoteMessage)

    Note over FIREBASE_SVC: BlueBubblesFirebaseMessagingService.kt:19
    FIREBASE_SVC->>FIREBASE_SVC: Extract type = "ft-call-status-changed"
    FIREBASE_SVC->>WORK_MGR: createWorker(context, type, data)

    Note over WORK_MGR: DartWorkManager.kt:19
    WORK_MGR->>WORK_MGR: Create OneTimeWorkRequest<br/>with expedited execution
    WORK_MGR->>DART_WORKER: Enqueue work

    Note over DART_WORKER: DartWorker.kt:45
    DART_WORKER->>DART_WORKER: startWork()
    DART_WORKER->>DART_WORKER: Check if MainActivity<br/>engine exists

    alt MainActivity Engine Available
        DART_WORKER->>FLUTTER: Use MainActivity engine
    else No Engine
        DART_WORKER->>FLUTTER: Initialize new FlutterEngine
        FLUTTER->>DART_WORKER: Engine ready
    end

    DART_WORKER->>METHOD_CHANNEL: invokeMethod("ft-call-status-changed", data)

    Note over METHOD_CHANNEL: method_channel_service.dart:309
    METHOD_CHANNEL->>METHOD_CHANNEL: case "ft-call-status-changed"
    METHOD_CHANNEL->>METHOD_CHANNEL: Check if app is alive (ls.isAlive)

    alt App Not Alive (Background)
        METHOD_CHANNEL->>METHOD_CHANNEL: Parse ServerPayload
        METHOD_CHANNEL->>ACTION_HANDLER: handleFaceTimeStatusChange(data)

        Note over ACTION_HANDLER: action_handler.dart:401
        ACTION_HANDLER->>ACTION_HANDLER: Check status_id

        alt status_id == 4 (Incoming Call)
            ACTION_HANDLER->>ACTION_HANDLER: handleIncomingFaceTimeCall(data)

            Note over ACTION_HANDLER: action_handler.dart:411
            ACTION_HANDLER->>ACTION_HANDLER: Extract:<br/>- callUuid<br/>- caller address<br/>- isAudio flag
            ACTION_HANDLER->>ACTION_HANDLER: Load contact info<br/>from ContactsService
            ACTION_HANDLER->>NOTIF_SVC: createIncomingFaceTimeNotification(<br/>callUuid, caller, avatar, isAudio)

            Note over NOTIF_SVC: notifications_service.dart:222
            NOTIF_SVC->>NOTIF_SVC: Generate notification text:<br/>"Answer FaceTime Audio/Video Call"
            NOTIF_SVC->>NOTIF_SVC: Convert callUuid to numeric ID
            NOTIF_SVC->>METHOD_CHANNEL: invokeMethod("create-incoming-facetime-notification", {<br/>channel_id, notification_id,<br/>title, body, caller, call_uuid,<br/>caller_avatar})

            METHOD_CHANNEL->>ANDROID: MethodCallHandler.methodCallHandler()

            Note over ANDROID: MethodCallHandler.kt:68
            ANDROID->>ANDROID: Route to CreateIncomingFaceTimeNotification

            Note over ANDROID: CreateIncomingFaceTimeNotification.kt:25
            ANDROID->>ANDROID: Extract parameters
            ANDROID->>ANDROID: Build Person object with caller info
            ANDROID->>ANDROID: Create Answer PendingIntent:<br/>- Opens MainActivity<br/>- Type: "AnswerFaceTime"<br/>- Extra: callUuid, answer=true
            ANDROID->>ANDROID: Create Decline PendingIntent:<br/>- Broadcasts to InternalIntentReceiver<br/>- Type: "DeleteNotification"
            ANDROID->>ANDROID: Build NotificationCompat:<br/>- Category: CALL<br/>- Priority: MAX<br/>- Ongoing: true<br/>- Timeout: 30 seconds
            ANDROID->>ANDROID_NOTIF: notificationManager.notify(<br/>tag: "NEW_FACETIME_NOTIFICATION",<br/>id: notification_id)

            ANDROID_NOTIF->>ANDROID_NOTIF: Display heads-up notification<br/>with Answer/Ignore buttons
        end

        alt status_id == 6 (Call Ended)
            ACTION_HANDLER->>ACTION_HANDLER: hideFaceTimeOverlay(uuid)
        end
    end

    DART_WORKER->>DART_WORKER: Work complete
    DART_WORKER->>DART_WORKER: Schedule engine cleanup (5s delay)
```

### Sequence Diagram - Socket.IO Path (Web/Desktop)

```mermaid
sequenceDiagram
    participant MAC as macOS Server<br/>(Private API)
    participant SOCKET_SERVER as Socket.IO Server
    participant SOCKET_CLIENT as Socket.IO Client<br/>(Flutter)
    participant ACTION_HANDLER as ActionHandler
    participant NOTIF_SVC as NotificationsService
    participant FACETIME_HELPER as FaceTime Helpers
    participant UI as Flutter UI

    Note over MAC: User receives<br/>FaceTime call
    MAC->>MAC: Private API detects call
    MAC->>SOCKET_SERVER: Emit "ft-call-status-changed"<br/>event with data

    SOCKET_SERVER->>SOCKET_CLIENT: WebSocket message

    Note over SOCKET_CLIENT: socket_service.dart:92
    SOCKET_CLIENT->>SOCKET_CLIENT: on("ft-call-status-changed")
    SOCKET_CLIENT->>ACTION_HANDLER: handleEvent("ft-call-status-changed", data)

    Note over ACTION_HANDLER: action_handler.dart:517
    ACTION_HANDLER->>ACTION_HANDLER: handleFaceTimeStatusChange(data)
    ACTION_HANDLER->>ACTION_HANDLER: Check status_id == 4
    ACTION_HANDLER->>ACTION_HANDLER: handleIncomingFaceTimeCall(data)
    ACTION_HANDLER->>ACTION_HANDLER: Load contact info

    alt App Not Alive (!ls.isAlive)
        ACTION_HANDLER->>NOTIF_SVC: createIncomingFaceTimeNotification()

        alt Desktop
            NOTIF_SVC->>FACETIME_HELPER: showFaceTimeOverlay()
        end
    else App Alive (ls.isAlive)
        ACTION_HANDLER->>FACETIME_HELPER: showFaceTimeOverlay(<br/>callUuid, caller, avatar, isAudio)

        Note over FACETIME_HELPER: facetime_helpers.dart:29
        FACETIME_HELPER->>FACETIME_HELPER: Apply redacted mode if enabled
        FACETIME_HELPER->>FACETIME_HELPER: Load default avatar if needed
        FACETIME_HELPER->>FACETIME_HELPER: Clip avatar to circle
        FACETIME_HELPER->>UI: showDialog() with AlertDialog
        UI->>UI: Display modal with:<br/>- Caller avatar<br/>- Caller name<br/>- "Incoming FaceTime Audio/Video Call"<br/>- Accept button (green)<br/>- Ignore button (red)
    end
```

### Flow Chart - Incoming Call Processing

```mermaid
flowchart TD
    START([FaceTime Call Received<br/>on macOS Server]) --> DETECT[Private API Detects Call]
    DETECT --> EXTRACT[Extract Call Data:<br/>UUID, Caller, isAudio]
    EXTRACT --> PLATFORM{Platform?}

    PLATFORM -->|Android| FCM_SEND[Send FCM Push:<br/>type: ft-call-status-changed<br/>status_id: 4]
    PLATFORM -->|Web/Desktop| SOCKET_SEND[Emit Socket.IO Event:<br/>ft-call-status-changed]

    FCM_SEND --> FCM_RECEIVE[Android FCM Receiver]
    FCM_RECEIVE --> FIREBASE_SVC[BlueBubblesFirebaseMessagingService<br/>.onMessageReceived]
    FIREBASE_SVC --> CREATE_WORKER[DartWorkManager.createWorker<br/>method: ft-call-status-changed]
    CREATE_WORKER --> WORKER_START[DartWorker.startWork]
    WORKER_START --> ENGINE_CHECK{MainActivity<br/>Engine Exists?}

    ENGINE_CHECK -->|Yes| USE_MAIN[Use MainActivity Engine]
    ENGINE_CHECK -->|No| INIT_ENGINE[Initialize FlutterEngine<br/>Load Dart Callback]
    INIT_ENGINE --> USE_MAIN

    USE_MAIN --> INVOKE_METHOD[MethodChannel.invokeMethod<br/>ft-call-status-changed]

    SOCKET_SEND --> SOCKET_RECEIVE[Socket.IO Client Receives Event]
    SOCKET_RECEIVE --> INVOKE_METHOD

    INVOKE_METHOD --> METHOD_HANDLER[MethodChannelService<br/>Handles Method Call]
    METHOD_HANDLER --> ALIVE_CHECK{App Alive?<br/>ls.isAlive}

    ALIVE_CHECK -->|No - Background| PARSE_PAYLOAD[Parse ServerPayload]
    ALIVE_CHECK -->|Yes - Foreground| SOCKET_HANDLER[ActionHandler<br/>From Socket]

    PARSE_PAYLOAD --> ACTION_HANDLER[ActionHandler<br/>.handleFaceTimeStatusChange]
    SOCKET_HANDLER --> ACTION_HANDLER

    ACTION_HANDLER --> STATUS_CHECK{status_id?}

    STATUS_CHECK -->|4 - Incoming| HANDLE_INCOMING[handleIncomingFaceTimeCall]
    STATUS_CHECK -->|6 - Ended| HIDE_OVERLAY[hideFaceTimeOverlay]
    STATUS_CHECK -->|Other| END_FLOW([End])

    HANDLE_INCOMING --> LOAD_CONTACT[Load Contact Info:<br/>- Avatar<br/>- Display Name]
    LOAD_CONTACT --> ALIVE_CHECK2{App Alive?}

    ALIVE_CHECK2 -->|No| CREATE_NOTIF[createIncomingFaceTimeNotification]
    ALIVE_CHECK2 -->|Yes| SHOW_OVERLAY[showFaceTimeOverlay<br/>Dialog UI]

    CREATE_NOTIF --> PLATFORM_CHECK{Platform?}
    PLATFORM_CHECK -->|Android| INVOKE_NATIVE[Invoke Native Method:<br/>create-incoming-facetime-notification]
    PLATFORM_CHECK -->|Desktop| DESKTOP_NOTIF[Show Desktop Notification]
    PLATFORM_CHECK -->|Web| WEB_NOTIF[Show Web Notification]

    INVOKE_NATIVE --> NATIVE_HANDLER[Android MethodCallHandler<br/>Routes to CreateIncomingFaceTimeNotification]
    NATIVE_HANDLER --> BUILD_NOTIF[Build Android Notification]
    BUILD_NOTIF --> SET_ACTIONS[Set Actions:<br/>1. Answer - Opens MainActivity<br/>2. Ignore - Deletes Notification]
    SET_ACTIONS --> NOTIF_DISPLAY[Display Notification:<br/>Category: CALL<br/>Priority: MAX<br/>Timeout: 30s]

    SHOW_OVERLAY --> DIALOG_DISPLAY[Display Alert Dialog<br/>with Accept/Ignore]

    NOTIF_DISPLAY --> END_FLOW
    DESKTOP_NOTIF --> END_FLOW
    WEB_NOTIF --> END_FLOW
    DIALOG_DISPLAY --> END_FLOW
    HIDE_OVERLAY --> END_FLOW
```

### Key Data Structures

#### FCM Push Notification Payload
```json
{
  "type": "ft-call-status-changed",
  "status_id": 4,
  "uuid": "E621E1F8-C36C-495A-93FC-0C247A3E6E5F",
  "handle": {
    "address": "+1234567890"
  },
  "address": "+1234567890",
  "is_audio": false
}
```

#### Android Notification Parameters
```kotlin
Parameters (CreateIncomingFaceTimeNotification.kt:31-39):
- channel_id: String = "com.bluebubbles.incoming_facetimes"
- notification_id: Int = [derived from UUID]
- call_uuid: String = "E621E1F8-C36C-495A-93FC-0C247A3E6E5F"
- title: String = "John Doe"
- body: String = "Answer FaceTime Video Call"
- caller: String = "John Doe"
- caller_avatar: ByteArray? = [avatar image bytes]
```

#### PendingIntent IDs
```kotlin
Constants.kt:
- pendingIntentAnswerFaceTimeOffset = -100000
- pendingIntentDeclineFaceTimeOffset = -200000

Actual IDs:
- Answer Intent ID = notification_id + (-100000)
- Decline Intent ID = notification_id + (-200000)
```

---

## Answering FaceTime Call Flow

### Overview
When a user taps "Answer" on a FaceTime notification, the app requests a FaceTime web link from the server and opens it in an external application.

### Sequence Diagram - Answer Flow

```mermaid
sequenceDiagram
    participant USER as User
    participant ANDROID_NOTIF as Android Notification
    participant ANDROID_OS as Android OS
    participant MAIN_ACTIVITY as MainActivity
    participant RECEIVE_INTENT as ReceiveIntent Plugin
    participant INTENT_SVC as IntentsService
    participant HTTP_SVC as HttpService
    participant SERVER as BlueBubbles Server<br/>(macOS)
    participant SERVER_API as Private API<br/>(macOS)
    participant BROWSER as External Browser
    participant FACETIME_WEB as FaceTime Web

    USER->>ANDROID_NOTIF: Tap "Answer" button

    Note over ANDROID_NOTIF: PendingIntent triggered
    ANDROID_NOTIF->>ANDROID_OS: Start Activity with Intent:<br/>Type: "AnswerFaceTime"<br/>Extra: {callUuid, answer: true, caller}

    ANDROID_OS->>MAIN_ACTIVITY: onCreate/onNewIntent

    Note over MAIN_ACTIVITY: MainActivity.kt
    MAIN_ACTIVITY->>MAIN_ACTIVITY: Flutter engine starts
    MAIN_ACTIVITY->>RECEIVE_INTENT: Intent available via plugin

    RECEIVE_INTENT->>INTENT_SVC: receivedIntentStream

    Note over INTENT_SVC: intents_service.dart:35
    INTENT_SVC->>INTENT_SVC: handleIntent(intent)
    INTENT_SVC->>INTENT_SVC: Check intent.extra["callUuid"]
    INTENT_SVC->>INTENT_SVC: Check intent.extra["answer"] == true

    Note over INTENT_SVC: intents_service.dart:106
    INTENT_SVC->>INTENT_SVC: answerFaceTime(callUuid)

    Note over INTENT_SVC: intents_service.dart:115
    INTENT_SVC->>USER: Show AlertDialog:<br/>"Generating link for call..."<br/>CircularProgressIndicator

    INTENT_SVC->>HTTP_SVC: answerFaceTime(callUuid)

    Note over HTTP_SVC: http_service.dart:981
    HTTP_SVC->>HTTP_SVC: Build request:<br/>POST /api/v1/facetime/answer/{callUuid}
    HTTP_SVC->>HTTP_SVC: Add auth params:<br/>guid = guidAuthKey
    HTTP_SVC->>SERVER: POST /api/v1/facetime/answer/{callUuid}<br/>?guid=<auth_key>

    Note over SERVER: Server processes request
    SERVER->>SERVER_API: Call Private API to generate<br/>FaceTime web link
    SERVER_API->>SERVER_API: Generate FaceTime link for UUID
    SERVER_API->>SERVER: Return link
    SERVER->>HTTP_SVC: Response 200 OK<br/>{data: {link: "https://facetime.apple.com/join#v=1&p=..."}}

    HTTP_SVC->>INTENT_SVC: Return Response with link

    INTENT_SVC->>INTENT_SVC: Extract link from response.data["data"]["link"]
    INTENT_SVC->>USER: Dismiss loading dialog

    alt Link Retrieved Successfully
        INTENT_SVC->>INTENT_SVC: Parse link as Uri
        INTENT_SVC->>BROWSER: launchUrl(Uri.parse(link),<br/>mode: LaunchMode.externalApplication)

        ANDROID_OS->>BROWSER: Open external browser
        BROWSER->>FACETIME_WEB: Load FaceTime web URL
        FACETIME_WEB->>FACETIME_WEB: Initialize WebRTC connection
        FACETIME_WEB->>FACETIME_WEB: Request camera/microphone permissions
        FACETIME_WEB->>USER: Display FaceTime call interface

        Note over FACETIME_WEB: User is now in FaceTime call<br/>handled by browser
    else Link Generation Failed
        INTENT_SVC->>USER: showSnackbar(<br/>"Failed to answer FaceTime",<br/>"Unable to generate FaceTime link!")
    end
```

### Flow Chart - Answer Process

```mermaid
flowchart TD
    START([User Taps Answer Button]) --> PENDING_INTENT[PendingIntent Triggered]
    PENDING_INTENT --> START_ACTIVITY[Start MainActivity with Intent:<br/>Type: AnswerFaceTime<br/>callUuid, answer=true]

    START_ACTIVITY --> MAIN_ACTIVITY_INIT[MainActivity Initializes]
    MAIN_ACTIVITY_INIT --> FLUTTER_ENGINE[Flutter Engine Starts]
    FLUTTER_ENGINE --> RECEIVE_INTENT[ReceiveIntent Plugin<br/>Delivers Intent]

    RECEIVE_INTENT --> INTENT_STREAM[receivedIntentStream]
    INTENT_STREAM --> HANDLE_INTENT[IntentsService.handleIntent]

    HANDLE_INTENT --> CHECK_UUID{Has callUuid<br/>Extra?}
    CHECK_UUID -->|No| END_FLOW([End])
    CHECK_UUID -->|Yes| CHECK_ANSWER{answer Extra<br/>== true?}

    CHECK_ANSWER -->|No| SHOW_OVERLAY[showFaceTimeOverlay<br/>Display Dialog]
    CHECK_ANSWER -->|Yes| ANSWER_FLOW[answerFaceTime Function]

    ANSWER_FLOW --> SHOW_LOADING[Show Loading Dialog:<br/>Generating link for call...]
    SHOW_LOADING --> HIDE_OVERLAY_CALL[hideFaceTimeOverlay<br/>Clear any existing overlay]

    HIDE_OVERLAY_CALL --> HTTP_CALL[HttpService.answerFaceTime<br/>POST /api/v1/facetime/answer/{callUuid}]
    HTTP_CALL --> ADD_AUTH[Add Query Param:<br/>guid = guidAuthKey]
    ADD_AUTH --> SEND_REQUEST[dio.post to server]

    SEND_REQUEST --> RESPONSE_CHECK{Response<br/>Success?}

    RESPONSE_CHECK -->|Error/Timeout| DISMISS_LOADING_ERR[Dismiss Loading Dialog]
    RESPONSE_CHECK -->|Success 200| PARSE_RESPONSE[Parse Response JSON]

    DISMISS_LOADING_ERR --> SHOW_ERROR[showSnackbar:<br/>Failed to answer FaceTime<br/>Unable to generate link]
    SHOW_ERROR --> END_FLOW

    PARSE_RESPONSE --> EXTRACT_LINK[Extract:<br/>link = response.data.data.link]
    EXTRACT_LINK --> LINK_CHECK{Link<br/>Present?}

    LINK_CHECK -->|No/Null| DISMISS_LOADING_ERR
    LINK_CHECK -->|Yes| DISMISS_LOADING_OK[Dismiss Loading Dialog]

    DISMISS_LOADING_OK --> PLATFORM_CHECK{Platform?}

    PLATFORM_CHECK -->|Mobile| LAUNCH_EXTERNAL[launchUrl(Uri.parse(link),<br/>mode: LaunchMode.externalApplication)]
    PLATFORM_CHECK -->|Web| WEB_TODO[TODO: Implement web FaceTime]

    LAUNCH_EXTERNAL --> OS_HANDLER[Android OS Handles URL]
    OS_HANDLER --> BROWSER_OPEN[Open Default Browser]
    BROWSER_OPEN --> LOAD_FACETIME[Load FaceTime Web URL]
    LOAD_FACETIME --> WEBRTC_INIT[Initialize WebRTC Connection]
    WEBRTC_INIT --> PERMISSIONS[Request Camera/Mic Permissions]
    PERMISSIONS --> CALL_UI[Display FaceTime Call UI]
    CALL_UI --> USER_IN_CALL([User in FaceTime Call<br/>via Browser])

    SHOW_OVERLAY --> USER_CHOICE{User Action?}
    USER_CHOICE -->|Accept| ANSWER_FLOW
    USER_CHOICE -->|Ignore| HIDE_OVERLAY_IGNORE[hideFaceTimeOverlay]
    HIDE_OVERLAY_IGNORE --> END_FLOW

    WEB_TODO --> END_FLOW
```

### API Request Details

#### HTTP Request
```
Method: POST
URL: https://[server-address]/api/v1/facetime/answer/E621E1F8-C36C-495A-93FC-0C247A3E6E5F
Query Parameters:
  - guid: [guidAuthKey from settings]
Headers:
  - [Custom headers from settings]
  - Content-Type: application/json
Body: {}
```

#### HTTP Response (Success)
```json
{
  "status": 200,
  "message": "Success",
  "data": {
    "link": "https://facetime.apple.com/join#v=1&p=ABC123XYZ&k=DEF456UVW"
  }
}
```

#### HTTP Response (Error)
```json
{
  "status": 500,
  "message": "Failed to generate FaceTime link",
  "error": "Call not found or expired"
}
```

### Key Code References

#### IntentsService.answerFaceTime
**File**: `lib/services/backend/java_dart_interop/intents_service.dart:115-158`

```dart
Future<void> answerFaceTime(String callUuid) async {
  // Show loading dialog
  showDialog(
    context: Get.context!,
    builder: (BuildContext context) {
      return AlertDialog(
        title: Text("Generating link for call..."),
        content: CircularProgressIndicator(),
      );
    }
  );

  // Clear any existing overlay
  hideFaceTimeOverlay(callUuid);

  // Request link from server
  String? link;
  try {
    final call = await http.answerFaceTime(callUuid);
    link = call.data?["data"]?["link"];
  } catch (_) {}

  // Dismiss loading dialog
  if (Get.context != null) {
    Navigator.of(Get.context!).pop();
  }

  // Handle response
  if (link == null) {
    return showSnackbar("Failed to answer FaceTime", "Unable to generate FaceTime link!");
  }

  // Launch external browser
  if (!kIsWeb) {
    await launchUrl(Uri.parse(link), mode: LaunchMode.externalApplication);
  }
}
```

#### HttpService.answerFaceTime
**File**: `lib/services/network/http_service.dart:981-991`

```dart
Future<Response> answerFaceTime(String callUuid, {CancelToken? cancelToken}) async {
  return runApiGuarded(() async {
    final response = await dio.post(
      "$apiRoot/facetime/answer/$callUuid",
      queryParameters: buildQueryParams(),
      data: {},
      cancelToken: cancelToken
    );
    return returnSuccessOrError(response);
  });
}
```

---

## Call Status Change Flow

### Overview
The server sends status updates for FaceTime calls. Status ID 4 indicates an incoming call, while status ID 6 indicates the call has ended.

### Status ID Codes

```
Status IDs (observed):
- 4: Incoming call (ringing)
- 6: Call ended/terminated
```

### Sequence Diagram - Status Change

```mermaid
sequenceDiagram
    participant SERVER as BlueBubbles Server
    participant FCM as Firebase/Socket
    participant CLIENT as Android Client
    participant ACTION_HANDLER as ActionHandler
    participant FACETIME_HELPER as FaceTime Helpers

    Note over SERVER: Call status changes
    SERVER->>FCM: Send event:<br/>ft-call-status-changed<br/>{uuid, status_id}
    FCM->>CLIENT: Deliver notification
    CLIENT->>ACTION_HANDLER: handleFaceTimeStatusChange(data)

    ACTION_HANDLER->>ACTION_HANDLER: Check status_id

    alt status_id == 4
        ACTION_HANDLER->>ACTION_HANDLER: handleIncomingFaceTimeCall(data)
        Note over ACTION_HANDLER: Creates notification/overlay
    else status_id == 6
        ACTION_HANDLER->>FACETIME_HELPER: hideFaceTimeOverlay(uuid)
        FACETIME_HELPER->>FACETIME_HELPER: Clear notification
        FACETIME_HELPER->>FACETIME_HELPER: Remove dialog overlay
    end
```

### Flow Chart - Status Change Handling

```mermaid
flowchart TD
    START([Status Change Event Received]) --> PARSE[Parse Event Data:<br/>uuid, status_id]
    PARSE --> HANDLER[ActionHandler.handleFaceTimeStatusChange]
    HANDLER --> STATUS_CHECK{status_id?}

    STATUS_CHECK -->|4 - Incoming| INCOMING[handleIncomingFaceTimeCall]
    STATUS_CHECK -->|6 - Ended| ENDED[hideFaceTimeOverlay]
    STATUS_CHECK -->|Other| UNKNOWN[Log unknown status]

    INCOMING --> SHOW_NOTIF[Show Notification/Overlay]
    SHOW_NOTIF --> END([End])

    ENDED --> CLEAR_NOTIF[notif.clearFaceTimeNotification<br/>callUuid]
    CLEAR_NOTIF --> CHECK_OVERLAY{Overlay Exists<br/>in Map?}
    CHECK_OVERLAY -->|Yes| REMOVE_ROUTE[Get.removeRoute<br/>Remove dialog]
    CHECK_OVERLAY -->|No| REMOVE_MAP[Remove from<br/>faceTimeOverlays map]
    REMOVE_ROUTE --> REMOVE_MAP
    REMOVE_MAP --> END

    UNKNOWN --> END
```

### Code Reference - Status Change Handler

**File**: `lib/services/backend/action_handler.dart:401-409`

```dart
Future<void> handleFaceTimeStatusChange(Map<String, dynamic> data) async {
  if (data["status_id"] == null) return;
  final int statusId = data["status_id"] as int;

  if (statusId == 4) {
    // Incoming call
    await ActionHandler().handleIncomingFaceTimeCall(data);
  } else if (statusId == 6 && data["uuid"] != null) {
    // Call ended
    hideFaceTimeOverlay(data["uuid"]!);
  }
}
```

**File**: `lib/helpers/ui/facetime_helpers.dart:19-25`

```dart
void hideFaceTimeOverlay(String callUuid) {
  // Clear the notification
  notif.clearFaceTimeNotification(callUuid);

  // Remove the dialog overlay if it exists
  if (faceTimeOverlays.containsKey(callUuid)) {
    Get.removeRoute(faceTimeOverlays[callUuid]!);
    faceTimeOverlays.remove(callUuid);
  }
}
```

---

## Component Details

### Android Native Components

#### 1. BlueBubblesFirebaseMessagingService

**File**: `android/app/src/main/kotlin/com/bluebubbles/messaging/services/firebase/BlueBubblesFirebaseMessagingService.kt`

**Purpose**: Receives FCM push notifications from server

**Key Methods**:
- `onMessageReceived(message: RemoteMessage)` - Line 19
  - Extracts message type from `message.data["type"]`
  - Creates DartWorker to process the event
  - Optionally sends event to Tasker if configured

**FCM Message Structure**:
```kotlin
message.data = HashMap<String, String> {
  "type": "ft-call-status-changed",
  "status_id": "4",
  "uuid": "E621E1F8-...",
  "handle": "{\"address\":\"+1234567890\"}",
  "address": "+1234567890",
  "is_audio": "false"
}
```

#### 2. DartWorkManager

**File**: `android/app/src/main/kotlin/com/bluebubbles/messaging/services/backend_ui_interop/DartWorkManager.kt`

**Purpose**: Creates background workers to process FCM events

**Key Methods**:
- `createWorker(context, method, arguments, callback)` - Line 19
  - Serializes arguments to JSON using Gson
  - Creates OneTimeWorkRequest with expedited execution policy
  - Enqueues work with WorkManager
  - Observes completion and runs callback

**Work Configuration**:
```kotlin
OneTimeWorkRequest.Builder(DartWorker::class.java)
  .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
  .setInputData(Data.Builder()
    .putString("method", method)
    .putString("data", gson.toJson(arguments))
    .build())
  .addTag(Constants.dartWorkerTag)
  .build()
```

#### 3. DartWorker

**File**: `android/app/src/main/kotlin/com/bluebubbles/messaging/services/backend_ui_interop/DartWorker.kt`

**Purpose**: Executes Dart code in background to process FCM events

**Key Methods**:
- `startWork()` - Line 45
  - Checks if MainActivity engine is available
  - Initializes new FlutterEngine if needed
  - Invokes method on Flutter side via MethodChannel
  - Schedules engine cleanup after 5 seconds

**Engine Initialization** (Line 118-145):
```kotlin
private suspend fun initNewEngine() {
  // Ensure Flutter is initialized
  FlutterMain.startInitialization(applicationContext)
  FlutterMain.ensureInitializationComplete(applicationContext, null)

  // Create new Flutter engine
  workerEngine = FlutterEngine(applicationContext)

  // Set up MethodChannel
  MethodChannel(workerEngine!!.dartExecutor.binaryMessenger, Constants.methodChannel)
    .setMethodCallHandler { call, result ->
      if (call.method == "ready") {
        // Engine ready
      } else {
        MethodCallHandler().methodCallHandler(call, result, applicationContext)
      }
    }

  // Execute Dart callback
  val callbackInfo = FlutterCallbackInformation.lookupCallbackInformation(
    applicationContext.getSharedPreferences("FlutterSharedPreferences", 0)
      .getLong("flutter.backgroundCallbackHandle", -1)
  )
  val callback = DartExecutor.DartCallback(applicationContext.assets, info.flutterAssetsDir, callbackInfo)
  workerEngine!!.dartExecutor.executeDartCallback(callback)
}
```

#### 4. CreateIncomingFaceTimeNotification

**File**: `android/app/src/main/kotlin/com/bluebubbles/messaging/services/notifications/CreateIncomingFaceTimeNotification.kt`

**Purpose**: Creates and displays Android notification for incoming FaceTime calls

**Key Methods**:
- `handleMethodCall(call, result, context)` - Line 25

**Parameters Received** (Lines 31-41):
```kotlin
val channelId: String = call.argument("channel_id")!!
val notificationId: Int = call.argument("notification_id")!!
val callUuid: String? = call.argument("call_uuid")
val title: String = call.argument("title")!!
val body: String = call.argument("body")!!
val callerName: String = call.argument("caller")!!
val callerIcon: ByteArray? = call.argument("caller_avatar")
```

**Notification Actions**:

1. **Answer Action** (Lines 67-79):
```kotlin
val answerIntent = PendingIntent.getActivity(
  context,
  notificationId + Constants.pendingIntentAnswerFaceTimeOffset, // -100000
  Intent(context, MainActivity::class.java)
    .putExtras(extras)
    .putExtra("answer", true)
    .putExtra("caller", callerName)
    .setType("AnswerFaceTime"),
  PendingIntent.FLAG_IMMUTABLE
)
```

2. **Decline Action** (Lines 82-92):
```kotlin
val declineIntent = PendingIntent.getBroadcast(
  context,
  notificationId + Constants.pendingIntentDeclineFaceTimeOffset, // -200000
  Intent(context, InternalIntentReceiver::class.java)
    .putExtra("notificationId", notificationId)
    .setType("DeleteNotification"),
  PendingIntent.FLAG_IMMUTABLE
)
```

**Notification Properties** (Lines 94-115):
```kotlin
val notificationBuilder = NotificationCompat.Builder(context, channelId)
  .setSmallIcon(R.mipmap.ic_stat_icon)
  .setAutoCancel(true)
  .setOngoing(true)  // Cannot be dismissed
  .setCategory(NotificationCompat.CATEGORY_CALL)
  .setPriority(NotificationCompat.PRIORITY_MAX)
  .setContentTitle(title)
  .setContentText(body)
  .addPerson(caller)
  .setColor(4888294)  // Blue color
  .addAction(answerAction)
  .addAction(declineAction)
  .setTimeoutAfter(30000)  // Clear after 30 seconds
```

#### 5. MainActivity

**File**: `android/app/src/main/kotlin/com/bluebubbles/messaging/MainActivity.kt`

**Purpose**: Main Flutter activity, receives intents from notifications

**Key Points**:
- Extends FlutterFragmentActivity
- Configures Flutter engine and MethodChannel
- Does NOT explicitly handle FaceTime intents
- Intent extras are read by Flutter's ReceiveIntent plugin

**MethodChannel Setup** (Lines 22-24):
```kotlin
MethodChannel(flutterEngine.dartExecutor.binaryMessenger, Constants.methodChannel)
  .setMethodCallHandler { call, result ->
    MethodCallHandler().methodCallHandler(call, result, this)
  }
```

### Flutter/Dart Components

#### 1. IntentsService

**File**: `lib/services/backend/java_dart_interop/intents_service.dart`

**Purpose**: Handles Android intents, including FaceTime answer intents

**Key Methods**:

**init()** - Line 29:
- Sets up intent stream listener
- Uses ReceiveIntent plugin
- Handles initial intent and stream

**handleIntent(Intent?)** - Line 48:
```dart
void handleIntent(Intent? intent) async {
  if (intent == null) return;

  switch (intent.action) {
    // Handle various action types
    default:
      if (intent.extra?["callUuid"] != null) {
        await StartupTasks.waitForUI();
        if (intent.extra?["answer"] == true) {
          // User tapped Answer button
          await answerFaceTime(intent.extra?["callUuid"]!);
        } else {
          // User tapped notification body
          await showFaceTimeOverlay(
            intent.extra?["callUuid"],
            intent.extra?["caller"],
            null,
            false
          );
        }
      }
  }
}
```

**answerFaceTime(String)** - Line 115:
```dart
Future<void> answerFaceTime(String callUuid) async {
  if (Get.context != null) {
    showDialog(
      context: Get.context!,
      builder: (BuildContext context) {
        return AlertDialog(
          backgroundColor: context.theme.colorScheme.properSurface,
          title: Text("Generating link for call..."),
          content: Container(
            height: 70,
            child: Center(
              child: CircularProgressIndicator(),
            ),
          ),
        );
      }
    );
    hideFaceTimeOverlay(callUuid);
  }

  String? link;
  try {
    final call = await http.answerFaceTime(callUuid);
    link = call.data?["data"]?["link"];
  } catch (_) {}

  if (Get.context != null) {
    Navigator.of(Get.context!).pop();
  }

  if (link == null) {
    return showSnackbar("Failed to answer FaceTime", "Unable to generate FaceTime link!");
  }

  if (!kIsWeb) {
    await launchUrl(Uri.parse(link), mode: LaunchMode.externalApplication);
  } else if (kIsWeb) {
    // TODO: Implement web FaceTime
  }
}
```

#### 2. ActionHandler

**File**: `lib/services/backend/action_handler.dart`

**Purpose**: Central event handler for server events

**Key Methods**:

**handleEvent(String, Map, String)** - Line 455:
- Main event router
- Handles "ft-call-status-changed" and "incoming-facetime" events

**handleFaceTimeStatusChange(Map)** - Line 401:
```dart
Future<void> handleFaceTimeStatusChange(Map<String, dynamic> data) async {
  if (data["status_id"] == null) return;
  final int statusId = data["status_id"] as int;

  if (statusId == 4) {
    // Incoming call
    await ActionHandler().handleIncomingFaceTimeCall(data);
  } else if (statusId == 6 && data["uuid"] != null) {
    // Call ended
    hideFaceTimeOverlay(data["uuid"]!);
  }
}
```

**handleIncomingFaceTimeCall(Map)** - Line 411:
```dart
Future<void> handleIncomingFaceTimeCall(Map<String, dynamic> data) async {
  Logger.info("Handling incoming FaceTime call");
  await cs.init();  // Initialize contacts service

  final callUuid = data["uuid"];
  String? address = data["handle"]?["address"];
  String caller = data["address"] ?? "Unknown Number";
  bool isAudio = data["is_audio"];
  Uint8List? chatIcon;

  // Find the contact info for the caller
  if (address != null) {
    Contact? contact = cs.getContact(address);
    chatIcon = contact?.avatar;
    caller = contact?.displayName ?? caller;
  }

  if (!ls.isAlive) {
    // App not alive, create notification
    if (kIsDesktop) {
      await showFaceTimeOverlay(callUuid, caller, chatIcon, isAudio);
    }
    await notif.createIncomingFaceTimeNotification(callUuid, caller, chatIcon, isAudio);
  } else {
    // App alive, show overlay
    await showFaceTimeOverlay(callUuid, caller, chatIcon, isAudio);
  }
}
```

**handleIncomingFaceTimeCallLegacy(Map)** - Line 438:
```dart
Future<void> handleIncomingFaceTimeCallLegacy(Map<String, dynamic> data) async {
  Logger.info("Handling incoming FaceTime call (legacy)");
  await cs.init();
  String? address = data["caller"];
  String? caller = address;
  Uint8List? chatIcon;

  // Find contact info
  if (address != null) {
    Contact? contact = cs.getContact(address);
    chatIcon = contact?.avatar;
    caller = contact?.displayName ?? caller;
    await notif.createIncomingFaceTimeNotification(null, caller!, chatIcon, false);
  }
}
```

#### 3. NotificationsService

**File**: `lib/services/backend/notifications/notifications_service.dart`

**Purpose**: Manages all notifications including FaceTime

**Constants** (Lines 34-38):
```dart
static const String FACETIME_CHANNEL = "com.bluebubbles.incoming_facetimes";
static const String NEW_FACETIME_TAG = "com.bluebubbles.messaging.NEW_FACETIME_NOTIFICATION";
```

**Key Methods**:

**createIncomingFaceTimeNotification()** - Line 222:
```dart
Future<void> createIncomingFaceTimeNotification(
  String? callUuid,
  String caller,
  Uint8List? chatIcon,
  bool isAudio
) async {
  // Set notification defaults
  String title = caller;
  String text = "${callUuid == null ? "Incoming" : "Answer"} FaceTime ${isAudio ? 'Audio' : 'Video'} Call";
  chatIcon ??= (await rootBundle.load("assets/images/person64.png")).buffer.asUint8List();

  if (kIsWeb && Notification.permission == "granted") {
    // Web notification
    final notif = Notification(title, body: text, icon: "data:image/png;base64,${base64Encode(chatIcon)}", tag: callUuid);
    if (callUuid != null) {
      notif.onClick.listen((event) async {
        await intents.answerFaceTime(callUuid);
      });
    }
  } else if (kIsDesktop) {
    // Desktop notification
    _lock.synchronized(() async => await showPersistentDesktopFaceTimeNotif(callUuid, caller, chatIcon, isAudio));
  } else {
    // Android notification
    final numeric = callUuid?.numericOnly();
    await mcs.invokeMethod("create-incoming-facetime-notification", {
      "channel_id": FACETIME_CHANNEL,
      "notification_id": numeric != null ? int.parse(numeric.substring(0, min(8, numeric.length))) : Random().nextInt(9998) + 1,
      "title": title,
      "body": text,
      "caller_avatar": chatIcon,
      "caller": caller,
      "call_uuid": callUuid
    });
  }
}
```

**clearFaceTimeNotification()** - Line 251:
```dart
Future<void> clearFaceTimeNotification(String callUuid) async {
  if (kIsDesktop) {
    await clearDesktopFaceTimeNotif(callUuid);
  } else if (!kIsWeb) {
    final numeric = callUuid.numericOnly();
    mcs.invokeMethod(
      "delete-notification",
      {
        "notification_id": int.parse(numeric.substring(0, min(8, numeric.length))),
        "notification_tag": NEW_FACETIME_TAG
      }
    );
  }
}
```

#### 4. FaceTime Helpers

**File**: `lib/helpers/ui/facetime_helpers.dart`

**Purpose**: UI helpers for FaceTime overlays

**Global State**:
```dart
Map<String, Route> faceTimeOverlays = {};  // Map from call uuid to overlay route
```

**Key Functions**:

**hideFaceTimeOverlay()** - Line 19:
```dart
void hideFaceTimeOverlay(String callUuid) {
  notif.clearFaceTimeNotification(callUuid);
  if (faceTimeOverlays.containsKey(callUuid)) {
    Get.removeRoute(faceTimeOverlays[callUuid]!);
    faceTimeOverlays.remove(callUuid);
  }
}
```

**showFaceTimeOverlay()** - Line 29:
```dart
Future<void> showFaceTimeOverlay(
  String callUuid,
  String caller,
  Uint8List? chatIcon,
  bool isAudio
) async {
  // Apply redacted mode if enabled
  if (ss.settings.redactedMode.value && ss.settings.hideContactInfo.value) {
    if (chatIcon != null) chatIcon = null;
    caller = faker.person.name();
  }

  // Load default avatar if needed
  chatIcon ??= (await rootBundle.load("assets/images/person64.png")).buffer.asUint8List();
  chatIcon = await clip(chatIcon, size: 256, circle: true);

  // Close existing overlay
  hideFaceTimeOverlay(callUuid);

  // Show dialog
  showDialog(
    context: Get.context!,
    barrierDismissible: false,
    builder: (_) {
      return BackdropFilter(
        filter: ImageFilter.blur(sigmaX: 2, sigmaY: 2),
        child: AlertDialog(
          icon: Image.memory(chatIcon!, width: 48, height: 48),
          title: Text(caller),
          content: Text(
            "Incoming FaceTime ${isAudio ? "Audio" : "Video"} Call",
            textAlign: TextAlign.center,
          ),
          actionsAlignment: MainAxisAlignment.center,
          actions: [
            // Accept button
            MaterialButton(
              color: Colors.green.withOpacity(0.2),
              child: Column(
                children: [
                  Icon(Icons.call_outlined, color: Colors.green),
                  const Text("Accept"),
                ],
              ),
              onPressed: () async {
                await intents.answerFaceTime(callUuid);
              },
            ),
            const SizedBox(width: 16.0),
            // Ignore button
            MaterialButton(
              color: Colors.red.withOpacity(0.2),
              child: Column(
                children: [
                  Icon(Icons.call_end_outlined, color: Colors.red),
                  const Text("Ignore"),
                ],
              ),
              onPressed: () {
                hideFaceTimeOverlay(callUuid);
              },
            ),
          ],
        ),
      );
    }
  ).then((_) => faceTimeOverlays.remove(callUuid));

  // Save dialog as overlay route
  faceTimeOverlays[callUuid] = Get.rawRoute!;
}
```

#### 5. HttpService

**File**: `lib/services/network/http_service.dart`

**Purpose**: HTTP client for server API calls

**API Configuration** (Lines 19-30):
```dart
String get origin => originOverride ?? (Uri.parse(ss.settings.serverAddress.value).hasScheme ? Uri.parse(ss.settings.serverAddress.value).origin : '');
String get apiRoot => "$origin/api/v1";

Map<String, dynamic> buildQueryParams([Map<String, dynamic> params = const {}]) {
  if (params.isEmpty) {
    params = {};
  }
  params['guid'] = ss.settings.guidAuthKey.value;
  return params;
}
```

**answerFaceTime()** - Line 981:
```dart
/// Answers a facetime call with the given [callUuid].
/// The response is a data object with a `link` key that contains the link to the call.
Future<Response> answerFaceTime(String callUuid, {CancelToken? cancelToken}) async {
  return runApiGuarded(() async {
    final response = await dio.post(
      "$apiRoot/facetime/answer/$callUuid",
      queryParameters: buildQueryParams(),
      data: {},
      cancelToken: cancelToken
    );
    return returnSuccessOrError(response);
  });
}
```

#### 6. SocketService

**File**: `lib/services/network/socket_service.dart`

**Purpose**: Socket.IO client for real-time server communication

**Key Methods**:

**startSocket()** - Line 57:
```dart
void startSocket() {
  OptionBuilder options = OptionBuilder()
    .setQuery({"guid": password})
    .setTransports(['websocket', 'polling'])
    .setExtraHeaders(http.headers)
    .disableAutoConnect()
    .enableReconnection();
  socket = io(serverAddress, options.build());

  // Connection handlers
  socket.onConnect((data) => handleStatusUpdate(SocketState.connected, data));
  socket.onDisconnect((data) => handleStatusUpdate(SocketState.disconnected, data));
  socket.onError((data) => handleStatusUpdate(SocketState.error, data));

  // Custom events - only for web/desktop (Android uses FCM)
  if (kIsWeb || kIsDesktop) {
    socket.on("incoming-facetime", (data) => ah.handleEvent("incoming-facetime", jsonDecode(data), 'DartSocket'));
  }

  // Events for all platforms
  socket.on("ft-call-status-changed", (data) => ah.handleEvent("ft-call-status-changed", data, 'DartSocket'));

  socket.connect();
}
```

**Event Listeners** (Lines 84-92):
```dart
// Only listen to these events from socket on web/desktop (FCM handles on Android)
if (kIsWeb || kIsDesktop) {
  socket.on("group-name-change", (data) => ah.handleEvent("group-name-change", data, 'DartSocket'));
  socket.on("participant-removed", (data) => ah.handleEvent("participant-removed", data, 'DartSocket'));
  socket.on("participant-added", (data) => ah.handleEvent("participant-added", data, 'DartSocket'));
  socket.on("participant-left", (data) => ah.handleEvent("participant-left", data, 'DartSocket'));
  socket.on("incoming-facetime", (data) => ah.handleEvent("incoming-facetime", jsonDecode(data), 'DartSocket'));
}

socket.on("ft-call-status-changed", (data) => ah.handleEvent("ft-call-status-changed", data, 'DartSocket'));
```

#### 7. MethodChannelService

**File**: `lib/services/backend/java_dart_interop/method_channel_service.dart`

**Purpose**: Handles method calls from Android native code (via DartWorker)

**Event Handlers**:

**"incoming-facetime"** - Line 295:
```dart
case "incoming-facetime":
  await Database.waitForInit();
  Logger.info("Received legacy incoming facetime from FCM");
  try {
    Map<String, dynamic>? data = arguments;
    if (!isNullOrEmpty(data)) {
      final payload = ServerPayload.fromJson(data!);
      await ActionHandler().handleIncomingFaceTimeCallLegacy(payload.data);
    }
  } catch (e, s) {
    return Future.error(e, s);
  }
  return Future.value(true);
```

**"ft-call-status-changed"** - Line 309:
```dart
case "ft-call-status-changed":
  if (ls.isAlive) return Future.value(true);  // Skip if app is alive
  await Database.waitForInit();
  Logger.info("Received facetime call status change from FCM");

  try {
    Map<String, dynamic>? data = arguments;
    if (!isNullOrEmpty(data)) {
      final payload = ServerPayload.fromJson(data!);
      await ActionHandler().handleFaceTimeStatusChange(payload.data);
    }
  } catch (e, s) {
    return Future.error(e, s);
  }

  return Future.value(true);
```

---

## Private API Integration

### Server-Side Private API

The BlueBubbles server (running on macOS) uses Apple's private APIs to:

1. **Detect Incoming FaceTime Calls**
   - Monitors system for FaceTime call events
   - Extracts call metadata (UUID, caller, audio/video type)

2. **Generate FaceTime Web Links**
   - Creates shareable FaceTime links
   - Links can be opened in web browsers
   - Uses Apple's FaceTime web infrastructure

3. **Send Notifications**
   - Pushes call data to clients via FCM or Socket.IO
   - Includes call UUID, caller information, and call type

### Client-Side Private API Usage

**Important**: The Android client does NOT directly use any Private APIs.

All Private API functionality is handled server-side:
- Call detection: Server
- Link generation: Server
- Media streaming: External browser (via FaceTime web)

### Private API Requirements

From the settings panel (`lib/app/layouts/settings/pages/advanced/private_api_panel.dart`):

**Server Requirements**:
- Private API bundle must be installed on macOS server
- Server setting `ss.settings.serverPrivateAPI.value` must be `true`

**Client Settings**:
- `ss.settings.enablePrivateAPI.value` - Enables Private API features in client
- `ss.settings.privateAPISend.value` - Use Private API for sending messages
- `ss.settings.privateAPIAttachmentSend.value` - Use Private API for attachments

**FaceTime-Specific**:
- No specific FaceTime settings in client
- FaceTime availability depends solely on server having Private API enabled

### Data Flow - Private API Context

```mermaid
graph LR
    subgraph "macOS Server with Private API"
        IMESSAGE_D[imagent/IMCore]
        FACETIME_D[avconferenced/FaceTime.framework]
        BB_SERVER[BlueBubbles Server]
        PRIVATE_API_BUNDLE[Private API Bundle]
    end

    subgraph "Android Client"
        BB_CLIENT[BlueBubbles Client]
    end

    subgraph "External"
        BROWSER[Web Browser]
    end

    IMESSAGE_D -.Private API.-> PRIVATE_API_BUNDLE
    FACETIME_D -.Private API.-> PRIVATE_API_BUNDLE
    PRIVATE_API_BUNDLE --> BB_SERVER
    BB_SERVER -->|FCM/Socket| BB_CLIENT
    BB_SERVER -->|FaceTime Link| BB_CLIENT
    BB_CLIENT -->|Opens Link| BROWSER
```

### Server API Endpoint

**Endpoint**: `/api/v1/facetime/answer/:callUuid`

**Method**: POST

**Authentication**: Query parameter `guid` (GUID auth key)

**Request**:
```
POST /api/v1/facetime/answer/E621E1F8-C36C-495A-93FC-0C247A3E6E5F?guid=<auth-key>
Body: {}
```

**Response (Success)**:
```json
{
  "status": 200,
  "message": "Success",
  "data": {
    "link": "https://facetime.apple.com/join#v=1&p=<token>&k=<key>"
  }
}
```

**Response (Error)**:
```json
{
  "status": 500,
  "message": "Failed to generate FaceTime link",
  "error": "Call not found or expired"
}
```

### FaceTime Web Link Format

The server generates links in Apple's FaceTime web format:

```
https://facetime.apple.com/join#v=1&p=<participant-token>&k=<key>
```

**Components**:
- `v=1`: Version
- `p=<token>`: Participant token (encrypted/encoded)
- `k=<key>`: Key for call authentication

These links can be opened in:
- Safari (iOS/macOS)
- Chrome (Android/Desktop)
- Edge (Desktop)
- Other modern browsers with WebRTC support

---

## Summary

### Key Architectural Points

1. **No Direct Media Handling**
   - Android client does NOT handle audio/video streaming
   - Media is handled by external browsers via FaceTime web

2. **Notification Relay Pattern**
   - Client receives notifications via FCM/Socket.IO
   - Displays Android notifications with Answer/Ignore actions
   - Requests FaceTime links from server
   - Opens links externally

3. **Server-Centric Architecture**
   - All Private API usage is server-side
   - Server detects calls and generates links
   - Server sends notifications to clients

4. **External Application Dependency**
   - Actual FaceTime call happens in web browser
   - Browser handles WebRTC connections
   - Browser manages audio/video streaming

### Communication Paths

**Android Background (FCM)**:
```
Server → FCM → FirebaseMessagingService → DartWorker → Flutter → Notification
```

**Web/Desktop (Socket.IO)**:
```
Server → Socket.IO → SocketService → ActionHandler → Notification/Overlay
```

**Answer Flow**:
```
User → Notification → MainActivity → IntentsService → HttpService → Server → Link → Browser
```

### File Location Summary

**Android Native**:
- `android/app/src/main/kotlin/com/bluebubbles/messaging/services/firebase/BlueBubblesFirebaseMessagingService.kt`
- `android/app/src/main/kotlin/com/bluebubbles/messaging/services/backend_ui_interop/DartWorkManager.kt`
- `android/app/src/main/kotlin/com/bluebubbles/messaging/services/backend_ui_interop/DartWorker.kt`
- `android/app/src/main/kotlin/com/bluebubbles/messaging/services/notifications/CreateIncomingFaceTimeNotification.kt`
- `android/app/src/main/kotlin/com/bluebubbles/messaging/MainActivity.kt`
- `android/app/src/main/kotlin/com/bluebubbles/messaging/Constants.kt`

**Flutter/Dart**:
- `lib/services/backend/java_dart_interop/intents_service.dart`
- `lib/services/backend/action_handler.dart`
- `lib/services/backend/notifications/notifications_service.dart`
- `lib/helpers/ui/facetime_helpers.dart`
- `lib/services/network/http_service.dart`
- `lib/services/network/socket_service.dart`
- `lib/services/backend/java_dart_interop/method_channel_service.dart`

---

## Conclusion

The BlueBubbles Android FaceTime implementation is a **notification relay system** rather than a full WebRTC implementation. It cleverly leverages:

1. Apple's Private APIs (server-side only)
2. Firebase Cloud Messaging for push notifications
3. FaceTime web links for actual call handling
4. External browsers for media streaming

This architecture allows Android users to participate in FaceTime calls without requiring complex WebRTC implementation in the client app, while maintaining a seamless user experience.
