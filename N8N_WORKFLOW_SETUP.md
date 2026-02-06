# n8n SMS Study Program — Setup Guide

> **Two workflows have already been created on your n8n instance at `https://dev.rrwo.us`.**  
> This guide walks you through configuring them to go live.

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
│                                          Send SMS (Twilio)──►│
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
│  Twilio Trigger ──► Load Pending Answer ──► Send Answer SMS │
│  (incoming SMS)     (from static data)      (via Twilio)    │
└──────────────────────────────────────────────────────────────┘
```

### How It Works
1. **3x per day** (8am, 12:30pm, 6pm ET), Workflow 1 fires
2. It calculates which **day** of the 30-day program it is and which **time slot** (morning/midday/evening)
3. It looks up the matching question from `sms_schedule.json`
4. It **sends the question** via Twilio SMS
5. It **stores the correct answer** in n8n's persistent static data
6. When the student **replies with ANY text** (their guess), Workflow 2 fires
7. Workflow 2 **retrieves the stored answer** and **sends it back** via SMS

---

## Prerequisites

### 1. Twilio Account (Required)
You need a Twilio account with an SMS-capable phone number.

1. **Sign up** at [twilio.com](https://www.twilio.com/) (free trial gives you $15 credit)
2. **Get a phone number** — Go to Console → Phone Numbers → Buy a Number (pick one with SMS capability)
3. **Note these values:**
   - **Account SID** (found on dashboard)
   - **Auth Token** (found on dashboard)
   - **Twilio Phone Number** (the number you bought, e.g., `+1234567890`)

### 2. Configure Twilio Webhook for Incoming SMS
This is how Twilio tells n8n when someone replies.

1. Go to **Twilio Console → Phone Numbers → Manage → Active Numbers**
2. Click your phone number
3. Scroll to **Messaging Configuration**
4. Under **"A message comes in"**, set:
   - **Webhook URL:** `https://dev.rrwo.us/webhook/cam-sms-reply` *(this will be the Twilio Trigger URL from Workflow 2)*
   - **HTTP Method:** `POST`
5. **Save**

> ⚠️ **Note:** The exact webhook URL will be generated when you activate Workflow 2. Check the Twilio Trigger node in n8n to get the production URL, then paste it into Twilio.

### 3. n8n Credentials Setup

In your n8n instance (`https://dev.rrwo.us`):

1. Go to **Settings → Credentials → Add Credential**
2. Search for **"Twilio"**
3. Enter:
   - **Account SID:** Your Twilio Account SID
   - **Auth Token:** Your Twilio Auth Token
4. **Save**

### 4. Environment Variables

Set these in your n8n environment (or use the Set node to hardcode them):

| Variable | Value | Example |
|----------|-------|---------|
| `TWILIO_PHONE_NUMBER` | Your Twilio number | `+1234567890` |
| `STUDENT_PHONE_NUMBER` | The student's phone number | `+1987654321` |

**To set environment variables in n8n:**
- If self-hosted: Add to your `.env` file or Docker environment
- Alternative: Replace `$env.TWILIO_PHONE_NUMBER` and `$env.STUDENT_PHONE_NUMBER` in the Twilio nodes with hardcoded values

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

### Step 3: Configure Twilio Nodes

In **both workflows**, click each Twilio node and:
1. Select your **Twilio credential** (created in Prerequisites step 3)
2. Verify the `from` and `to` phone numbers are correct
3. If not using environment variables, hardcode the numbers directly

### Step 4: Share State Between Workflows

The two workflows need to share the "pending answer" data. The simplest approach:

**Recommended: Use a shared webhook approach**

Add an HTTP Request node at the end of Workflow 1 that calls a webhook on Workflow 2 to pass the answer data. Or use the approach already built in — both workflows use `$getWorkflowStaticData('global')`.

**Since static data is per-workflow**, the cleanest solution is to modify Workflow 2's "Load Pending Answer" node to call the n8n API to read Workflow 1's static data:

```javascript
// In the Reply Handler's "Load Pending Answer" node:
const incomingMessage = $input.first().json.Body || $input.first().json.body || '';
const fromNumber = $input.first().json.From || $input.first().json.from || '';

// Read the Sender workflow's execution data via n8n API
const SENDER_WORKFLOW_ID = 'k4eXoRCzRowBDPpu';

// Use n8n's internal API to get the latest execution
const executions = await this.helpers.httpRequest({
  method: 'GET',
  url: `${$env.N8N_API_URL || 'https://dev.rrwo.us'}/api/v1/executions?workflowId=${SENDER_WORKFLOW_ID}&status=success&limit=1`,
  headers: {
    'X-N8N-API-KEY': $env.N8N_API_KEY || 'YOUR_API_KEY_HERE'
  },
  json: true
});

// Alternative simpler approach: Store answer in a file or use a Set node
// For now, use this workflow's own static data (populated by a webhook from Workflow 1)
const staticData = $getWorkflowStaticData('global');

const pendingAnswer = staticData.pendingAnswer || null;
const alreadyAnswered = staticData.answered || false;

if (!pendingAnswer || alreadyAnswered) {
  return [{ json: {
    hasAnswer: false,
    reply: alreadyAnswered 
      ? "You already answered this one! Next question coming soon. 📚"
      : "No pending question. Next question coming at the scheduled time! 📱",
    fromNumber: fromNumber
  }}];
}

staticData.answered = true;

return [{ json: {
  hasAnswer: true,
  answer: pendingAnswer,
  fromNumber: fromNumber,
  studentAnswer: incomingMessage
}}];
```

**SIMPLEST ALTERNATIVE: Merge into one workflow** (see "Single Workflow Option" below).

### Step 5: Activate Both Workflows

1. Open each workflow in n8n
2. Toggle the **Active** switch to ON
3. Workflow 1 will start firing at 8am, 12:30pm, and 6pm ET
4. Workflow 2 will listen for incoming SMS replies 24/7

---

## Single Workflow Option (Recommended for Simplicity)

Instead of two separate workflows, you can combine everything into **one workflow** with two trigger paths. This eliminates the shared-state problem entirely:

```
Path 1: Schedule Trigger → Calculate Day/Slot → Load Question → Send SMS → Store Answer (static data)
Path 2: Twilio Trigger → Load Answer (same static data) → Send Answer SMS
```

**To do this:**
1. Create a new workflow with BOTH a Schedule Trigger AND a Twilio Trigger
2. Each trigger feeds its own chain of nodes
3. Both chains use `$getWorkflowStaticData('global')` — since they're in the SAME workflow, they share the same static data automatically
4. No API calls or webhooks needed between workflows

This is the **recommended approach** and eliminates all cross-workflow complexity.

---

## Testing

### Test Workflow 1 (Question Sender)
1. Open Workflow 1 in n8n
2. Click **"Execute Workflow"** (manual trigger)
3. Check each node's output:
   - `Calculate Day & Slot` should show the current day number and slot
   - `Load Question Data` should find the matching question
   - `Send Question SMS` should send the SMS (check your phone!)

### Test Workflow 2 (Reply Handler)
1. Make sure Workflow 2 is **active**
2. Send a text message to your Twilio number (any text, like "B")
3. You should receive the answer back within seconds

### Dry Run (No SMS)
To test without sending actual SMS:
1. Disable the Twilio nodes temporarily
2. Run the workflow manually
3. Check the output of each node to verify the logic

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
// In a Code node before Twilio:
const students = ['+1111111111', '+2222222222', '+3333333333'];
return students.map(phone => ({ json: { ...items[0].json, to: phone } }));
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

### Twilio SMS Costs
- **Outbound SMS:** ~$0.0079/message (US)
- **Inbound SMS:** ~$0.0075/message (US)
- **Phone number:** ~$1.15/month

### 30-Day Program Cost
| Item | Count | Cost |
|------|-------|------|
| Outbound questions | 90 | $0.71 |
| Outbound answers | 90 | $0.71 |
| Inbound replies | 90 | $0.68 |
| Phone number | 1 month | $1.15 |
| **Total** | | **~$3.25** |

> The entire 30-day automated study program costs about **$3.25** in Twilio fees.

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| No SMS received | Check Twilio credentials, phone number format (must include +1), and account balance |
| Wrong question sent | Verify `START_DATE` in Calculate Day & Slot node matches your actual start date |
| Reply doesn't trigger answer | Check Twilio webhook URL points to your n8n Twilio Trigger URL. Must be the production URL, not test. |
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
