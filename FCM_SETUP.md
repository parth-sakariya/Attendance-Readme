# FCM Push Notifications — Setup (P8)

In-app notifications work with **no setup** (the parent portal bell reads them
from Firestore). To also deliver **push** notifications when the app is closed,
follow these steps. **Requires the Blaze (pay-as-you-go) Firebase plan.**

## What's already in the codebase
- **Compose UI** — admins: *Manage → Send Notification* (audience: all parents /
  parents of a group). Writes one doc per recipient under
  `mainTemples/{templeId}/notifications/{id}`.
- **Client token registration** — `lib/app/data/services/push_service.dart`
  requests permission, fetches the FCM token, and stores it on
  `users/{uid}.fcmTokens`. Called automatically on login / session restore, and
  removed on logout. (`firebase_messaging` is already in `pubspec.yaml`.)
- **Cloud Function** — `functions/index.js` → `pushOnNotification` triggers on a
  new notification doc, finds the recipient's user account(s)
  (`users where parentId == notification.parentId`), and pushes to their tokens
  (pruning stale ones).

## One-time setup

1. **Upgrade to Blaze** in the Firebase console (Functions require it).

2. **Deploy the function:**
   ```bash
   cd functions && npm install && cd ..
   firebase deploy --only functions
   ```

3. **Deploy rules/indexes** (also covers P2–P7):
   ```bash
   firebase deploy --only firestore,storage
   ```

4. **Android:** no extra key needed (uses `google-services.json`, already present).

5. **iOS:** upload an **APNs key** (Apple Developer → Keys → enable APNs) to
   Firebase console → Project settings → Cloud Messaging → *Apple app
   configuration*. Enable Push Notifications + Background Modes (Remote
   notifications) in Xcode.

6. **Web:** in Firebase console → Project settings → Cloud Messaging → *Web Push
   certificates*, generate a key pair and paste the public key into
   `PushService.webVapidKey`. Add a `web/firebase-messaging-sw.js` service
   worker (standard FlutterFire web messaging snippet).

## How it flows
```
Admin "Send Notification"
  → NotificationRepository.createForParents()  (1 doc per parent)
    → Firestore create trigger: pushOnNotification
      → users where parentId == doc.parentId  → collect fcmTokens
        → FCM sendEachForMulticast → device push
Parent portal bell renders the same docs (in-app), regardless of push.
```

## Notes
- `PushService` is fully guarded: if a user declines permission, or the web
  VAPID key is missing, no token is stored and only in-app notifications show —
  nothing breaks.
- The function prunes invalid/expired tokens automatically.
