# n8n SMS Study Program — Setup Guide (Bird.com / MessageBird)

> **Two workflows have already been created on your n8n instance at `https://dev.rrwo.us`.**  
> This guide walks you through configuring them to go live using **Bird.com** (formerly MessageBird) as the SMS provider.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  WORKFLOW 1: Question Sender                 │
│                  ID: k4eXoRCzRowBDPpu                       │
│                                                              │
│  Schedule Trigger ──► Calculate Day/Slot ──► Should Send?   │
│  (8am, 12pm, 6pm)                              │            │
│                                                 ▼            │
│                                          Load Question ──►  │
│                                          Question Found? ──►│
│                                          Send SMS (Bird) ──►│
│                                          Store Answer        │
└──────────────────────────────────────────────────────────────┘
                          │
                          │ Student receives question via SMS
                          │ Student replies with ANY text
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                  WORKFLOW 2: Reply Handler                    │
│                  ID: XbBybhVWkCutoj68                        │
│                                                              │
│  Webhook (POST) ──► Load Pending Answer ──► Send Answer SMS │
│  (Bird inbound)     (from static data)      (via Bird)      │
└──────────────────────────────────────────────────────────────┘
```

### How It Works
1. **3x per day** (8am, 12:30pm, 6pm ET), Workflow 1 fires
2. It calculates which **day** of the 30-day program it is and which **time slot** (morning/midday/evening)
3. It looks up the matching question from `sms_schedule.json`
4. It **sends the question** via Bird.com (MessageBird) SMS
5. It **stores the correct answer** in n8n's persistent static data
6. When the student **replies with ANY text** (their guess), Bird.com forwards the SMS to n8n's webhook, triggering Workflow 2
7. Workflow 2 **retrieves the stored answer** and **sends it back** via Bird.com SMS

---

## Prerequisites

### 1. Bird.com Account (Required)
You need a Bird.com account with an SMS-capable number or sender ID.

1. **Sign up** at [bird.com](https://www.bird.com/) (formerly MessageBird — free trial credit available)
2. **Get a phone number** — Go to **Numbers** in the Bird dashboard and purchase a virtual number with SMS capability
3. **Get your API key** — Go to **Developers → API access** and create a REST API key
4. **Note these values:**
   - **API Key** (starts with `live_` for production or `test_` for sandbox)
   - **Bird Phone Number / Originator** (the number or alphanumeric sender ID, e.g., `+1234567890`)

### 2. Configure Bird.com Webhook for Incoming SMS
This is how Bird tells n8n when someone replies.

1. Go to **Bird Dashboard → Flow Builder** or **Developers → Webhooks**
2. Create a new webhook / flow for **Inbound SMS messages**
3. Set the **Webhook URL** to:
   ```
   https://dev.rrwo.us/webhook/cam-sms-reply
   ```
4. Set **HTTP Method** to `POST`
5. **Save and activate**

> ⚠️ **Note:** The webhook path `cam-sms-reply` is already configured in Workflow 2's Webhook node. When you activate the workflow in n8n, the URL becomes live. The production URL is:  
> `https://dev.rrwo.us/webhook/cam-sms-reply`

### 3. n8n Credentials Setup

In your n8n instance (`https://dev.rrwo.us`):

1. Go to **Settings → Credentials → Add Credential**
2. Search for **"MessageBird"**
3. Enter:
   - **API Key:** Your Bird.com REST API key
4. **Save**

### 4. Environment Variables

Set these in your n8n environment (or hardcode them directly in the nodes):

| Variable | Value | Example |
|----------|-------|---------|
| `BIRD_PHONE_NUMBER` | Your Bird.com number / originator | `+1234567890` |
| `STUDENT_PHONE_NUMBER` | The student's phone number | `+1987654321` |

**To set environment variables in n8n:**
- If self-hosted: Add to your `.env` file or Docker environment
- Alternative: Replace `$env.BIRD_PHONE_NUMBER` and `$env.STUDENT_PHONE_NUMBER` in the MessageBird nodes with hardcoded values

---

## Step-by-Step Configuration

### Step 1: Load the Question Data

The **"Load Question Data"** node in Workflow 1 needs the full question schedule. You have two options:

#### Option A: Inline JSON (Simplest)
1. Open Workflow 1 (`CAM Study - SMS Question Sender`) in n8n
2. Click the **"Load Question Data"** node
3. Replace `PASTE_SMS_SCHEDULE_JSON_HERE` with the contents of `sms_schedule.json` from this repo
4. The code should look like:
```javascript
const day = $input.first().json.day;
const slot = $input.first().json.slot;

const schedule = [
  { "day": 1, "slot": "morning", ... },
  { "day": 1, "slot": "midday", ... },
  // ... all 90 entries from sms_schedule.json
];

const match = schedule.find(q => q.day === day && q.slot === slot);

if (!match) {
  return [{ json: { found: false, reason: `No question for day ${day}, slot ${slot}` } }];
}

return [{ json: { ...match, found: true } }];
```

#### Option B: Fetch from GitHub (Auto-updates)
Replace the Load Question Data code with:
```javascript
const day = $input.first().json.day;
const slot = $input.first().json.slot;

// Fetch from your GitHub repo
const response = await fetch(
  'https://raw.githubusercontent.com/robbiepryan/CAM_LICENSE_STUDY/master/sms_schedule.json'
);
const schedule = await response.json();

const match = schedule.find(q => q.day === day && q.slot === slot);

if (!match) {
  return [{ json: { found: false, reason: `No question for day ${day}, slot ${slot}` } }];
}

return [{ json: { ...match, found: true } }];
```

### Step 2: Set Your Start Date

1. Open the **"Calculate Day & Slot"** node
2. Change `const START_DATE = '2026-02-10';` to your actual start date
3. The program runs for 30 days from this date

### Step 3: Configure MessageBird (Bird.com) Nodes

In **both workflows**, click each **MessageBird** node and:
1. Select your **MessageBird credential** (created in Prerequisites step 3)
2. Verify the **From** (originator) and **To** (recipients) phone numbers are correct
3. If not using environment variables, hardcode the numbers directly

### Step 4: Verify the Webhook Path

Workflow 2 uses a **Webhook** node (not a provider-specific trigger) to receive incoming SMS from Bird.com.

1. Open Workflow 2 (`CAM Study - SMS Reply Handler`)
2. Click the **"Bird Incoming SMS"** webhook node
3. Confirm the path is `cam-sms-reply`
4. The full production URL will be: `https://dev.rrwo.us/webhook/cam-sms-reply`
5. This is the URL you entered in Bird.com's webhook configuration (Prerequisites step 2)

### Step 5: Adjust Incoming SMS Field Mapping (If Needed)

The **"Load Pending Answer"** code node in Workflow 2 parses the incoming SMS payload from Bird.com. Bird's webhook payload format may vary depending on your configuration. The node currently handles multiple common field names:

```javascript
// These lines handle various Bird.com payload formats:
const incomingMessage = body.body || body.message?.text || body.text || body.Body || 'no message';
const fromNumber = body.originator || body.message?.from || body.from || body.From || body.sender || '';
```

**To verify the exact field names:**
1. Activate Workflow 2
2. Send a test SMS to your Bird number
3. Check the webhook node's output in n8n to see the exact JSON payload
4. Adjust the field names in the code node if needed

### Step 6: Share State Between Workflows

The two workflows need to share the "pending answer" data. The simplest approach:

**Recommended: Merge into one workflow** (see "Single Workflow Option" below).

If keeping two workflows, add an **HTTP Request** node at the end of Workflow 1 that POSTs the answer data to Workflow 2's webhook. Or use the n8n API approach:

```javascript
// In the Reply Handler's "Load Pending Answer" node, read Sender's data via API:
const executions = await this.helpers.httpRequest({
  method: 'GET',
  url: `https://dev.rrwo.us/api/v1/executions?workflowId=k4eXoRCzRowBDPpu&status=success&limit=1`,
  headers: { 'X-N8N-API-KEY': $env.N8N_API_KEY || 'YOUR_API_KEY_HERE' },
  json: true
});
```

### Step 7: Activate Both Workflows

1. Open each workflow in n8n
2. Toggle the **Active** switch to ON
3. Workflow 1 will start firing at 8am, 12:30pm, and 6pm ET
4. Workflow 2 will listen for incoming SMS replies 24/7 via the webhook

---

## Single Workflow Option (Recommended for Simplicity)

Instead of two separate workflows, you can combine everything into **one workflow** with two trigger paths. This eliminates the shared-state problem entirely:

```
Path 1: Schedule Trigger → Calculate Day/Slot → Load Question → Send SMS (Bird) → Store Answer (static data)
Path 2: Webhook (Bird inbound) → Load Answer (same static data) → Send Answer SMS (Bird)
```

**To do this:**
1. Create a new workflow with BOTH a Schedule Trigger AND a Webhook node
2. Each trigger feeds its own chain of nodes
3. Both chains use `$getWorkflowStaticData('global')` — since they're in the SAME workflow, they share the same static data automatically
4. No API calls or cross-workflow webhooks needed

This is the **recommended approach** and eliminates all cross-workflow complexity.

---

## Testing

### Test Workflow 1 (Question Sender)
1. Open Workflow 1 in n8n
2. Click **"Execute Workflow"** (manual trigger)
3. Check each node's output:
   - `Calculate Day & Slot` should show the current day number and slot
   - `Load Question Data` should find the matching question
   - `Send Question SMS` should send the SMS via Bird.com (check your phone!)

### Test Workflow 2 (Reply Handler)
1. Make sure Workflow 2 is **active**
2. Send a text message to your Bird.com number (any text, like "B")
3. You should receive the answer back within seconds

### Dry Run (No SMS)
To test without sending actual SMS:
1. Disable the MessageBird nodes temporarily
2. Run the workflow manually
3. Check the output of each node to verify the logic

### Test the Webhook Manually
You can also test the webhook with curl:
```bash
curl -X POST https://dev.rrwo.us/webhook-test/cam-sms-reply \
  -H "Content-Type: application/json" \
  -d '{"originator": "+1987654321", "body": "B"}'
```

---

## Customization

### Change Schedule Times
Edit the **Schedule Trigger** node's cron expression:
- Current: `0 8,12,18 * * *` (8am, 12pm, 6pm daily)
- Weekday only: `0 8,12,18 * * 1-5`
- Different times: `0 7,13,20 * * *` (7am, 1pm, 8pm)

### Add Multiple Students
Modify the **Send Question SMS** node to loop over an array of phone numbers:
```javascript
// In a Code node before MessageBird:
const students = ['+1111111111', '+2222222222', '+3333333333'];
return students.map(phone => ({ json: { ...items[0].json, recipients: phone } }));
```

### Track Progress
Add a **Google Sheets** node after "Store Pending Answer" to log:
- Day, slot, concept, question sent, timestamp
- Student's reply and whether it was correct
- Running score

### Add Scoring
In the Reply Handler, compare the student's answer to the correct answer letter:
```javascript
const correctLetter = pendingAnswer.charAt(2); // e.g., "B" from "✅ B)"
const studentLetter = incomingMessage.trim().toUpperCase().charAt(0);
const isCorrect = studentLetter === correctLetter;

// Append to the answer message
const scoreMsg = isCorrect 
  ? "🎉 CORRECT!\n\n" + pendingAnswer
  : "❌ Not quite.\n\n" + pendingAnswer;
```

---

## Cost Estimate

### Bird.com (MessageBird) SMS Costs
- **Outbound SMS (US):** ~€0.0070/message (~$0.0076)
- **Inbound SMS (US):** ~€0.0060/message (~$0.0065)
- **Virtual Number (US):** ~€1.00/month (~$1.09)
- Pricing varies by country — check [bird.com/pricing](https://www.bird.com/pricing)

### 30-Day Program Cost
| Item | Count | Cost |
|------|-------|------|
| Outbound questions | 90 | ~$0.68 |
| Outbound answers | 90 | ~$0.68 |
| Inbound replies | 90 | ~$0.59 |
| Virtual number | 1 month | ~$1.09 |
| **Total** | | **~$3.04** |

> The entire 30-day automated study program costs about **$3** in Bird.com fees.

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| No SMS received | Check MessageBird credentials (API key), phone number format (must include country code like +1), and account balance |
| Wrong question sent | Verify `START_DATE` in Calculate Day & Slot node matches your actual start date |
| Reply doesn't trigger answer | Check that Bird.com's inbound webhook URL points to `https://dev.rrwo.us/webhook/cam-sms-reply`. Workflow 2 must be **Active**. |
| Webhook receives data but wrong fields | Activate Workflow 2, send a test SMS, check the webhook node output, and adjust field names in "Load Pending Answer" code node |
| "No pending question" reply | The sender workflow hasn't run yet today, or the answer was already sent. Check execution logs. |
| Schedule not firing | Make sure the workflow is **Active** (toggled ON). Check timezone setting. |
| Static data not persisting | Ensure n8n is configured with persistent storage (not ephemeral). Check your n8n hosting setup. |

---

## Files Reference

| File | Purpose |
|------|---------|
| `sms_schedule.json` | The 90-message schedule — paste into the "Load Question Data" node |
| `SMS_STUDY_PROGRAM.md` | Human-readable version of the full 30-day program |
| `CAM_STUDY_GUIDE.md` | The comprehensive study guide (for reference) |

## Workflow IDs on Your Instance

| Workflow | ID | URL |
|----------|-----|-----|
| CAM Study - SMS Question Sender | `k4eXoRCzRowBDPpu` | `https://dev.rrwo.us/workflow/k4eXoRCzRowBDPpu` |
| CAM Study - SMS Reply Handler | `XbBybhVWkCutoj68` | `https://dev.rrwo.us/workflow/XbBybhVWkCutoj68` |
