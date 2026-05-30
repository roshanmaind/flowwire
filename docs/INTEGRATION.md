# Integration Guide — Connecting Workflows to a Node.js Backend

This document explains how your Node.js backend service interacts with the n8n workflows — both as a trigger (calling n8n webhooks) and as a receiver (being called by n8n).

---

## Architecture Overview

```
Node.js Backend (Express/Fastify)
        │
        │  ① POST /webhook/schedule-post       (triggers Workflow 04)
        │  ② GET  /api/internal/leads/…        (called by Workflow 01)
        │  ③ POST /api/internal/calls/missed   (called by Workflow 05)
        ▼
    n8n Engine
        │
        │  Reads/writes PostgreSQL
        │  Calls WhatsApp, Instagram, Facebook, LinkedIn, R2
        ▼
   External APIs + DB
```

n8n and your backend share the same PostgreSQL instance. n8n handles the messy API integrations; your backend handles business logic, auth, and serving the frontend.

---

## Shared Secret Authentication

Workflows that accept requests from the backend (Workflow 04) use a shared secret. Set it in `.env`:

```bash
BACKEND_API_SECRET=a-long-random-string-generated-with-openssl-rand-hex-32
```

Your backend must include this in the `Authorization` header:

```http
POST /webhook/schedule-post HTTP/1.1
Authorization: Bearer a-long-random-string-generated-with-openssl-rand-hex-32
Content-Type: application/json
```

Workflows that call your backend send the same secret in the request:

```js
// In your Express middleware
app.use('/api/internal', (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (token !== process.env.N8N_CALLBACK_SECRET) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
});
```

---

## Workflow 01 — Lead Capture

### What n8n does
Saves the lead to PostgreSQL, then (if no phone number) POSTs to your backend:

```
POST <BACKEND_API_URL>/api/internal/leads/notify-no-phone
Content-Type: application/json

{
  "leadgen_id": "123456789",
  "full_name": "Jane Doe",
  "email": "jane@example.com",
  "city": "Mumbai",
  "ad_name": "Summer Sale",
  "campaign_name": "Q1 Push",
  "status": "new"
}
```

### What your backend should do

```js
// routes/internal/leads.js
router.post('/notify-no-phone', async (req, res) => {
  const lead = req.body;

  // Example: create a task in your CRM for manual follow-up
  await crm.createTask({
    title: `Follow up with ${lead.full_name} (no phone)`,
    description: `Lead from ${lead.ad_name} — email: ${lead.email}`,
    priority: 'medium',
  });

  // Optionally send an internal Slack/email notification
  await slack.notify(`#sales`, `New lead with no phone: ${lead.full_name} (${lead.email})`);

  res.json({ received: true });
});
```

---

## Workflow 02 — Attendance Summary

This workflow is fully self-contained (cron → DB → WhatsApp). Your backend's only role is **populating the `check_ins` table** when the mobile app submits a GPS check-in.

### Mobile app → Backend → DB

```js
// POST /api/attendance/check-in
router.post('/check-in', authenticate, async (req, res) => {
  const { lat, lng, type } = req.body; // type: 'in' | 'out'
  const employeeId = req.user.id;

  if (type === 'in') {
    await db.query(
      `INSERT INTO check_ins (employee_id, check_in_time, lat_in, lng_in, status)
       VALUES ($1, NOW(), $2, $3, $4)`,
      [employeeId, lat, lng, isLate ? 'late' : 'present']
    );
  } else {
    await db.query(
      `UPDATE check_ins SET check_out_time = NOW(), lat_out = $1, lng_out = $2
       WHERE employee_id = $3 AND check_out_time IS NULL
       ORDER BY check_in_time DESC LIMIT 1`,
      [lat, lng, employeeId]
    );
  }

  res.json({ success: true });
});
```

n8n reads from this table at 6 PM — no real-time integration required.

---

## Workflow 03 — Expense Submission

This workflow is triggered by WhatsApp inbound (not your backend). After saving the expense, n8n creates a DB record with `status = 'pending_review'`.

### Your backend serves the review UI

```js
// GET /api/expenses?status=pending_review
router.get('/expenses', authenticate, requireRole('finance'), async (req, res) => {
  const { status = 'pending_review', page = 1 } = req.query;
  const expenses = await db.query(
    `SELECT * FROM expense_submissions WHERE status = $1 ORDER BY submitted_at DESC LIMIT 20 OFFSET $2`,
    [status, (page - 1) * 20]
  );
  res.json(expenses.rows);
});

// PATCH /api/expenses/:id/approve
router.patch('/expenses/:id/approve', authenticate, requireRole('finance'), async (req, res) => {
  await db.query(
    `UPDATE expense_submissions SET status = 'approved', reviewed_by = $1, reviewed_at = NOW() WHERE id = $2`,
    [req.user.id, req.params.id]
  );

  // Optionally trigger a WhatsApp notification to the employee here
  // by calling the WhatsApp API directly from the backend, or via a separate n8n webhook

  res.json({ approved: true });
});
```

---

## Workflow 04 — Social Media Scheduler

Your backend **triggers** this workflow. This is the primary integration point.

### Triggering a post

```js
// services/social-publisher.js
const axios = require('axios');

async function publishPost({ caption, imageUrl, platforms, scheduledAt }) {
  const n8nWebhookUrl = `${process.env.N8N_BASE_URL}/webhook/schedule-post`;

  const response = await axios.post(
    n8nWebhookUrl,
    {
      caption,
      image_url: imageUrl,
      platforms: platforms || ['instagram', 'facebook', 'linkedin'],
      scheduled_at: scheduledAt || new Date().toISOString(),
    },
    {
      headers: {
        'Authorization': `Bearer ${process.env.BACKEND_API_SECRET}`,
        'Content-Type': 'application/json',
      },
      timeout: 30_000,
    }
  );

  return response.data; // { post_id, ig_post_id, fb_post_id, li_post_id, … }
}

module.exports = { publishPost };
```

### Usage in a route

```js
// POST /api/posts/publish
router.post('/publish', authenticate, requireRole('marketing'), async (req, res) => {
  const { caption, imageUrl, platforms } = req.body;

  // Upload image to your CDN first
  const publicImageUrl = await cdn.upload(req.file);

  // Trigger n8n workflow
  const result = await socialPublisher.publishPost({
    caption,
    imageUrl: publicImageUrl,
    platforms,
  });

  res.json({ success: true, ...result });
});
```

### Scheduling future posts (optional)

If you want to schedule posts for a future time rather than publish immediately, implement a queue in your backend (e.g., BullMQ) that fires `publishPost()` at the scheduled time. n8n's webhook is the publish mechanism, not the scheduler.

---

## Workflow 05 — Missed Call Alert

n8n calls your backend **after** logging the missed call. Your backend can use this to create CRM follow-up tasks.

### What n8n POSTs to your backend

```
POST <BACKEND_API_URL>/api/internal/calls/missed
Content-Type: application/json

{
  "call_sid": "e3abcd1234567890",
  "caller_raw": "09876543210",
  "caller_phone": "919876543210",
  "called_phone": "918888880000",
  "status": "no-answer",
  "started_at": "2024-01-15T08:30:00.000Z",
  "caller_name": "Rahul Singh",
  "employee_id": 42,
  "department": "Sales",
  "whatsapp_msg": "Hi Rahul 👋 …"
}
```

### What your backend should do

```js
// routes/internal/calls.js
router.post('/missed', async (req, res) => {
  const call = req.body;

  // Create a follow-up task in your CRM
  await crm.createTask({
    title: `Callback: ${call.caller_name} (${call.caller_raw})`,
    due: addMinutes(new Date(), 30),
    assignedTo: call.department,
    priority: 'high',
    metadata: { call_sid: call.call_sid },
  });

  // Push a real-time notification to the sales dashboard
  await websocket.broadcast('sales-team', {
    type: 'MISSED_CALL',
    payload: call,
  });

  res.json({ received: true });
});
```

---

## Environment Variables Summary

Add these to your Node.js backend's `.env`:

```bash
# n8n integration
N8N_BASE_URL=http://localhost:5678          # or your production n8n URL
BACKEND_API_SECRET=same-value-as-in-n8n-env

# Shared database (same PostgreSQL instance)
DATABASE_URL=postgresql://n8n:n8n_secret@localhost:5433/n8n_db
```

---

## Error Handling Recommendations

- **Wrap n8n calls in try/catch** — if n8n is down, queue the request (Redis/BullMQ) and retry
- **Idempotency** — n8n workflows can fire more than once if retried; ensure your DB inserts use `ON CONFLICT DO NOTHING` where applicable
- **Timeouts** — set a 30-second timeout on calls to n8n webhooks; they involve external API calls that can be slow
- **Health check** — poll `http://localhost:5678/healthz` to detect if n8n is down before attempting to send posts
