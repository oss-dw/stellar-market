# Progressive Web App Implementation

This document describes the Progressive Web App (PWA) implementation for StellarMarket, adding offline support, background sync, and Web Push notifications.

## Overview

The PWA implementation enables:

- **Installability**: Users can install the app on mobile and desktop
- **Offline browsing**: Job listings and cached content work without internet
- **Background sync**: Pending actions (applications, messages) sync automatically when online
- **Web Push notifications**: Real-time notifications for job matches, applications, milestones, and messages

## Features Implemented

### 1. Web App Manifest

**File**: `frontend/public/site.webmanifest`

The manifest enables "Add to Home Screen" on mobile and the install prompt on desktop Chrome/Edge. It includes:

- App name and description
- Theme and background colors
- Icons at 192x192 and 512x512 pixels
- `display: "standalone"` for app-like experience

**Icons required**:

- `icon-192.png` (192x192 pixels)
- `icon-512.png` (512x512 pixels)

Generate these from `favicon.svg` using an image converter.

### 2. Service Worker with Workbox

**Configuration**: `frontend/next.config.js`

Uses `next-pwa` to generate a service worker at build time with caching strategies:

| Resource              | Strategy                           | Rationale                    |
| --------------------- | ---------------------------------- | ---------------------------- |
| Next.js static chunks | Cache-first                        | Content-hashed, immutable    |
| API GET responses     | Stale-while-revalidate (5 min TTL) | Show cached data instantly   |
| Stellar SDK WASM      | Cache-first                        | Large binary, rarely changes |
| Font files            | Cache-first (1 year TTL)           | Stable assets                |
| API write operations  | Network-only                       | Must not cache mutations     |

The service worker is disabled in development mode for easier debugging.

### 3. Background Sync

**File**: `frontend/src/utils/backgroundSync.ts`

Uses IndexedDB to queue pending actions when offline:

- Applications
- Messages

When connectivity is restored, the service worker automatically replays stored requests.

**Important constraint**: Stellar transaction signing requires Freighter (browser extension), which is not available in a service worker context. Background sync does NOT queue transaction submissions—only idempotent API calls.

**API**:

```typescript
// Queue an action for background sync
await queueAction('application', '/api/applications', 'POST', { ... });

// Get pending actions
const pending = await getPendingActions();

// Manually replay all pending actions
await replayPendingActions();
```

### 4. Offline Status Hook

**File**: `frontend/src/hooks/useOfflineStatus.ts`

React hook that exposes:

```typescript
const { isOnline, hasPendingSync } = useOfflineStatus();
```

- `isOnline`: Boolean indicating network connectivity
- `hasPendingSync`: Boolean indicating if there are pending sync operations

### 5. Offline Banner

**File**: `frontend/src/components/OfflineBanner.tsx`

Shows a banner at the top of the page when:

- User is offline: "You are offline. Some features may be unavailable."
- Syncing pending changes: "Syncing pending changes..."

The banner appears below the navbar and is accessible (ARIA-compliant).

### 6. Web Push Notifications

#### Backend

**Database Model**: `PushSubscription` (Prisma)

```prisma
model PushSubscription {
  id         String   @id @default(cuid())
  userId     String
  endpoint   String   @unique
  p256dh     String
  auth       String
  createdAt  DateTime @default(now())
  user       User     @relation(fields: [userId], references: [id])
}
```

**API Endpoints**:

- `POST /api/notifications/push/subscribe`: Subscribe to push notifications
- `DELETE /api/notifications/push/unsubscribe`: Unsubscribe from push notifications

**Service**: `NotificationService` extended with:

- `subscribeToPush()`: Save push subscription to database
- `unsubscribeFromPush()`: Remove push subscription
- `sendPushNotification()`: Send Web Push using `web-push` library

Push notifications are sent for:

- New job match
- Application accepted/rejected
- Milestone submitted/approved
- Dispute opened/resolved
- New message

#### Frontend

**Component**: `frontend/src/components/PushNotificationPrompt.tsx`

Prompts users to enable notifications after their first interaction with the app. The prompt:

- Appears once per session after a 3-second delay
- Can be dismissed (won't show again)
- Requests permission and subscribes to push notifications

**Configuration**:

- Requires VAPID keys (see Setup below)

## Setup Instructions

### 1. Generate VAPID Keys

Web Push requires VAPID keys for authentication:

```bash
cd backend
npx web-push generate-vapid-keys
```

This outputs:

```
Public Key: <public-key>
Private Key: <private-key>
```

### 2. Configure Environment Variables

#### Backend (`backend/.env`)

```env
VAPID_PUBLIC_KEY="<public-key-from-above>"
VAPID_PRIVATE_KEY="<private-key-from-above>"
VAPID_SUBJECT="mailto:admin@stellarmarket.io"
```

#### Frontend (`frontend/.env.local`)

```env
NEXT_PUBLIC_VAPID_PUBLIC_KEY="<public-key-from-above>"
```

### 3. Generate PWA Icons

Convert `frontend/public/favicon.svg` to PNG icons:

```bash
# Using ImageMagick or similar tool
convert -background none favicon.svg -resize 192x192 icon-192.png
convert -background none favicon.svg -resize 512x512 icon-512.png
```

Or use an online converter like [Favicon Generator](https://realfavicongenerator.net/).

### 4. Run Database Migration

```bash
cd backend
npm run prisma:migrate
```

This creates the `PushSubscription` table.

### 5. Build and Test

#### Development

```bash
# Backend
cd backend
npm run dev

# Frontend
cd frontend
npm run dev
```

**Note**: Service worker is disabled in development mode.

#### Production

```bash
# Frontend
cd frontend
npm run build
npm start
```

The service worker activates in production builds.

## Testing

### Test Offline Mode

1. Open DevTools → Application → Service Workers
2. Check "Offline" checkbox
3. Navigate to `/jobs` — cached content should render
4. Submit a message — it should queue for background sync
5. Uncheck "Offline" — message should send automatically

### Test Push Notifications

1. Perform an action (e.g., apply to a job) to trigger the push prompt
2. Click "Enable notifications"
3. Grant permission in browser prompt
4. Have another user perform an action that triggers a notification
5. Check that push notification appears (even if tab is not active)

### Test Installability

#### Mobile (Android Chrome)

1. Visit the site
2. Tap the browser menu (⋮)
3. Select "Add to Home Screen"
4. App icon appears on home screen
5. Tap icon → app opens in standalone mode

#### Desktop (Chrome/Edge)

1. Visit the site
2. Look for install icon in address bar
3. Click to install
4. App opens in standalone window

## Lighthouse PWA Score

After implementation, run Lighthouse audit:

```bash
npm install -g lighthouse
lighthouse https://stellarmarket.io --view
```

Target score: **≥ 90/100** in the PWA category.

## Architecture Decisions

### Why IndexedDB over LocalStorage?

- IndexedDB supports larger data (no 5-10 MB limit)
- Asynchronous API (doesn't block UI thread)
- Supports complex queries and indexing
- Required for Background Sync API

### Why next-pwa over Custom Service Worker?

- Automatic Workbox configuration
- Integrates seamlessly with Next.js App Router
- Handles precaching of Next.js assets
- Production-ready out of the box

### Why Not Queue Transactions for Background Sync?

Stellar transaction signing requires Freighter (browser extension), which is:

1. Not accessible from service worker context
2. Requires user interaction (signature approval)
3. Not suitable for automatic background replay

For these reasons, transaction flows detect offline state and block with a clear message.

## Browser Support

| Feature          | Chrome | Firefox | Safari         | Edge |
| ---------------- | ------ | ------- | -------------- | ---- |
| Service Worker   | ✅     | ✅      | ✅             | ✅   |
| Web App Manifest | ✅     | ✅      | ✅             | ✅   |
| Background Sync  | ✅     | ❌      | ❌             | ✅   |
| Web Push         | ✅     | ✅      | ✅ (iOS 16.4+) | ✅   |

**Fallback**: On browsers without Background Sync, pending actions are replayed when the user returns online and the app is open.

## Maintenance

### Update Caching Strategy

Edit `frontend/next.config.js` → `runtimeCaching` array.

### Add New Notification Type

1. Update `backend/src/services/notification.service.ts` → `pushEnabledTypes` array
2. Send test notification to verify delivery

### Rotate VAPID Keys

1. Generate new keys: `npx web-push generate-vapid-keys`
2. Update environment variables in backend and frontend
3. Redeploy both backend and frontend
4. Existing subscriptions will fail and users will be prompted to re-subscribe

## Troubleshooting

### Service Worker Not Registering

- Check browser console for errors
- Ensure you're running a production build (`npm run build && npm start`)
- Verify HTTPS is enabled (required for service workers in production)

### Push Notifications Not Working

- Check VAPID keys are configured correctly
- Verify `NEXT_PUBLIC_VAPID_PUBLIC_KEY` matches backend `VAPID_PUBLIC_KEY`
- Check browser permissions (Settings → Notifications)
- Inspect `/api/notifications/push/subscribe` response in Network tab

### Offline Content Not Loading

- Check service worker is active (DevTools → Application → Service Workers)
- Verify caching strategy in `next.config.js`
- Clear cache and reload: DevTools → Application → Clear storage

## Security Considerations

- VAPID keys are stored in environment variables (not in code)
- Push subscription endpoints are unique per user/device
- Invalid subscriptions (410/404 errors) are automatically removed
- Service worker scope is limited to `/` (entire app)

## Performance Impact

- Service worker adds ~50 KB to initial load (one-time cost)
- Cached resources load instantly on subsequent visits
- Background sync operations use minimal battery
- Push notifications are sent server-side (no client overhead)

## Future Enhancements

- [ ] Add "Update Available" prompt when new service worker is detected
- [ ] Implement smarter caching strategies (e.g., cache only visited job pages)
- [ ] Add offline job search using cached index
- [ ] Enable push notification customization (per notification type)
- [ ] Add push notification action buttons (e.g., "View Job", "Reply")

## Resources

- [Next PWA Documentation](https://github.com/shadowwalker/next-pwa)
- [Workbox Documentation](https://developer.chrome.com/docs/workbox/)
- [Web Push Protocol](https://datatracker.ietf.org/doc/html/rfc8030)
- [Background Sync API](https://developer.chrome.com/blog/background-sync/)
- [Service Worker Cookbook](https://serviceworke.rs/)
