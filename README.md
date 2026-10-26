# Bill Splitter (Android) — setup, integration status and test results

What is connected, what was tested, and the **few console steps only the Firebase project owner can
do**. Backend hand-off: `ANDROID_APP_SETUP.md` (from the backend team, 2026-10-16).

---

## ⚠️ To do: 3 clicks in the Firebase console (friends & invites need them)

Checked live on 2026-10-16. Everything else is connected and tested; friend invites/requests are
the only feature still blocked, and only by these project settings (the backend team can't do them,
the app can't either):

| # | Firebase console ▸ project **`er-bill-splitter`** | Status now |
|---|---|---|
| 1 | **Authentication ▸ Sign-in method ▸ Anonymous ▸ Enable** | ❌ off (`ADMIN_ONLY_OPERATION`) |
| 2 | **Firestore Database ▸ Create database** (production mode, any region) | ❌ does not exist |
| 3 | **Firestore Database ▸ Rules** ▸ paste the whole `firestore.rules` file from this folder ▸ **Publish** | ❌ (needs #2) |

Re-checked after the new `google-services.json` (app `com.er.bill.splitter.calculator.expense`,
4:41 PM): adding the Android app to Firebase does **not** create the database — #1 and #2 are still
❌. That is the whole reason for *"Firestore didn't answer"*: the app writes to a database that doesn't
exist, Firestore keeps retrying silently, and the app gives up after 15 s. The app now checks this
itself and says *"The Firestore database … hasn't been created yet"* instead.

The new package is also in `hosting/public` (invite page, `assetlinks.json`) and `docs/assetlinks.json`.
**Backend team:** the page at `janbarkcloud.com/bill-splitter/invite/<id>` and its `assetlinks.json`
must use `com.er.bill.splitter.calculator.expense` too (Play Store link + App Links).

No CLI and no Firebase Hosting deploy are needed any more: invite links now use the backend's page
(see §3). Until these are done, **Send invite** shows *"Couldn't create the invite link"* with the
exact reason and a **Share without link** option. Afterwards it works without an app update.

Open: https://console.firebase.google.com/project/er-bill-splitter

---

## 1. Integration with the backend (done)

`local.properties` (not in git) — now the HTTPS production server:

```properties
BASE_URL=https://janbarkcloud.com/bill-splitter/
API_KEY=<key from the project owner>
```

The build also defaults to this URL. Release builds need HTTPS, which this is.

| Backend endpoint | Used by the app for | Tested 2026-10-16 |
|---|---|---|
| `POST auth/otp/request`, `auth/otp/verify`, `auth/register` | Email-code sign in / sign up | ✅ request + wrong-code error in the app (a real code needs a real inbox — see §5) |
| `POST auth/guest {device_id}` | **Continue as guest** (same phone → same guest) | ✅ in the app, incl. sign out → back in keeps the data |
| `POST auth/refresh`, `auth/logout` | Session renewal on 401, Sign out | ✅ sign out |
| `POST auth/firebase-token` | Email users → Firebase custom token (`bs_<id>`) | ✅ 403 for guests (as specified); guests use Anonymous Firebase |
| `POST users/me/upgrade` + `users/me/upgrade/verify {email, code}` | **NEW:** Edit profile ▸ *Add your email to keep this account* (guest → email account, data kept) | ✅ code sent; wrong code → "Invalid code" |
| `POST users/me/email` + `users/me/email/verify {email, code}` | **NEW:** Edit profile ▸ *Change email* (verified; no more `PATCH` email) | ✅ same dialogs as upgrade |
| `POST users/me/device-tokens {token, platform}` / `DELETE users/me/device-tokens {token}` | **NEW:** push notifications (FCM) — registered after sign in, removed on sign out | ✅ "Push token registered with the backend" (logcat tag `Push`) |
| `PATCH users/me/notification-settings` | Notifications switches; now also sent after every sign in so the server matches the phone | ✅ |
| `PATCH users/me`, `DELETE users/me` | Edit name/phone, Delete account | ✅ |
| friends, groups, members, expenses, settlements | Everything else | ✅ end-to-end (§4) |

### Invite links (now on the backend's domain)

Shared link: `https://janbarkcloud.com/bill-splitter/invite/<id>` — the backend's page is live and
opens the app via `billsplatter://invite/<id>` (tested: the app opens and keeps the invite until
sign in). Links on `er-bill-splitter.web.app/invite/<id>` (the optional `hosting/` site) are accepted too.
Change the address in one place: `inviteHost` / `invitePath` in `app/build.gradle.kts`.

**Ask the backend team** (one-time) to put the app's signing fingerprints into
`https://janbarkcloud.com/.well-known/assetlinks.json` (it has a placeholder now), so links open the
app directly instead of the browser page first:

- debug (this PC): `16:9C:32:DD:55:CC:FB:4C:3C:09:15:21:CE:A5:3D:07:C4:E3:5E:3F:EE:AB:38:D6:02:8C:66:D5:F7:2D:17:64`
- release: Play Console ▸ *Test and release ▸ App integrity ▸ App signing* ▸ SHA-256

### How the friend flow works (after the Firebase steps)

```
Alex: Friends ▸ friend ▸ Send invite ──▶ Firestore invites/{id} ──▶ share sheet with https://janbarkcloud.com/bill-splitter/invite/{id}
Ben taps the link ──▶ app opens ──▶ sign in if needed ──▶ "Alex wants to be friends" (Accept / Decline / Not now)
Ben: Accept ──▶ friendRequests/{alex}_{ben} = accepted ──▶ Ben's phone: Alex is Linked
Alex's phone (listening) ──▶ links Ben to the friend row the invite was for ──▶ both Linked; new shared expenses copy over
```

Email users can also be found by email (Friends ▸ Add ▸ Add by email ▸ Send friend request → the other
person sees it in *Incoming requests*). Guests use invite links.

## 2. Build

```bash
gradlew.bat assembleDebug
```

JDK: `F:\AndroidDev\Jdks\openjdk-26.0.1` (`JAVA_HOME`). Output: `app/build/outputs/apk/debug/app-debug.apk`.
Build, lint (0 errors) and unit tests pass.

## 3. Optional: Firebase Hosting / CLI

Not required any more. `firebase.json`, `.firebaserc` and `hosting/public/` remain if you prefer
links on `er-bill-splitter.web.app`; with the CLI, `firebase deploy --only firestore:rules,hosting`
publishes the rules and that page in one go.

## 4. What was tested on the emulator (Pixel 5, API 37, live HTTPS backend)

Clean install → onboarding → **Continue as guest** → notification permission prompt → push token
registered → Home (getting started) → Add friends (Maya, John) → Send invite (shows the Firebase
reason dialog; *Share without link* opens the share sheet) → Create group "Flat" picking existing
friends → Add expense $90 split equally ($30 each) → Balances (Maya/John owe $30) → Settle up with
Maya → group +$30, Friends list (Maya Settled, John +$30), John's screen (+$30 in Flat), Home
(+$30.00), Activity ("Just now") → Edit profile ▸ Add email (code sent, wrong code rejected) →
Sign out → guest again (same data back) → Delete account → guest again (fresh). Deep link
`billsplatter://invite/…` opens the app and keeps the invite. No crashes in the log.

Earlier fixes (still in): double-tap crashes, splash hang, duplicate friends from Add members,
per-friend balances, "$0 · Settled" header, double settle, future dates, guest Edit Profile,
Home first-run state, currency symbols, "Just now", invite screen scrolling, malformed replies.

## 5. Needs you (can't be done from here)

1. **Firebase console steps** at the top.
2. **One real email test:** public test inboxes don't receive the backend's mail, so a full
   email-code sign in and the guest → email upgrade need a real inbox. Sign in with your email, or
   use Edit profile ▸ *Add your email*.
3. **Two-phone friend test** after the Firebase steps: invite → Accept → both *Linked* → add an
   expense with the friend → it appears on the other phone.
4. Backend team: real fingerprints in `assetlinks.json` (above). Push *sending* (FCM) isn't wired on
   the backend yet; the app already registers tokens and shows pushes when they arrive.

## 6. Known limits

- **Anonymous (guest) Firebase ids are per install**: after reinstalling or signing out, earlier
  friend links don't carry over. Email users keep `bs_<id>` on every phone. Upgrading a guest to email
  switches it to `bs_<id>` (links made as a guest need re-inviting).
- Only email accounts can be found by email.
- Firestore only carries friend links and new expenses between linked friends; groups and balances
  stay on the REST backend.
- Still "Coming soon" by design: change photo, Pro purchase (no backend for them).
- Firestore rules were tested on the local emulator (24 cases).
