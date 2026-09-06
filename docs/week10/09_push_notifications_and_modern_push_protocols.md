# Topic 09: Push Notifications & Modern Push Protocols



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Topic 09: Push Notifications & Modern Push Protocols**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

---

## 1. The Final Push Problem: What Happens When the Tab Is Closed?

All previous client-update strategies — polling, long polling, SSE, WebSockets — share one critical limitation:

> **They all require the web page to be open in the browser.**

The moment the user closes the tab or navigates away, the connection is gone, the JavaScript stops running, and your app loses all ability to reach that user.

But users expect to be notified even when they're not actively using the app:

- 📬 "You have a new message" (WhatsApp Web, Gmail)
- 🛒 "Your order has been shipped" (Amazon, Flipkart)
- ⚽ "India scored! 150/3 after 20 overs" (live sports apps)
- 💬 "Vinay mentioned you in a comment" (Slack, GitHub)
- 🔔 "Your build passed" (GitHub Actions)

These notifications arrive even when the browser is minimised, the tab is closed, or the device is locked. This is the domain of **Push Notifications**.

---

## 2. The Core Challenge: Standard HTTP Can't Do This

Both SSE and WebSockets require:
- The browser tab to be open
- A JavaScript execution context to be active

Once the tab is closed:
- The `EventSource` connection is torn down
- The `WebSocket` closes
- JavaScript stops executing entirely

To push notifications to a device when no page is open, you need a mechanism that:
1. Operates **at the OS/browser level** — not at the webpage level
2. Maintains a **persistent background connection** to a push server
3. Can **wake up** the browser or app and display a notification

This is exactly what **Service Workers** + the **Web Push Protocol** provide for the web, and what **FCM/APNs** provide for native mobile apps.

---

## 3. The Architecture: Three-Party Push System

All modern push notification systems — web and native — use the same fundamental three-party architecture:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THREE-PARTY PUSH ARCHITECTURE                        │
├─────────────────┬───────────────────────────┬───────────────────────────┤
│   Your Server   │   Push Service (Broker)   │   Client Device/Browser   │
│  (App Server)   │  (FCM / APNs / Browser    │                           │
│                 │   Push Service)            │                           │
└────────┬────────┴────────────┬──────────────┴──────────┬────────────────┘
         │                    │                          │
         │  1. Send push      │                          │
         │  message           │                          │
         │──────────────────► │  2. Deliver to           │
         │                    │  registered device       │
         │                    │─────────────────────────►│
         │                    │                          │  3. Display
         │                    │                          │  notification
         │                    │                          │  (even if
         │                    │                          │  page closed)
```

The **Push Service** (the middle party) is the critical piece:
- Maintained by Google (FCM), Apple (APNs), or the browser vendor (Mozilla, Chrome's push service)
- It maintains **persistent background TCP connections** to every registered client device
- These connections are maintained at the **OS/browser level**, not at the webpage level
- Your server sends a single HTTP request to the push service → it delivers to the device

---

## 4. Web Push — Browser-Based Push Notifications

### The Components

Web push for browsers involves three technologies working together:

| Component | Role |
| :--- | :--- |
| **Service Worker** | Background script that receives push events even when page is closed |
| **Push API** (`PushManager`) | Browser API to subscribe to push notifications |
| **Web Push Protocol** | IETF standard (RFC 8030) defining how app servers talk to push services |
| **VAPID** | Authentication standard (RFC 8292) for identifying your server to the push service |

---

### Step 1: Service Worker Registration

A **Service Worker** is a JavaScript file that the browser runs as a **persistent background process** — independent of any web page. It survives tab closes and browser restarts.

```javascript
// In your main app JavaScript (e.g., app.js)
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
        .then(registration => {
            console.log('Service Worker registered:', registration.scope);
        })
        .catch(err => {
            console.error('Service Worker registration failed:', err);
        });
}
```

The browser downloads and installs `/sw.js` — it now runs in the background.

---

### Step 2: Subscribe to Push Notifications (Push API)

```javascript
// In your main app JavaScript — after service worker is registered
async function subscribeToPush() {
    const registration = await navigator.serviceWorker.ready;

    // Request permission from the user
    const permission = await Notification.requestPermission();
    if (permission !== 'granted') {
        console.log('Push permission denied');
        return;
    }

    // Subscribe using the Push API
    const subscription = await registration.pushManager.subscribe({
        userVisibleOnly: true,          // Must be true — push must result in visible notification
        applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY)  // Your server's public key
    });

    // Send subscription object to your server for storage
    await fetch('/api/push/subscribe', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(subscription)
    });
}
```

The `subscription` object returned by `pushManager.subscribe()` contains:
```json
{
  "endpoint": "https://fcm.googleapis.com/fcm/send/dYp8Xz...",
  "keys": {
    "p256dh": "BNcRdreALRFXTkOOUHK...",
    "auth": "tBHItJI5SVYh..."
  }
}
```

- **`endpoint`**: The URL on the push service (e.g., Google's FCM) where your server POSTs to deliver a message to **this specific device**.
- **`keys`**: Encryption keys for securing the message payload.

Your server stores this subscription in a database, keyed to the user.

---

### Step 3: Your Server Sends a Push Message

When an event occurs (e.g., new message arrives), your server:
1. Looks up the user's stored subscription objects (there may be multiple — phone, laptop, tablet)
2. Makes an HTTP POST request to each subscription's `endpoint` URL, following the **Web Push Protocol**

```python
# Python — using pywebpush library
from pywebpush import webpush, WebPushException
import json

def send_push_notification(subscription_info, message_data):
    try:
        webpush(
            subscription_info=subscription_info,  # The stored subscription object
            data=json.dumps(message_data),
            vapid_private_key="your_vapid_private_key",
            vapid_claims={
                "sub": "mailto:you@yourapp.com"
            }
        )
    except WebPushException as ex:
        print("Push failed:", ex)
        if ex.response.status_code == 410:
            # 410 Gone = subscription expired — remove from DB
            remove_subscription_from_db(subscription_info['endpoint'])

# Usage when a new chat message arrives:
user_subscriptions = db.get_push_subscriptions(user_id=42)
for sub in user_subscriptions:
    send_push_notification(sub, {
        "title": "New Message",
        "body": "Priya sent you a message",
        "icon": "/icon.png",
        "url": "/chat/priya"
    })
```

---

### Step 4: Service Worker Receives & Displays the Notification

The push event is received by the **Service Worker** (which is running in the background even if the page is closed):

```javascript
// sw.js — Service Worker file

// Fired when a push event arrives from the push service
self.addEventListener('push', function(event) {
    const data = event.data.json();  // Parse the payload

    const options = {
        body: data.body,
        icon: data.icon || '/default-icon.png',
        badge: '/badge.png',
        data: { url: data.url },      // Pass URL for click handling
        actions: [
            { action: 'open', title: 'Open' },
            { action: 'dismiss', title: 'Dismiss' }
        ],
        vibrate: [100, 50, 100],       // Vibration pattern (mobile)
        requireInteraction: false      // Auto-dismiss after a while
    };

    // waitUntil ensures the browser doesn't kill SW before notification shows
    event.waitUntil(
        self.registration.showNotification(data.title, options)
    );
});

// Fired when the user clicks the notification
self.addEventListener('notificationclick', function(event) {
    event.notification.close();  // Close the notification

    const url = event.notification.data.url;

    event.waitUntil(
        clients.matchAll({ type: 'window' }).then(windowClients => {
            // If a window is already open, focus it and navigate
            for (const client of windowClients) {
                if (client.url === url && 'focus' in client) {
                    return client.focus();
                }
            }
            // Otherwise, open a new window/tab
            if (clients.openWindow) {
                return clients.openWindow(url);
            }
        })
    );
});
```

---

### The IETF Web Push Protocol (RFC 8030)

The **Web Push Protocol** is an IETF standard that governs how your application server communicates with the browser's push service:

- Defines the HTTP POST request format for sending push messages to an `endpoint` URL
- Specifies **message encryption** (using the client's `p256dh` and `auth` keys) — so the push service cannot read your payload
- Defines **VAPID** (Voluntary Application Server Identification — RFC 8292): your server uses a public/private key pair to authenticate itself to the push service
- Supports **message urgency**: `Urgency: very-low | low | normal | high` — allows the push service to prioritize and defer delivery (e.g., save battery by batching low-urgency notifications)
- Supports **TTL (Time-To-Live)**: if the device is offline, how long should the push service hold the message before discarding it

```
POST https://fcm.googleapis.com/fcm/send/dYp8Xz... HTTP/1.1
TTL: 3600
Urgency: normal
Content-Type: application/octet-stream
Content-Encoding: aes128gcm
Authorization: vapid t=<jwt_token>,k=<public_key>

[encrypted payload bytes]
```

---

## 5. Public Push Notification Providers

### Firebase Cloud Messaging (FCM) — Google

**FCM** (formerly Google Cloud Messaging / GCM) is Google's push notification infrastructure:

- Supports: **Android native apps**, **Chrome browser push**, **Web Push Protocol**
- For Chrome, the `endpoint` URL in a push subscription will be an `fcm.googleapis.com` URL — meaning Chrome routes all web push through FCM
- Also supports **data messages** (silent background wakeup) vs. **notification messages** (visible alerts)
- Free tier is generous; widely used

```
[Your Server] ──── POST to FCM HTTP API ────► [FCM Server]
FCM maintains persistent connections to all Android devices and Chrome browsers
[FCM Server] ──────────────────────────────► [Android Device / Chrome Browser]
```

### Apple Push Notification service (APNs)

**APNs** is Apple's push notification infrastructure, mandatory for all iOS/macOS/Safari notifications:

- Supports: **iOS native apps**, **macOS native apps**, **Safari Web Push** (added properly in Safari 16+ / macOS Ventura)
- Uses **HTTP/2** protocol with mutual TLS authentication
- Requires an Apple Developer account and certificates/keys
- Notification payload is JSON with strict size limits (4KB)

```json
// APNs payload format
{
  "aps": {
    "alert": {
      "title": "New Message",
      "body": "Priya sent you a message"
    },
    "badge": 3,
    "sound": "default",
    "content-available": 1
  },
  "custom_data": { "chat_id": "priya_123" }
}
```

### Device Token Registration Workflow

Both FCM and APNs use a **device token** (equivalent to the web push `endpoint`) to identify a specific app installation on a specific device:

```
Step 1: App starts → requests push permission from OS
Step 2: OS registers with FCM/APNs → receives a unique device token
        e.g., "dYp8XzABCDEF..." (FCM) or "a9d3f2..." (APNs)
Step 3: App sends this token to your server: POST /register-device { token: "..." }
Step 4: Your server stores token → user mapping in DB
Step 5: When event occurs, your server POSTs to FCM/APNs with the token
Step 6: FCM/APNs delivers to the exact device
```

---

## 6. Web Apps vs. Native Apps — Key Distinctions

This is a crucial comparison the course highlights:

| Aspect | Web App (Browser Push) | Native App (FCM/APNs) |
| :--- | :--- | :--- |
| **Technology** | Web Push Protocol + Service Worker | Native SDK (Firebase SDK, APNs framework) |
| **Background connection** | Browser maintains a TCP socket to FCM/Mozilla push | OS maintains a persistent TCP socket to FCM/APNs |
| **Works when app closed** | ✅ Yes (browser must be running, not tab) | ✅ Yes (OS-level — even if app fully closed) |
| **Works when browser closed** | ❌ No | ✅ Yes (OS independent) |
| **Cross-platform** | ✅ Any browser/OS | ❌ Platform-specific (Android vs iOS) |
| **Permission model** | Browser permission prompt | OS permission prompt |
| **Encryption** | VAPID + AES128GCM (E2E encrypted) | TLS to APNs/FCM (provider can read payload) |
| **Delivery guarantees** | Best-effort | Best-effort + retry |

### The Native App Advantage

Native apps (Android, iOS) have a deeper integration: the OS itself maintains a **persistent background TCP socket** to Google's (FCM) or Apple's (APNs) servers. This socket is maintained 24/7 by the OS — even when all user apps are closed. When a push message arrives, the OS wakes the specific app process to handle it.

```
OS Level:
  [Android OS] ───── persistent TCP connection ─────► [Google FCM servers]
  [iOS OS]     ───── persistent TCP connection ─────► [Apple APNs servers]

When push arrives:
  FCM ──────── pushes to Android OS ──────────────► [Android OS wakes app]
  APNs ─────── pushes to iOS OS ──────────────────► [iOS OS wakes app]
```

Web browsers achieve something similar — Chrome maintains a persistent connection to FCM even when no Chrome tabs are open (as long as Chrome is running as a background process). But if the user has fully exited the browser, web push cannot reach them — unlike native apps.

---

## 7. End-to-End Web Push Flow Summary

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE WEB PUSH FLOW                                    │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  SETUP (once):                                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ 1. Browser loads your page                                                 │ │
│  │ 2. App registers Service Worker (/sw.js)                                   │ │
│  │ 3. App calls pushManager.subscribe() → browser contacts push service       │ │
│  │ 4. Push service returns a unique endpoint URL + encryption keys            │ │
│  │ 5. App POSTs this subscription to your server → stored in DB               │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  SENDING (per event):                                                            │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │ 1. Event occurs on your server (new message, order shipped, etc.)          │ │
│  │ 2. Your server fetches user's subscription(s) from DB                      │ │
│  │ 3. Your server encrypts payload + POSTs to each subscription endpoint URL  │ │
│  │ 4. Push service (FCM/Mozilla) delivers to the device                       │ │
│  │ 5. Browser's Service Worker receives 'push' event                          │ │
│  │ 6. Service Worker calls showNotification() → visible OS notification       │ │
│  │ 7. User clicks notification → notificationclick handler → opens page       │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Importance of Push in Modern UX

Push notifications are a critical driver of **user retention and engagement**:

- **Re-engagement**: Users who've left your app can be brought back with a timely, relevant notification.
- **Real-time trust**: "Your payment was received" arriving instantly feels responsive and reliable.
- **Actionability**: OS-level notifications with action buttons (Reply, Dismiss, View) allow users to act without even opening the app.
- **Opt-in and opt-out**: Proper implementation respects user preferences — misconfigured, spammy push is one of the fastest ways to lose users.

Modern best practices:
- Always explain **why** you need permission before triggering the browser prompt (permission denial is permanent without developer override).
- Use **urgency levels** (IETF TTL, FCM priority) — don't interrupt a user at 3 AM for a low-priority marketing notification.
- Handle `410 Gone` responses from push services — these mean the subscription has expired and should be removed from your DB.

---

## 9. Full Stack Summary: All Client Update Strategies

```
┌──────────────────┬──────────────────────────────┬────────────────────────────────┐
│ Strategy         │ Mechanism                    │ Best For                       │
├──────────────────┼──────────────────────────────┼────────────────────────────────┤
│ Fixed Polling    │ Client asks on setInterval   │ Simple prototypes, rare updates│
│ Long Polling     │ Server holds request open    │ Near-real-time, HTTP-only      │
│ SSE              │ Persistent HTTP stream       │ One-way real-time feeds        │
│ WebSockets       │ Full-duplex TCP connection   │ Bidirectional real-time apps   │
│ Web Push (SSW)   │ Service Worker + Push API    │ Browser notifications (closed) │
│ FCM / APNs       │ OS-level persistent socket   │ Native mobile push             │
└──────────────────┴──────────────────────────────┴────────────────────────────────┘
```

> This completes the full Week 10 topic on Messaging — from internal service message queues, through webhooks, to client-side push in all its forms.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

