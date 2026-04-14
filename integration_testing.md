# Origin Round Integration Testing & Payload Examples

To ensure your integration is robust before going live, write automated tests against the **Sandbox Environment**.

Because the Origin Round Sandbox is perfectly stateless, you do not need to mock HTTP requests or spin up fake databases. You can hit the actual `/test` endpoint using deterministic Test Codes.

---

## Important Architectural Rules

1. **Test Mode Limitations:** You can **only** use the magic `code` provided in the Origin Round UI to test against the Sandbox (`/test`). You cannot test a `ref` here — the Sandbox uses the code's prefix to calculate the mock state.
2. **Live Mode Consistency:** When you switch to the Live API (`/verify`), whether you query using a Secret Code (`code`) or a Public Reference (`ref`), the JSON response payload is **100% identical**.

---

## 🛠️ Integration Test Suite (JavaScript)

Copy this into `origin-round-test.mjs`, swap in your Project UUID and Test API Key, and run with `node origin-round-test.mjs`.
```javascript
// 1. Configure your keys
const PROJECT_UUID = 'your_project_uuid_here';
const TEST_API_KEY = 'or_test_your_key_here';

// 2. Grab your Safe-Hex Test Codes from the Origin Round Dashboard
const CODES = {
    FRESH:   'TFRES-HABKN-PKCPF-LGKHG-ELMPL',
    USED:    'TUSED-DABKN-PKCPF-LGKHG-ELMPL',
    EXPIRED: 'TEXPR-DABKN-PKCPF-LGKHG-ELMPL',
    BLOCKED: 'TBLOC-KABKN-PKCPF-LGKHG-ELMPL'
};

// 3. The API call wrapper
async function verifyUserAccess(testCode) {
    const response = await fetch(`https://api.originround.com/api/partners/${PROJECT_UUID}/test`, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${TEST_API_KEY}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ code: testCode })
    });
    return await response.json();
}

// 4. Run the Assertions
async function runIntegrationTests() {
    console.log(" Starting Origin Round Integration Tests...\n");

    try {
        // Test 1: Grants access to a brand new, unused asset
        const fresh = await verifyUserAccess(CODES.FRESH);
        console.assert(fresh.data.is_valid === true,          "FRESH code should be valid");
        console.assert(fresh.data.already_in_use === false,   "FRESH code should not be in use");
        console.log(" Test 1 Passed: Brand new asset accepted.");

        // Test 2: Handles a returning user (Already Consumed)
        const used = await verifyUserAccess(CODES.USED);
        console.assert(used.data.is_valid === true,           "USED code should be valid");
        console.assert(used.data.already_in_use === true,     "USED code SHOULD be in use");
        console.log(" Test 2 Passed: Returning user recognized.");

        // Test 3: Denies access for an Expired Subscription
        const expired = await verifyUserAccess(CODES.EXPIRED);
        console.assert(expired.data.is_valid === false,                         "EXPIRED code should be invalid");
        console.assert(expired.data.asset.billing_status === 'CANCELED',        "EXPIRED code should be CANCELED");
        console.log(" Test 3 Passed: Expired subscription blocked.");

        // Test 4: Denies access for a Blocked/Refunded Asset
        const blocked = await verifyUserAccess(CODES.BLOCKED);
        console.assert(blocked.data.is_valid === false,                "BLOCKED code should be invalid");
        console.assert(blocked.data.asset.status === 'BLOCKED',        "BLOCKED code should be BLOCKED");
        console.log(" Test 4 Passed: Blocked asset denied.");

        console.log("\n All integration tests passed successfully!");
    } catch (err) {
        console.error(" Test Failed:", err);
    }
}

runIntegrationTests();
```

---

## Raw Payload Examples

Use these payloads to generate data structs and interfaces in statically typed languages (TypeScript, Go, Rust, etc.).

---

### Security & Validation Errors

**Missing Auth Header — `401 Unauthorized`**
```json
{ 
  "error": "Unauthorized: Missing or invalid Bearer token." 
}
```

**Wrong Environment Key — `401 Unauthorized`**
```json
{
  "status": "error",
  "message": "Unauthorized: This endpoint requires a LIVE key.",
  "request_id": "019ce726-e1f1-768a-832a-16beccd6c783"
}
```

**Both `code` and `ref` Provided — `400 Bad Request`**
```json
{
  "status": "error",
  "message": "You cannot provide both \"code\" and \"ref\" at the same time.",
  "request_id": "019ce726-e1f5-75d4-bc7b-893ef5ae0896"
}
```

**Attempt to Consume via Public Ref — `400 Bad Request`**
```json
{
  "status": "error",
  "message": "Bad Request: You cannot consume an asset using a public ref. You must provide the secret code.",
  "request_id": "019ce726-e1fa-706e-98d4-fd4ec992b776"
}
```

---

### Live Environment

**Verify via Public Ref — Invalid / Unconsumed**
> Payload sent: `{"ref": "OR-3529-624071", "action": "verify"}`
```json
{
  "status": "success",
  "livemode": true,
  "data": { 
    "is_valid": false, 
    "already_in_use": false 
  }
}
```

**Verify via Public Ref — Valid Subscription, Claimed**
> Payload sent: `{"ref": "OR-2E33-BCFF4A", "action": "verify"}`
```json
{
  "status": "success",
  "livemode": true,
  "request_id": "019ce726-e205-719f-a040-610887035aae",
  "data": {
    "is_valid": true,
    "already_in_use": true,
    "project": {
      "id": "019cd415-0a7e-717b-a837-944995243add",
      "title": "test proper"
    },
    "offer": {
      "id": "019cd92d-5a69-764a-bda5-892f3b3714e6",
      "title": "test more then year",
      "billing_mode": "subscription",
      "type": "access",
      "value": 1
    },
    "asset": {
      "public_ref": "OR-2E33-BCFF4A",
      "status": "CONSUMED",
      "billing_status": "ACTIVE",
      "expires_at": "2026-04-13T10:46:35.000Z",
      "partner_activated_at": "2026-03-13T11:48:31.000Z"
    },
    "custom_metadata": { "license_tier": "pro", "max_workspaces": 3 }
  }
}
```

**Consume via Secret Code**
> Payload sent: `{"code": "NASYD-BQME5-Y3UGJ-W64PV-T9AHW", "action": "consume"}`
>
> `partner_activated_at` is stamped at the exact time of consumption. Subsequent calls will return `already_in_use: true`.
```json
{
  "status": "success",
  "livemode": true,
  "request_id": "019ce726-e20b-747e-bd09-159429975d9a",
  "data": {
    "is_valid": true,
    "already_in_use": true,
    "project": {
      "id": "019cd415-0a7e-717b-a837-944995243add",
      "title": "test proper"
    },
    "offer": {
      "id": "019cd92d-5a69-764a-bda5-892f3b3714e6",
      "title": "test more then year",
      "billing_mode": "subscription",
      "type": "access",
      "value": 1
    },
    "asset": {
      "public_ref": "OR-2E33-BCFF4A",
      "status": "CONSUMED",
      "billing_status": "ACTIVE",
      "expires_at": "2026-04-13T10:46:35.000Z",
      "partner_activated_at": "2026-03-13T11:48:31.000Z"
    },
    "custom_metadata": { "license_tier": "pro", "max_workspaces": 3 }
  }
}
```

---

### Sandbox Environment

Deterministic responses from the `/test` endpoint using mock keys.

**Sandbox: FRESH Asset**
```json
{
  "status": "success",
  "livemode": false,
  "data": {
    "is_valid": true,
    "already_in_use": false,
    "project": {
      "id": "019cd415-0a7e-717b-a837-944995243add",
      "title": "test proper"
    },
    "offer": {
      "id": "019cd92d-5a69-764a-bda5-892f3b3714e6",
      "title": "test more then year",
      "billing_mode": "subscription",
      "type": "access",
      "value": 1
    },
    "asset": {
      "public_ref": "OR-AAAA-000001",
      "status": "CONSUMED",
      "billing_status": "ACTIVE",
      "expires_at": "2026-04-12T12:23:31.094Z",
      "partner_activated_at": null
    },
    "custom_metadata": { "license_tier": "pro", "max_workspaces": 3 }
  }
}
```

**Sandbox: EXPIRED Asset**
```json
{
  "status": "success",
  "livemode": false,
  "data": {
    "is_valid": false,
    "already_in_use": false,
    "project": {
      "id": "019cd415-0a7e-717b-a837-944995243add",
      "title": "test proper"
    },
    "offer": {
      "id": "019cd92d-5a69-764a-bda5-892f3b3714e6",
      "title": "test more then year",
      "billing_mode": "subscription",
      "type": "access",
      "value": 1
    },
    "asset": {
      "public_ref": "OR-CCCC-000003",
      "status": "EXPIRED",
      "billing_status": "CANCELED",
      "expires_at": "2026-03-12T12:23:31.109Z",
      "partner_activated_at": null
    },
    "custom_metadata": { "license_tier": "pro", "max_workspaces": 3 }
  }
}
```
