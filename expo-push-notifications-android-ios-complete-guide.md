# Expo Push Notifications Setup Guide — Android + iOS

This guide explains how to set up **Expo Push Notifications** from the beginning for both:

- **Android** using Firebase Cloud Messaging **FCM V1**
- **iOS** using Apple Push Notification service **APNs**
- **Expo Push Service** as the common notification delivery layer
- **EAS Build** for credentials and native builds

It also includes:

- `.gitignore`
- `.easignore`
- Android Firebase setup
- iOS APNs setup
- App-side notification code
- Receiving notifications
- Sending notifications from a backend
- Push tickets and receipts
- Common errors and troubleshooting

---

# Table of Contents

1. [How Expo Push Notifications Work](#1-how-expo-push-notifications-work)
2. [What You Need to Know First](#2-what-you-need-to-know-first)
3. [Prerequisites](#3-prerequisites)
4. [Install Expo Notification Packages](#4-install-expo-notification-packages)
5. [Add Expo Notifications Plugin](#5-add-expo-notifications-plugin)
6. [Configure App Identifiers](#6-configure-app-identifiers)
7. [Set Up EAS](#7-set-up-eas)
8. [Android Firebase + FCM V1 Setup](#8-android-firebase--fcm-v1-setup)
9. [iOS APNs Setup](#9-ios-apns-setup)
10. [Configure .gitignore and .easignore](#10-configure-gitignore-and-easignore)
11. [Final app.json Example](#11-final-appjson-example)
12. [Create Notification Helper](#12-create-notification-helper)
13. [Use Notifications in App.tsx](#13-use-notifications-in-apptsx)
14. [Receiving Notifications](#14-receiving-notifications)
15. [Send Notifications from Backend](#15-send-notifications-from-backend)
16. [Push Tickets and Push Receipts](#16-push-tickets-and-push-receipts)
17. [Build and Test](#17-build-and-test)
18. [Production Checklist](#18-production-checklist)
19. [Common Issues](#19-common-issues)
20. [Quick Summary](#20-quick-summary)
21. [References](#21-references)

---

# 1. How Expo Push Notifications Work

Expo gives you one shared push notification flow for Android and iOS.

```txt
Your mobile app
→ asks the user for notification permission
→ gets an ExpoPushToken
→ sends that ExpoPushToken to your backend
→ backend sends notification to Expo Push API
→ Expo routes the notification:
   → FCM V1 for Android
   → APNs for iOS
→ user's device receives the notification
```

Your app does **not** usually send directly to Firebase FCM or Apple APNs.

Instead:

```txt
Your backend → Expo Push Service → FCM/APNs → Device
```

This is easier than managing FCM and APNs directly.

---

# 2. What You Need to Know First

Before starting setup, understand these notification concepts.

---

## 2.1 Push Notifications vs Local Notifications

There are two main notification types:

```txt
Push notifications / remote notifications
→ sent from a remote server to the user's device

Local notifications / in-app scheduled notifications
→ created by the app itself and shown locally on the device
```

This guide focuses on **push notifications**.

---

## 2.2 Expo Go Is Not Enough for Push Notifications

For real push notification testing, use:

```txt
- EAS development build
- EAS preview build
- EAS production build
```

Do **not** rely on Expo Go for full push notification setup.

---

## 2.3 App States Matter

Push behavior depends on the app state.

### Foreground

```txt
App is open and visible on screen.
```

The app controls how to handle the notification using:

```ts
Notifications.setNotificationHandler(...)
```

### Background

```txt
App is minimized but still available in the background.
```

The OS usually displays regular notification messages.

### Terminated / Closed

```txt
App has been killed or fully closed.
```

The OS may still display normal notification messages, but background execution behavior is more limited.

Important Android note:

```txt
If the user force-stops the app from Android device settings,
notifications may not work again until the user manually opens the app.
```

Some Android battery optimization settings can also block delivery when the app is closed.

---

## 2.4 Notification Message

A notification message contains visible information like:

```txt
title
subtitle
body
sound
badge
```

Example:

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "title": "New message",
  "body": "You have a new message",
  "sound": "default"
}
```

Use this when you want the user to see a notification.

---

## 2.5 Notification Message with Data Payload

You can send visible notification content plus hidden/custom data.

Example:

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "title": "New message",
  "body": "John sent you a message",
  "data": {
    "conversationId": "conversation_123",
    "messageId": "message_456"
  }
}
```

Your app can later read:

```ts
notification.request.content.data
```

This is useful for navigation after the user taps a notification.

---

## 2.6 Headless Background Notifications

Headless background notifications are data-only notifications that can trigger background processing.

They are more advanced and are not the default recommendation.

Use them only when:

```txt
- you need background JavaScript execution
- you understand platform limitations
- you do not need a visible notification
```

Rule of thumb:

```txt
Use normal notification messages unless you specifically need background processing.
```

---

# 3. Prerequisites

You need:

```txt
- Expo project
- Expo account
- Firebase account
- Apple Developer account for iOS
- EAS CLI
- Physical Android or iOS device recommended
```

You can test push notifications on:

```txt
- Physical Android device
- Android Emulator with Google Play services
- Physical iOS device
- iOS Simulator on Xcode 14+ / iOS 16+
```

For iOS credentials and real iOS push setup, you need a paid Apple Developer account.

---

# 4. Install Expo Notification Packages

From the root of your Expo project, run:

```bash
npx expo install expo-notifications expo-constants
```

What each package does:

```txt
expo-notifications
→ asks for notification permission
→ creates Android notification channels
→ gets the ExpoPushToken
→ receives foreground notifications
→ detects notification taps

expo-constants
→ reads EAS projectId from your Expo config
```

---

# 5. Add Expo Notifications Plugin

Open:

```txt
app.json
```

or:

```txt
app.config.js
```

Add the plugin:

```json
{
  "expo": {
    "plugins": [
      "expo-notifications"
    ]
  }
}
```

If you already have plugins:

```json
{
  "expo": {
    "plugins": [
      "expo-router",
      "expo-notifications"
    ]
  }
}
```

---

# 6. Configure App Identifiers

You need stable identifiers for both platforms.

```txt
Android → package
iOS     → bundleIdentifier
```

Example:

```json
{
  "expo": {
    "name": "YourApp",
    "slug": "your-app",
    "ios": {
      "bundleIdentifier": "com.yourcompany.yourapp"
    },
    "android": {
      "package": "com.yourcompany.yourapp"
    }
  }
}
```

Important:

```txt
Do not keep changing these identifiers.
They are connected to Firebase, Apple, EAS credentials, and store builds.
```

---

# 7. Set Up EAS

Install EAS CLI:

```bash
npm install -g eas-cli
```

Login:

```bash
eas login
```

Initialize EAS:

```bash
eas init
```

Configure EAS builds:

```bash
eas build:configure
```

This should create:

```txt
eas.json
```

It also links the app to an EAS project and gives you a `projectId`.

The `projectId` is needed when requesting the Expo push token:

```ts
Notifications.getExpoPushTokenAsync({ projectId })
```

---

## 7.1 Example eas.json

```json
{
  "cli": {
    "version": ">= 7.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

---

# 8. Android Firebase + FCM V1 Setup

Android push notifications require Firebase Cloud Messaging.

---

## 8.1 Create or Open Firebase Project

Go to:

```txt
https://console.firebase.google.com/
```

Then either:

```txt
Create a new project
```

or:

```txt
Open an existing project
```

---

## 8.2 Add Android App in Firebase

Inside Firebase:

```txt
Project Overview
→ Add app
→ Android
```

Firebase will ask for:

```txt
Android package name
```

Use the same value from `app.json`:

```txt
com.yourcompany.yourapp
```

This must match:

```json
{
  "expo": {
    "android": {
      "package": "com.yourcompany.yourapp"
    }
  }
}
```

If these do not match, push notifications can fail.

---

## 8.3 Download google-services.json

Firebase will generate:

```txt
google-services.json
```

Place it in your project root:

```txt
your-project/
  app.json
  eas.json
  package.json
  google-services.json
```

---

## 8.4 Add googleServicesFile to app.json

Update `app.json`:

```json
{
  "expo": {
    "android": {
      "package": "com.yourcompany.yourapp",
      "googleServicesFile": "./google-services.json"
    }
  }
}
```

This tells Expo/EAS where the Android Firebase config is.

---

## 8.5 Important: google-services.json vs Service Account Key

There are two different Firebase JSON files.

### google-services.json

```txt
Used by the Android app build.
Needed by EAS Build if app.json points to it.
Usually safe to commit, but some teams prefer not to.
```

### Firebase service account private key

```txt
Used by EAS/Expo for FCM V1 push credentials.
Private and sensitive.
Never commit this file.
```

Do not confuse these two files.

---

## 8.6 Generate Firebase Service Account Key

In Firebase:

```txt
Project Settings
→ Service accounts
→ Generate new private key
→ Generate key
```

This downloads a private JSON file.

Example file names:

```txt
your-project-firebase-adminsdk-abc123.json
firebase-service-account.json
service-account.json
```

This file is sensitive.

```txt
Do not commit it.
Do not upload it unnecessarily.
Do not share it.
```

---

## 8.7 Upload FCM V1 Key to EAS

Run:

```bash
eas credentials
```

Choose:

```txt
Android
→ production or development
→ Google Service Account
→ Manage your Google Service Account Key for Push Notifications (FCM V1)
→ Set up a Google Service Account Key for Push Notifications (FCM V1)
→ Upload a new service account key
```

Upload the private Firebase service account JSON.

---

## 8.8 Android Checklist

```txt
[ ] Firebase project created
[ ] Android app added in Firebase
[ ] Firebase Android package matches expo.android.package
[ ] google-services.json downloaded
[ ] google-services.json placed in project
[ ] app.json has android.googleServicesFile
[ ] Firebase service account private key generated
[ ] FCM V1 private key uploaded to EAS
[ ] Private service account key is in .gitignore
[ ] Private service account key is in .easignore
```

---

# 9. iOS APNs Setup

iOS push notifications use Apple Push Notification service.

---

## 9.1 Configure Bundle Identifier

In `app.json`:

```json
{
  "expo": {
    "ios": {
      "bundleIdentifier": "com.yourcompany.yourapp"
    }
  }
}
```

This should match the Apple app identifier managed through EAS/Apple Developer.

---

## 9.2 Apple Developer Account

You need:

```txt
Paid Apple Developer account
```

Without it, iOS push credential setup will not work properly.

---

## 9.3 Register Device for Development Builds

If testing on your iPhone with an internal development build:

```bash
eas device:create
```

Follow the instructions on your phone.

---

## 9.4 Generate APNs Credentials with EAS

Run:

```bash
eas credentials
```

Choose:

```txt
iOS
→ production or development
→ Push Notifications
→ Let EAS manage credentials
```

When prompted:

```txt
Setup Push Notifications for your project?
```

Answer:

```txt
Yes
```

When prompted:

```txt
Generate a new Apple Push Notifications service key?
```

Answer:

```txt
Yes
```

EAS will create or link the APNs key.

---

## 9.5 iOS Checklist

```txt
[ ] Apple Developer account active
[ ] ios.bundleIdentifier configured
[ ] Device registered using eas device:create if using development build
[ ] APNs credentials configured through eas credentials
[ ] Push notifications enabled during EAS setup
[ ] iOS build created using EAS
```

---

# 10. Configure .gitignore and .easignore

Your screenshots mention ignoring files. This is important.

---

## 10.1 Difference Between .gitignore and .easignore

### .gitignore

Controls what gets committed to GitHub.

Use it to prevent secrets from entering your repository.

### .easignore

Controls what gets uploaded to EAS Build.

Use it to prevent secrets or unnecessary files from being uploaded to EAS.

Important:

```txt
If a file is required for the EAS build, do not add it to .easignore.
```

---

## 10.2 Recommended .gitignore

Create or update:

```txt
.gitignore
```

Add:

```gitignore
# Dependencies
node_modules/

# Expo
.expo/
dist/
web-build/

# EAS / build outputs
*.apk
*.aab
*.ipa

# Environment files
.env
.env.local
.env.development
.env.production

# Firebase private service account keys
*-firebase-adminsdk-*.json
firebase-service-account.json
service-account.json

# OS files
.DS_Store
```

---

## 10.3 Should google-services.json Be in .gitignore?

This depends on your team.

### Option A — Commit google-services.json

This is simplest for EAS Build.

```txt
Do not add google-services.json to .gitignore.
Do not add google-services.json to .easignore.
```

### Option B — Ignore google-services.json

If you do not want it in GitHub:

```gitignore
google-services.json
```

But then you must make sure EAS Build still receives it another way.

Warning:

```txt
If app.json points to ./google-services.json
and EAS Build cannot access the file,
your Android build can fail.
```

---

## 10.4 Recommended .easignore

Create:

```txt
.easignore
```

Add:

```gitignore
# Git files
.git/
.github/

# Dependencies
node_modules/

# Environment files
.env
.env.local
.env.development
.env.production

# Firebase private service account keys
*-firebase-adminsdk-*.json
firebase-service-account.json
service-account.json

# Build outputs
*.apk
*.aab
*.ipa
dist/
web-build/

# OS files
.DS_Store
```

---

## 10.5 Do Not Add google-services.json to .easignore by Default

Do **not** add this by default:

```gitignore
google-services.json
```

because EAS needs it when your config has:

```json
{
  "expo": {
    "android": {
      "googleServicesFile": "./google-services.json"
    }
  }
}
```

Only add it to `.easignore` if you are generating it during the build or providing it another way.

---

# 11. Final app.json Example

```json
{
  "expo": {
    "name": "YourApp",
    "slug": "your-app",
    "scheme": "yourapp",

    "plugins": [
      "expo-notifications"
    ],

    "ios": {
      "bundleIdentifier": "com.yourcompany.yourapp"
    },

    "android": {
      "package": "com.yourcompany.yourapp",
      "googleServicesFile": "./google-services.json"
    },

    "extra": {
      "eas": {
        "projectId": "YOUR-EAS-PROJECT-ID"
      }
    }
  }
}
```

Replace:

```txt
com.yourcompany.yourapp
YOUR-EAS-PROJECT-ID
```

with your real values.

---

# 12. Create Notification Helper

Create:

```txt
src/lib/notifications.ts
```

Add:

```ts
import * as Notifications from "expo-notifications";
import Constants from "expo-constants";
import { Platform } from "react-native";

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: true,
    shouldSetBadge: true,
    shouldShowBanner: true,
    shouldShowList: true,
  }),
});

export async function registerForPushNotificationsAsync() {
  if (Platform.OS === "android") {
    await Notifications.setNotificationChannelAsync("default", {
      name: "default",
      importance: Notifications.AndroidImportance.MAX,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: "#FF231F7C",
    });
  }

  const { status: existingStatus } =
    await Notifications.getPermissionsAsync();

  let finalStatus = existingStatus;

  if (existingStatus !== "granted") {
    const { status } =
      await Notifications.requestPermissionsAsync();

    finalStatus = status;
  }

  if (finalStatus !== "granted") {
    throw new Error("Permission not granted for push notifications");
  }

  const projectId =
    Constants.expoConfig?.extra?.eas?.projectId ??
    Constants.easConfig?.projectId;

  if (!projectId) {
    throw new Error("EAS projectId not found");
  }

  const token = await Notifications.getExpoPushTokenAsync({
    projectId,
  });

  return token.data;
}
```

This file does the following:

```txt
1. Creates an Android notification channel
2. Checks current permission status
3. Requests permission if needed
4. Reads the EAS projectId
5. Gets the ExpoPushToken
6. Returns the token
```

---

# 13. Use Notifications in App.tsx

Example:

```tsx
import { useEffect, useState } from "react";
import { Text, View, Platform } from "react-native";
import * as Notifications from "expo-notifications";

import { registerForPushNotificationsAsync } from "./src/lib/notifications";

export default function App() {
  const [expoPushToken, setExpoPushToken] = useState("");
  const [notification, setNotification] =
    useState<Notifications.Notification | undefined>(undefined);

  useEffect(() => {
    registerForPushNotificationsAsync()
      .then((token) => {
        setExpoPushToken(token);

        console.log("Expo Push Token:", token);

        // Send token to backend
        // savePushTokenToBackend(token, Platform.OS);
      })
      .catch((error) => {
        console.error("Push registration error:", error);
      });

    const notificationListener =
      Notifications.addNotificationReceivedListener((notification) => {
        setNotification(notification);
        console.log("Notification received:", notification);
      });

    const responseListener =
      Notifications.addNotificationResponseReceivedListener((response) => {
        console.log("Notification tapped:", response);

        const data = response.notification.request.content.data;
        console.log("Notification data:", data);
      });

    return () => {
      notificationListener.remove();
      responseListener.remove();
    };
  }, []);

  return (
    <View>
      <Text>Expo Push Token:</Text>
      <Text>{expoPushToken}</Text>
    </View>
  );
}
```

---

# 14. Receiving Notifications

Expo provides listeners for notifications.

---

## 14.1 Notification Received Listener

Use this when the app is open and receives a notification.

```ts
const notificationListener =
  Notifications.addNotificationReceivedListener((notification) => {
    console.log("Notification received:", notification);

    const data = notification.request.content.data;
    console.log("Custom data:", data);
  });
```

Use this for:

```txt
- showing custom in-app UI
- updating local state
- refreshing a chat thread
- incrementing unread count
```

---

## 14.2 Notification Response Listener

Use this when the user taps a notification.

```ts
const responseListener =
  Notifications.addNotificationResponseReceivedListener((response) => {
    console.log("Notification tapped:", response);

    const data = response.notification.request.content.data;

    if (data.conversationId) {
      // Navigate to conversation screen
    }
  });
```

Use this for:

```txt
- opening a chat
- opening an order
- opening an appointment
- navigating to a specific screen
```

---

## 14.3 Access Custom Data

If you send:

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "title": "New message",
  "body": "John sent you a message",
  "data": {
    "conversationId": "conversation_123",
    "messageId": "message_456"
  }
}
```

You can read:

```ts
const data = notification.request.content.data;

console.log(data.conversationId);
console.log(data.messageId);
```

---

## 14.4 Foreground Notification Behavior

When the app is open, you decide whether to show notifications.

```ts
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: true,
    shouldSetBadge: true,
    shouldShowBanner: true,
    shouldShowList: true,
  }),
});
```

Meaning:

```txt
shouldPlaySound
→ play notification sound

shouldSetBadge
→ update app badge

shouldShowBanner
→ show banner while app is foregrounded

shouldShowList
→ show in notification list
```

Example if you do not want sound while app is open:

```ts
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: false,
    shouldSetBadge: false,
    shouldShowBanner: true,
    shouldShowList: true,
  }),
});
```

---

## 14.5 Removing Listeners

Always clean up listeners:

```ts
return () => {
  notificationListener.remove();
  responseListener.remove();
};
```

This prevents memory leaks and duplicate handling.

---

## 14.6 Closed App Behavior

When the app is closed:

```txt
- OS controls delivery/display of regular visible notifications
- user tapping the notification opens the app
- response listener can handle the tap after startup
```

On Android, manufacturer battery optimizations may prevent delivery in some closed-app cases.

---

# 15. Send Notifications from Backend

Your backend should send notifications to Expo Push API.

Endpoint:

```txt
https://exp.host/--/api/v2/push/send
```

---

## 15.1 Basic cURL Example

```bash
curl -H "Content-Type: application/json" \
  -X POST "https://exp.host/--/api/v2/push/send" \
  -d '{
    "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
    "sound": "default",
    "title": "Test Notification",
    "body": "Hello from Expo Push Notifications",
    "data": {
      "screen": "home"
    }
  }'
```

---

## 15.2 Message Format

Only `to` is required, but most messages include:

```txt
to
title
body
sound
data
```

Example:

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "title": "Appointment Reminder",
  "body": "Your appointment is tomorrow at 10:00 AM.",
  "sound": "default",
  "data": {
    "type": "appointment_reminder",
    "appointmentId": "appt_123"
  }
}
```

---

## 15.3 Sending Multiple Notifications

You can send an array:

```json
[
  {
    "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
    "title": "Hello",
    "body": "Message one"
  },
  {
    "to": "ExponentPushToken[yyyyyyyyyyyyyyyyyyyyyy]",
    "title": "Hello",
    "body": "Message two"
  }
]
```

---

## 15.4 Save Tokens in Your Backend

When the app gets an ExpoPushToken, send it to your backend.

Example table:

```txt
push_tokens

- id
- user_id
- expo_push_token
- platform
- device_id
- created_at
- updated_at
- disabled_at
```

Example API request:

```ts
await fetch("https://your-api.com/push-tokens", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${authToken}`,
  },
  body: JSON.stringify({
    expoPushToken,
    platform: Platform.OS,
  }),
});
```

---

## 15.5 Send from Node.js Backend

Install the Expo server SDK:

```bash
npm install expo-server-sdk
```

Example:

```ts
import { Expo } from "expo-server-sdk";

const expo = new Expo();

export async function sendPushNotification(
  expoPushToken: string,
  title: string,
  body: string,
  data: Record<string, unknown> = {}
) {
  if (!Expo.isExpoPushToken(expoPushToken)) {
    throw new Error(`Invalid Expo push token: ${expoPushToken}`);
  }

  const messages = [
    {
      to: expoPushToken,
      sound: "default",
      title,
      body,
      data,
    },
  ];

  const chunks = expo.chunkPushNotifications(messages);
  const tickets = [];

  for (const chunk of chunks) {
    const ticketChunk = await expo.sendPushNotificationsAsync(chunk);
    tickets.push(...ticketChunk);
  }

  return tickets;
}
```

---

## 15.6 Reliability Rules

When sending notifications:

```txt
- limit concurrent requests
- retry temporary failures
- use exponential backoff for HTTP 429 or 5xx
- do not retry malformed payloads until fixed
- check receipts after sending
```

The Expo Node SDK helps by chunking messages and limiting concurrency.

---

# 16. Push Tickets and Push Receipts

Expo uses two result stages:

```txt
Push ticket
→ Expo accepted or rejected the notification request

Push receipt
→ Expo later reports whether FCM/APNs accepted the notification
```

---

## 16.1 Push Ticket

After sending a notification, Expo returns a ticket.

Example success:

```json
{
  "data": [
    {
      "status": "ok",
      "id": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
    }
  ]
}
```

Important:

```txt
status: ok means Expo accepted the notification.
It does not mean the user received it.
```

---

## 16.2 Push Ticket Error

Example:

```json
{
  "data": [
    {
      "status": "error",
      "message": "\"ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]\" is not a registered push notification recipient",
      "details": {
        "error": "DeviceNotRegistered"
      }
    }
  ]
}
```

If you see:

```txt
DeviceNotRegistered
```

stop sending to that token.

---

## 16.3 Check Push Receipts

Use the ticket `id` values to check receipts later.

Endpoint:

```txt
https://exp.host/--/api/v2/push/getReceipts
```

Example:

```bash
curl -H "Content-Type: application/json" \
  -X POST "https://exp.host/--/api/v2/push/getReceipts" \
  -d '{
    "ids": [
      "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
      "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
    ]
  }'
```

Example success:

```json
{
  "data": {
    "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX": {
      "status": "ok"
    }
  }
}
```

---

## 16.4 When to Check Receipts

A good backend flow:

```txt
1. Send notification
2. Store ticket IDs
3. Wait around 15 minutes
4. Fetch receipts
5. Remove invalid tokens
6. Log/report errors
```

Receipts are cleared after a period of time, so do not wait too long.

---

## 16.5 Receipt Errors to Handle

Common errors:

```txt
DeviceNotRegistered
→ remove the token from your database

MessageTooBig
→ reduce payload size

MessageRateExceeded
→ slow down sends

InvalidCredentials
→ check FCM/APNs/EAS credentials
```

---

# 17. Build and Test

---

## 17.1 Build Android Development App

```bash
eas build --platform android --profile development
```

Install it on your Android device.

---

## 17.2 Build iOS Development App

```bash
eas build --platform ios --profile development
```

Install it on your iPhone.

---

## 17.3 Build Both Platforms

```bash
eas build --platform all --profile development
```

---

## 17.4 Start Expo

After installing the build:

```bash
npx expo start
```

Open the installed development build.

You should see a token like:

```txt
ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]
```

---

## 17.5 Test with Expo Push Tool

Open:

```txt
https://expo.dev/notifications
```

Paste your token and send a test notification.

---

## 17.6 Test with cURL

```bash
curl -H "Content-Type: application/json" \
  -X POST "https://exp.host/--/api/v2/push/send" \
  -d '{
    "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
    "sound": "default",
    "title": "Test Notification",
    "body": "Hello from Expo",
    "data": {
      "test": true
    }
  }'
```

---

# 18. Production Checklist

---

## 18.1 Android Checklist

```txt
[ ] android.package is final
[ ] Firebase Android app package matches android.package
[ ] google-services.json exists
[ ] app.json points to google-services.json
[ ] Firebase service account private key uploaded to EAS
[ ] Private Firebase key is not committed
[ ] Private Firebase key is not uploaded unnecessarily to EAS Build
[ ] Android notification channel is created in code
[ ] Android EAS build works
[ ] Real Android device receives notification
```

---

## 18.2 iOS Checklist

```txt
[ ] ios.bundleIdentifier is final
[ ] Apple Developer account is active
[ ] Device registered for development build if needed
[ ] APNs key configured in EAS
[ ] Push notifications enabled during EAS setup
[ ] iOS EAS build works
[ ] Real iPhone receives notification
```

---

## 18.3 Backend Checklist

```txt
[ ] ExpoPushToken saved per user/device
[ ] Backend sends to Expo Push API
[ ] Backend stores push ticket IDs
[ ] Backend checks push receipts
[ ] Backend removes DeviceNotRegistered tokens
[ ] Backend retries temporary failures with backoff
[ ] Backend logs malformed payload errors
```

---

# 19. Common Issues

---

## 19.1 EAS projectId Not Found

Error:

```txt
EAS projectId not found
```

Fix:

```bash
eas init
```

Then check `app.json`:

```json
{
  "expo": {
    "extra": {
      "eas": {
        "projectId": "YOUR-EAS-PROJECT-ID"
      }
    }
  }
}
```

---

## 19.2 Android Build Fails Because google-services.json Is Missing

Cause:

```txt
app.json points to ./google-services.json
but EAS Build cannot access it.
```

Fix options:

```txt
Option A:
Commit google-services.json.

Option B:
Provide/generate google-services.json during EAS Build.

Option C:
Remove googleServicesFile until Android Firebase setup is ready.
```

---

## 19.3 Android Notifications Not Showing

Check:

```txt
[ ] Are you using a development or production build?
[ ] Did you create notification channel?
[ ] Does Firebase package match android.package?
[ ] Is google-services.json configured?
[ ] Is FCM V1 key uploaded to EAS?
[ ] Is the app force-stopped in Android settings?
[ ] Are battery optimizations blocking the app?
```

---

## 19.4 iOS Notifications Not Working

Check:

```txt
[ ] Do you have a paid Apple Developer account?
[ ] Is ios.bundleIdentifier correct?
[ ] Did EAS create APNs credentials?
[ ] Did you answer yes to push notification setup?
[ ] Are you using an EAS build, not Expo Go?
[ ] Are notifications enabled in iOS settings?
```

---

## 19.5 Token Is DeviceNotRegistered

Meaning:

```txt
The device can no longer receive notifications for that token.
```

Possible causes:

```txt
- app uninstalled
- notification permission revoked
- token expired or changed
- app reinstalled
```

Fix:

```txt
Remove or disable the token in your backend.
If the app registers again, save the new token.
```

---

## 19.6 Payload Too Large

If you receive:

```txt
MessageTooBig
```

Reduce the size of:

```txt
title
body
data payload
```

Keep notification data small. Store large data on your backend and send only IDs in the notification payload.

---

## 19.7 Accidentally Committed Firebase Private Key

If you committed a Firebase service account private key:

```txt
1. Delete the file from the repo
2. Revoke/delete the key in Firebase or Google Cloud
3. Generate a new service account key
4. Upload the new key to EAS
5. Add key patterns to .gitignore
6. Consider the old key exposed
```

---

# 20. Quick Summary

```txt
1. Install expo-notifications and expo-constants
2. Add expo-notifications plugin
3. Configure android.package
4. Configure ios.bundleIdentifier
5. Run eas init
6. Run eas build:configure
7. Create/open Firebase project
8. Add Android app in Firebase
9. Download google-services.json
10. Add android.googleServicesFile to app.json
11. Generate Firebase service account private key
12. Upload FCM V1 key to EAS
13. Configure iOS APNs credentials through EAS
14. Add private keys to .gitignore
15. Add private keys to .easignore
16. Create src/lib/notifications.ts
17. Request permission and get ExpoPushToken
18. Save ExpoPushToken to backend
19. Build app using EAS
20. Test with Expo Push Tool or cURL
21. Backend sends notifications through Expo Push API
22. Backend checks push tickets and receipts
23. Remove invalid tokens
```

---

# 21. References

- Expo Push Notifications Overview: https://docs.expo.dev/push-notifications/overview/
- What You Need to Know About Notifications: https://docs.expo.dev/push-notifications/what-you-need-to-know/
- Expo Push Notifications Setup: https://docs.expo.dev/push-notifications/push-notifications-setup/
- Send Notifications with the Expo Push Service: https://docs.expo.dev/push-notifications/sending-notifications/
- Handle Incoming Notifications: https://docs.expo.dev/push-notifications/receiving-notifications/
- Android FCM V1 Credentials: https://docs.expo.dev/push-notifications/fcm-credentials/
