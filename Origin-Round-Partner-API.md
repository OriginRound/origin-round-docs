# Origin Round Partner API

Welcome to the Origin Round Partner API. This Server-to-Server (B2B) integration allows your backend to securely verify, consume, and monitor digital assets (licenses, subscriptions, and items) purchased on Origin Round.

** Zero-Trust Rule:** Never expose your API keys in a frontend client. All requests must originate from your secure server.

---

## ⚡ 1. The "Fast Start" Integration Logic

Origin Round does the heavy lifting for you. You do not need to calculate expiration dates, grace periods, or ban states. We provide the absolute source of truth via two master flags: `is_valid` and `already_in_use`.

Here is the exact logic you should implement on your backend:

```javascript
// 1. Send the user's Secret Code to the Origin Round Live /verify endpoint
const originResponse = await verifyAsset(userInputCode);

// 2. Evaluate the Master Switch
if (!originResponse.data.is_valid) {
    //  STOP: The asset is expired, refunded, banned, or fake.
    return "Access Denied. Code is invalid or expired."; 
}

// 3. Handle First-Time vs. Returning Users
// Note: Origin Round will return a 200 OK here, NOT an error. It is up to you to reject the double-spend based on this flag.
if (originResponse.data.already_in_use) {
    //  SECURITY: This code is valid, but was consumed in the past.
    // If your app binds codes to specific accounts, you must check your database here:
    // - If this is the original owner logging in -> Grant access.
    // - If this is a stranger (e.g., the code was leaked online) -> Deny access.
    return "This code has already been claimed."; 
} else {
    //  This is a brand new, untouched code! 
    // It is now permanently consumed. Bind it to this user's account in your database.
    return "Premium Unlocked! Access Granted.";
}
```

---

##  2. Identifiers & Use Cases

Origin Round uses two different strings to identify an asset. It is critical to understand when to use which.

### The Secret Code (`code`)
**Format:** 29 Characters `XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`
**Intent:** User UI Input & Claiming.
* This is the sensitive "license key" the user copies from Origin Round and pastes into your application.
* **Validation:** Contains uppercase letters and numbers (excludes `0, 1, I, O` to prevent typos). You can validate the shape on your frontend before sending it to your backend:
  `const isValid = /^[A-Z2-9]{5}(-[A-Z2-9]{5}){4}$/.test(input);`

> **💡 UX Tip:** Run this regex validation on your frontend *before* calling our API. It prevents unnecessary network requests and allows you to instantly warn the user if they accidentally pasted a code containing a `0` or an `O`.

### The Public Reference (`ref`)
**Format:** 14 Characters `OR-XXXX-XXXXXX` (e.g., `OR-A1B2-C3D4E5`)
**Intent:** Internal DB Storage & Background Syncs.
* This is a safe, read-only ID. You should store this in your database alongside the user's profile.
* **Why use it?** If you need to run a nightly CRON job to check if a user's subscription is still active, you query our API using this `ref` instead of their Secret Code.
* **Validation:** `const isValid = /^OR-[A-F0-9]{4}-[A-F0-9]{6}$/.test(input);`

---

## 3. How Subscriptions Work (`expires_at`)

If the user bought a subscription, the payload will include an `expires_at` timestamp and a `billing_status`.

You do **not** need to calculate time. Simply cache the `expires_at` date in your database and grant the user access until that exact moment.

* **When they auto-renew:** Origin Round handles the payment. The next time you verify that user's `code` or `ref`, the `expires_at` date will automatically be pushed forward.
* **If they cancel:** `billing_status` will change to `"CANCELED"`, but `is_valid` will remain `true` until their current paid period naturally hits the `expires_at` date.

### Handling One-Time Items (Quantity/Access)
If the user purchased a one-time item (like a pack of coins or a capacity boost), check `data.offer.value` in the success payload. This tells your backend exactly how many items to grant. You can safely map this integer directly to your database update queries.

---

## 4. Authentication & Endpoints

Get your API keys from the **Developer API** tab in your Origin Round Project Dashboard.

* **Header:** `Authorization: Bearer <YOUR_API_KEY>`
* **Content-Type:** `application/json`

### Option A: The Sandbox (Testing)

**Endpoint:** `POST https://api.originround.com/api/partners/:project_uuid/test`

>  **Prerequisite:** You must create at least one Listing (Offer) in your Project dashboard before the sandbox can generate test codes.

Use your `or_test_...` key. This endpoint is entirely **stateless** and does not affect real database inventory. Use the safe-hex test codes from your dashboard:

| Test Code Prefix | Simulates | 
| ----- | ----- | 
| `TFRES-...` | A brand new, valid code | 
| `TUSED-...` | An already consumed, valid code | 
| `TEXPR-...` | A past-due subscription | 
| `TBLOC-...` | An asset disabled due to chargeback | 

### Option B: Live Production

**Endpoint:** `POST https://api.originround.com/api/partners/:project_uuid/verify`

Use your `or_live_...` key. This connects to real user inventory. There is only **one URL**, but you change the JSON payload based on your intent:

**1. To Claim an Asset (Destructive / User UI):**
Permanently locks the asset and stamps your activation time.
```json
{
  "code": "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX",
  "action": "consume"
}
```

**2. To Check Access internally (Read-Only / CRON jobs):**
Safely checks if an asset is valid or expired without altering its state.
```json
{
  "ref": "OR-A1B2-C3D4E5"
}
```

>  **Important:** You can verify using a Public Ref, but you can **only** consume using a Secret Code.

---

##  5. The Success Payload

A successful lookup always returns **HTTP 200**. We include the `request_id` for debugging.

```json
{
  "status": "success",
  "livemode": true,
  "request_id": "019ce6cf-ea2f-7667-ba0a-c11684a6c670",
  "data": {
    "is_valid": true,
    "already_in_use": true,
    "project": {
      "id": "019cd415-0a7e-717b-a837-944995243add",
      "title": "My Awesome Game"
    },
    "offer": {
      "id": "019cd92d-5a69-764a-bda5-892f3b3714e6",
      "title": "Pro Tier Subscription",
      "billing_mode": "subscription",
      "type": "access",
      "value": 1
    },
    "asset": {
      "public_ref": "OR-2E33-BCFF4A",
      "status": "CONSUMED",
      "billing_status": "ACTIVE",
      "expires_at": "2026-04-13T10:46:35.000Z",
      "partner_activated_at": "2026-03-13T10:48:31.549Z"
    },
    "custom_metadata": {
      "server_realm": "EU-West",
      "discord_role_id": "123456789"
    }
  }
}
```

| Field | Description | 
| ----- | ----- | 
| `is_valid` | **The Master Switch.** If `false`, deny access immediately. | 
| `already_in_use` | If `true`, someone already ran `action: "consume"` on this code. | 
| `billing_mode` | `"payment"` (one-time) or `"subscription"` | 
| `type` | `"access"`, `"quantity"`, or `"discount"` | 
| `billing_status` | `"ACTIVE"`, `"CANCELED"`, or `"PAST_DUE"` | 
| `expires_at` | `null` for one-time payments; timestamp for subscriptions | 
| `partner_activated_at` | The exact UTC time this asset was first consumed | 
| `custom_metadata` | Custom JSON attached to the listing by the creator | 

---

## ❌ 6. Concurrency, Errors & Limits

### Race Conditions & Idempotency

Our sys uses **atomic database locks**. If your server accidentally double-fires a `consume` request simultaneously (or a user clicks a button twice), only one will succeed. The second request will safely return `200 OK` with `already_in_use: true`.

> It is always safe to retry timed-out requests.

### Error Payload Shape

If a request fails validation, authentication, or hits a rate limit, the API will return standard HTTP error codes with a consistent JSON envelope.

> Always include the `request_id` if you contact Origin Round Support.

```json
{
  "status": "error",
  "message": "Rate Limit Exceeded: Too many verification requests. Please slow down.",
  "request_id": "019ce6cf-ea14-716a-9559-8814df9e1f3b"
}
```

### HTTP Status Codes

| Code | Reason | 
| ----- | ----- | 
| **400** | Bad Request (e.g., trying to consume a Public Ref, missing both `code` and `ref`). | 
| **401** | Unauthorized (Missing token, or using a TEST key on a LIVE endpoint). | 
| **403** | Forbidden (Your API key was revoked in the dashboard). | 
| **429** | Too Many Requests (Limit: **600 requests per minute** per API key. Check standard `RateLimit-*` headers to see when to retry). |
