# One Day Leader

A single-file Firebase application for managing a 10-student daily "One Day Leader" (日直) rotation, weekday schedules, personal availability plans, and automatic absence fallback.

## App

- `index.html` — self-contained UI using Tailwind CDN and Firebase Modular SDK v9+.
- Firebase Authentication (email/password) with a 10-student whitelist.
- Cloud Firestore real-time sync via `onSnapshot`.
- No lesson reports, logs, comments, or report history.

## Features

- Firebase Auth uses explicit `browserSessionPersistence`: reloads in the same tab stay signed in, while new visits start at Sign In.
- Anti-cache meta tags and an `APP_VERSION` auto-reload check keep deployed clients current.

- 2-month (60-day) availability calendar with Japanese national holidays fetched from `holidays-jp.github.io`.
- Personal statuses: 未登録 (default), 出席 (Present), 欠席 (Absent), 公欠 (Official Absence), 遅刻 (Late Arrival), 早退 (Early Leave).
- Late/early statuses require a class-period selection.
- Live class tracker banner based on the current device clock and the weekly timetable.
- Weekdays only:
  - Saturdays/Sundays and Japanese national holidays are displayed as **休日 / 祝日**.
  - No leader is assigned on weekends or holidays.
  - Rotation freezes over the weekend and resumes Monday.
- Round-robin ring rotation:
  - The base index advances by one for every valid school weekday.
  - Saturdays, Sundays, and Japanese holidays are skipped.
  - Same-day fallback searches forward through the ring without changing tomorrow's base index.
  - Cascades skip absent/official-absence students and the previous weekday's active leader.
- Initial class state (when Firestore is first initialized on a weekday):
  - Base: 鈴木 斗和 — 欠席
  - Next: 小林 大輝 — 公欠
  - Active: 吉田 瑛都
- Reactive class alert banner and Live Alerts panel derived only from the current Firestore status state. Historical alert logs are not stored or displayed.
- Scheduled browser notification at 10 minutes before the first class:
  - Mon / Wed / Thu: 09:00
  - Tue: 11:00
  - Fri: 10:00
  - Sat / Sun: off

## Firebase setup

The Firebase configuration is embedded in `index.html`.

1. Enable **Authentication → Sign-in method → Email/Password**.
2. Create a **Cloud Firestore** database.
3. Deploy suitable Firestore rules. Example development rules:

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() {
      return request.auth != null;
    }
    function whitelisted() {
      return signedIn() && request.auth.token.email.matches('.*@technos\\.cloud$');
    }

    match /config/rotation {
      allow read: if whitelisted();
      allow create: if whitelisted();
      allow update: if false;
    }
    match /statuses/{date} {
      allow read: if whitelisted();
      allow create, update: if whitelisted();
    }
    match /alerts/{alertId} {
      allow read: if whitelisted();
      allow create: if whitelisted();
      allow update, delete: if false;
    }
  }
}
```

For production, tighten the rules further to verify the exact 10 whitelisted emails and restrict status writes to the authenticated user's own key.

## Firestore data model

- `config/rotation` — rotation anchor date and initial index.
- `statuses/{YYYY-MM-DD}` — map of student code → `present | absent | official`.
- `alerts/{autoId}` — class-wide status/fallback notifications.

## Notifications

- Notification permission is requested on sign-in.
- The scheduled notification uses a client-side timer while the page is open. True background Web Push across devices requires Firebase Cloud Messaging plus a backend scheduler/service worker.
- Browser notifications require HTTPS or localhost.

## Run

Serve the folder with a local HTTP server and open it in the browser:

```bash
python -m http.server 8000
```

Then visit `http://localhost:8000/`.
