# SwiftDelivery

Logistics web app for the Nigerian market. Coordinates dispatchers, riders, and customers with real-time order tracking.

> **Stack reality check:** despite earlier briefs describing this as a React app, the codebase is a single-file vanilla JS app — everything ships from `index.html`. There is no build step, no `package.json`, and no `node_modules/`. Static-deploy to Vercel.

## What's in V3

- **Proof of Delivery** — the existing photo + signature + note modal now actually persists. Photos and signature PNGs upload to Firebase Storage at `pod/<orderId>/*`; the order doc gets `pod = { photoUrl, signatureUrl, note, completedAt, completedBy }`. Customers can view the proof from the History tab.
- **Customer ratings & reviews** — on delivered orders the customer sees a ⭐ Rate button. 1–5 stars + optional comment writes to a new `swd/reviews` Firestore doc and recomputes the rider's running average.
- **In-app payments (Paystack test mode)** — Pay button on unpaid active orders launches the Paystack inline checkout. Successful payments stamp `order.paymentStatus='paid'` and write a debit ledger entry.
- **Browser push notifications** — when an open tab is hidden/unfocused, status changes for the customer's own orders fire a `Notification`, and new rider notif entries do the same for the logged-in rider. No backend, no service worker — closed-tab push is intentionally out of scope (would need FCM + Cloud Functions).

## Setup

### 1. Clone

```bash
git clone https://github.com/retiredpotato/Swiftdelivery.git
cd Swiftdelivery
```

There's nothing to install — this is a single HTML file. To run locally, serve the directory with any static server:

```bash
python3 -m http.server 8000
# or
npx serve .
```

Then open http://localhost:8000.

### 2. Configure keys

Open `index.html` and replace the placeholder values in the top `<script>` block(s):

| `window.SWD_*` global | What to paste | Where to get it |
|---|---|---|
| `SWD_MAPS_KEY` | Google Maps API key (Maps JS + Places enabled) | console.cloud.google.com → APIs |
| `firebaseConfig` | Firebase web config object | console.firebase.google.com → Project settings → SDK setup |
| `SWD_PAYSTACK_KEY` (V3) | Paystack public **test** key (`pk_test_…`) | dashboard.paystack.com → Settings → API Keys |

`.env.example` lists the same values as a checklist; this app does not load env at runtime, so the file is documentation only.

### 3. Lock down keys (do this before going live)

The Google Maps and Firebase keys live in page source — they're effectively public. Real protection comes from server-side restrictions:

- **Google Maps:** Cloud Console → APIs & Services → Credentials → your key → set **HTTP referrer restrictions** to your Vercel domain (e.g. `https://swiftdelivery.vercel.app/*`).
- **Firebase Firestore:** paste `firestore.rules.example` into the Firebase Console → Firestore → Rules tab and tighten as you add Firebase Auth.
- **Firebase Storage:** paste `storage.rules.example` into Storage → Rules. The default caps POD uploads at 10 MB and image-only.
- **Paystack:** test keys are unrestricted by design. When swapping to a live key, enable **webhook signature verification** on your backend before trusting `paymentStatus = 'paid'` in production.

## V3 — How to test

Run the static server, open in two browsers (or two profiles) so you can play multiple roles.

1. **Sign in** as a customer (`amaka@demo.ng` / `demo123`) and as a rider (`chukwu@demo.ng` / `demo123`).
2. **Payment**: on the customer dashboard, find an active order, click **💳 Pay**. The Paystack popup opens. Use a test card from [paystack.com/docs/payments/test-payments](https://paystack.com/docs/payments/test-payments) (e.g. `4084 0840 8408 4081`, any CVV, any future expiry, OTP `123456`). On success the row should switch to **✓ Paid**.
3. **Proof of Delivery**: as the rider, advance an order to *Out for Delivery* and submit POD with a photo + signature + note. Confirm it shows **Uploading…** then **Delivery confirmed! Proof saved.** As the customer, switch to History and click **📦 POD** on that order — photo, signature, and note should all render.
4. **Ratings**: as the customer, click **⭐ Rate** on the just-delivered order. Pick stars, submit. The row gets a small **★N** badge and the rider's `rating` should reflect the running average.
5. **Push**: leave the customer tab hidden (switch to another window). As the rider/dispatcher, change the order status. A browser notification should fire on the customer's tab. (First run will prompt for permission — grant it.)
6. **Mobile (375px)**: open dev tools, set viewport to iPhone SE (375×667). All four V3 flows should be reachable without horizontal scroll.

## Regression checklist (V2 must still work)

- Login / signup / ban screen
- Order creation flow
- Status pipeline: Placed → In Transit → Out for Delivery → Delivered
- Live GPS fleet map (admin)
- Live tracking map (customer)
- Rider notifications + online toggle
- Wallet view + Fund Wallet modal

If any of those break, V3 hasn't shipped — all four V3 flows route through existing rendering and the existing `_flushToDB` debounce, so a regression here means a wiring mistake.

## Firestore layout (V3)

V3 keeps the V2 single-doc-per-state pattern (one `onSnapshot` listener per doc — no listener storms):

```
swd/
  users         → { list: [...] }
  orders        → { list: [...] }     // V3: orders gain optional .pod, .rating, .paymentStatus, .paymentRef, .paidAt
  riders        → { list: [...] }     // V3: rider.rating becomes a running avg of reviews
  misc          → { notifications, riderNotifs }
  transactions  → { data: { userId: [...txns] } }
  reviews       → { list: [...] }     // V3 — new
locations/_doc  → { rider_<id>: {lat, lng, ts, ...} }
```

## Deploy

The repo deploys to Vercel as a static site — push to `main` and Vercel picks up `index.html`. No build command, no framework preset.

After merging V3, **add the Paystack public test key to `index.html` on `main`** before redeploying — the placeholder will throw a toast on the Pay button otherwise.
