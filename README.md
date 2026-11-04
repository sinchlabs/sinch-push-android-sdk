# Sinch Push SDK for Android

Receive Sinch push notifications in your Android app over Firebase Cloud Messaging (FCM).
The SDK registers the device with Sinch, shows Sinch notifications, and lets you take over how they are handled.

> **Using Sinch Chat?** You don't need to add this SDK yourself. `sinch-chat-sdk` 1.0.0 and later already
> include it and set it up for you. Follow the [Sinch Chat SDK](https://github.com/sinchlabs/sinch-chat-android-sdk)
> instructions instead.

## Requirements

- Android 7.0 (API 24) or later. Compile with API 35 or later, with Android Gradle Plugin 8 and JDK 17.
- Kotlin 2.3 or later if your app uses Kotlin.
- A Firebase project with Cloud Messaging enabled, and your app registered in it.
- A Sinch project configured for push, and these values from it:
  - **Project ID**, **Client ID** and **Config ID**
  - the **region** of your project (EU1 or US1)
- Your Firebase project's FCM credentials registered with your Sinch project, so Sinch can send pushes to
  your app. Ask your Sinch contact if this hasn't been done yet.

## Installation

### 1. Add the repository

`settings.gradle`:

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url "https://raw.githubusercontent.com/sinchlabs/sinch-push-android-sdk/master/releases" }
    }
}
```

`settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://raw.githubusercontent.com/sinchlabs/sinch-push-android-sdk/master/releases") }
    }
}
```

The repository is public, so no credentials are needed.

### 2. Add the dependency

`app/build.gradle`:

```groovy
dependencies {
    implementation 'com.sinch.push.sdk:sinch-push-sdk:2.0.0'
}
```

The SDK brings in `firebase-messaging`, so you don't have to add it separately.

### 3. Set up Firebase

If your app doesn't use Firebase yet, follow
[Add Firebase to your Android project](https://firebase.google.com/docs/android/setup): add the
`com.google.gms.google-services` Gradle plugin and put your own `google-services.json` in the `app/` folder.

R8 / ProGuard rules ship with the SDK, so release builds need no extra configuration.

## Usage

### 1. Initialize the SDK

Initialize once, as early as possible, in `Application.onCreate`. Pass the current FCM token. FCM only calls
`onNewToken` when the token changes, so this is how a device that already has a token gets registered.

```kotlin
import android.app.Application
import com.google.firebase.messaging.FirebaseMessaging
import com.sinch.push.sdk.SinchPushSDK

class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        SinchPushSDK.initialize(this)
        FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
            SinchPushSDK.push.onNewToken(token)
        }
    }
}
```

Register the class in `AndroidManifest.xml` with `<application android:name=".MyApplication" ...>`.

### 2. Forward FCM events

Create a `FirebaseMessagingService` and hand the token and incoming messages to the SDK.
`onMessageReceived` returns `true` when the message was a Sinch message and was handled, and `false` when it is
one of your own messages.

```kotlin
import com.google.firebase.messaging.FirebaseMessagingService
import com.google.firebase.messaging.RemoteMessage
import com.sinch.push.sdk.SinchPushSDK

class MyFirebaseMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        SinchPushSDK.push.onNewToken(token)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        if (!SinchPushSDK.push.onMessageReceived(message)) {
            // Not a Sinch message: handle it yourself.
        }
    }
}
```

`AndroidManifest.xml`:

```xml
<service
    android:name=".MyFirebaseMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

Forward each token **once**. If another library's service already forwards it, don't forward it again.

### 3. Ask for the notification permission

On Android 13 (API 33) and later, notifications are only shown once the user grants `POST_NOTIFICATIONS`.
The SDK declares the permission, but your app has to ask for it at a suitable moment:

```kotlin
if (Build.VERSION.SDK_INT >= 33 &&
    checkSelfPermission(Manifest.permission.POST_NOTIFICATIONS) != PackageManager.PERMISSION_GRANTED
) {
    requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), REQUEST_CODE_NOTIFICATIONS)
}
```

### 4. Set the user's identity

Call `setIdentity` as soon as you know who the user is, for example after login. This is when the device is
subscribed to push. You can call it again, for example when another user logs in.

```kotlin
import com.sinch.chat.sdk.SinchConfig
import com.sinch.chat.sdk.SinchIdentity
import com.sinch.chat.sdk.SinchRegion
import com.sinch.push.sdk.SinchPushSDK

val config = SinchConfig(
    clientID = "YOUR_CLIENT_ID",
    projectID = "YOUR_PROJECT_ID",
    configID = "YOUR_CONFIG_ID",
    region = SinchRegion.EU1, // or SinchRegion.US1
    language = "",            // e.g. "en"; empty uses the default
)

SinchPushSDK.setIdentity(config, SinchIdentity.SelfSigned(userID, signedUserID)) { result ->
    if (result.isSuccess) {
        // The device receives pushes for this user.
    } else {
        Log.e("Push", "setIdentity failed", result.exceptionOrNull())
    }
}
```

The configuration types live in the `com.sinch.chat.sdk` package because they are shared with the Chat SDK.

**Identity types:**

- `SinchIdentity.SelfSigned(userID, signedUserID)`: a user of your system. `signedUserID` is the
  HMAC-SHA512 of `userID`, keyed with your Sinch signing secret, hex encoded in lowercase.
  **Compute it on your backend** and send it to the app. Never ship the signing secret in the app.
- `SinchIdentity.Anonymous`: a user without an account.

The identity is kept in memory, so set it again on every app start once the user is known.

### 5. Remove the identity on logout

`removeIdentity` unsubscribes the device and forgets the user. The callback is always called.

```kotlin
import com.sinch.push.sdk.SinchRemoveIdentityResult

SinchPushSDK.removeIdentity { result ->
    when (result) {
        is SinchRemoveIdentityResult.Success -> { /* logged out */ }
        is SinchRemoveIdentityResult.Error -> Log.e("Push", "removeIdentity failed", result.error)
    }
}
```

## Customizing notifications

### Default behaviour

A Sinch message that carries a notification is shown on the `sinch_chat_channel` channel. The SDK creates the
channel if it is missing. Tapping the notification opens your app's launcher activity. The small icon is your
app's icon.

### Use your own channel

Set the channel ID before the first notification. The SDK creates the channel if it doesn't exist yet. If you
want your own name or importance, create the channel yourself first.

```kotlin
SinchPushSDK.push.notificationChannelId = "my_channel"
```

### Handle messages yourself

Register a `SinchPushAddressee` to take over Sinch messages, for example to open a specific screen. Addressees
are asked in the order they were registered. The first one that returns `true` from `canHandle` and from `handle`
owns the message. If none does, the default notification is shown.

```kotlin
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import com.google.firebase.messaging.RemoteMessage
import com.sinch.push.sdk.IsHandled
import com.sinch.push.sdk.SinchPushAddressee
import com.sinch.push.sdk.SinchPushSDK

class OrderUpdatesAddressee : SinchPushAddressee {

    override fun canHandle(data: Map<String, String>): Boolean = data["topic_id"] != null

    override fun handle(context: Context, message: RemoteMessage): IsHandled {
        val intent = Intent(context, OrderActivity::class.java)
            .putExtra("topic_id", message.data["topic_id"])
        val tap = PendingIntent.getActivity(
            context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        SinchPushSDK.push.showNotification(context, message, tap)
        return true // return true without showing anything to drop the message
    }
}

// Once, e.g. in Application.onCreate:
SinchPushSDK.push.register(OrderUpdatesAddressee())
```

`unregister(addressee)` removes it again.

## Regions

| Region | `SinchRegion` |
| --- | --- |
| Europe | `SinchRegion.EU1` |
| United States | `SinchRegion.US1` |
| Other / custom environment | `SinchRegion.Custom(chatHost, pushHost)`, with hosts from your Sinch contact |

Use the region your Sinch project was created in.

## Push relay (preview, off by default)

Besides FCM, the SDK can receive pushes over a WebSocket relay while your app is in the foreground. The relay is
**not deployed to the production environments yet**, so it is off unless you turn it on:

```kotlin
val config = SinchConfig(clientID, projectID, configID, region, language, relay = SinchRelayConfig(enabled = true))
```

A message that arrives over both FCM and the relay is shown once. `SinchPushSDK.push.relayState` (a `StateFlow`) and
`relayMessages` report the relay's state and messages; `setRelayEnabled(false)` turns it off at runtime.

The AAR's manifest declares the relay's foreground service (`dataSync`) and the `FOREGROUND_SERVICE_DATA_SYNC`
permission, also while the relay is off. If you publish on Google Play, declare that foreground service type in Play
Console, or remove both from your merged manifest with `tools:node="remove"` while you don't use the relay.

## Debug logging

```kotlin
SinchPushLogs.ENABLED = true // off by default; don't enable in production builds
```

Logs use the Logcat tag `SINCH_PUSH`.

## Troubleshooting

- **No notification arrives.**
  - Check that `setIdentity` succeeded.
  - Check that your `FirebaseMessagingService` is declared in the manifest.
  - On Android 13+, check that the notification permission was granted.
  - Check that your Firebase project's FCM credentials are registered with your Sinch project.
- **`onMessageReceived` returns `false` for a Sinch push.** Only messages sent by Sinch (`owner` = `sinch` in
  the data payload) are handled. Others are left to your app.
- **Duplicate registrations.** Forward `onNewToken` from one place only.

## Changelog

### 2.0.0

- **Breaking:** requires Android 7.0 (API 24), compileSdk 35, AGP 8 / JDK 17 and Kotlin 2.3 or later.
- **Breaking:** the manifest declares the push relay's foreground service and `FOREGROUND_SERVICE_DATA_SYNC`
  (see [Push relay](#push-relay-preview-off-by-default)).
- `SinchRegion.EU1` and `SinchRegion.US1` use the same chat and push hosts as the iOS SDK.
- Push relay (preview, off by default): pushes over a WebSocket while the app is in the foreground, deduplicated
  against FCM.
- Data-only Sinch pushes with a `title`/`body` are shown, and delivery reports are sent to Sinch.
- The device is subscribed again only when the token or the device info changed.
- Uses firebase-messaging 26.0.0.

### 1.0.0

- First release as a standalone SDK. The push code was previously part of `sinch-chat-sdk`.

## License

Apache License 2.0, see [LICENSE.md](LICENSE.md).
