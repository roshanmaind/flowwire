# Data Flow Diagrams

ASCII diagrams showing how data moves through each of the five automation workflows.

---

## Workflow 01 — Lead Capture

```
Meta Ads Platform
      │
      │  Lead Ad form submitted by user
      │
      ▼
┌─────────────────────────────┐
│   Meta Ads Webhook (POST)   │  ← https://<n8n>/webhook/meta-ads-lead
│   n8n receives raw payload  │
└──────────────┬──────────────┘
               │  { entry[].changes[].value.leadgen_id }
               ▼
┌─────────────────────────────┐
│   Parse Lead Payload        │
│   Extract leadgen_id from   │
│   nested entry/changes      │
└──────────────┬──────────────┘
               │  leadgen_id
               ▼
┌─────────────────────────────┐     ┌──────────────────────────────┐
│  Fetch Lead Details         │────▶│  Meta Graph API              │
│  GET /v19.0/{leadgen_id}    │◀────│  GET /v19.0/{leadgen_id}     │
│  ?fields=field_data,…       │     │  → { field_data: [...] }     │
└──────────────┬──────────────┘     └──────────────────────────────┘
               │  { field_data: [{ name, values }] }
               ▼
┌─────────────────────────────┐
│   Flatten Field Data        │
│   field_data array          │
│   → { full_name, phone,     │
│       email, city, … }      │
└──────────────┬──────────────┘
               │  flat lead object
               ▼
┌─────────────────────────────┐     ┌──────────────────────────────┐
│  Save Lead to PostgreSQL    │────▶│  PostgreSQL                  │
│  INSERT INTO leads          │     │  table: leads                │
│  ON CONFLICT (leadgen_id)   │     │  (deduped by leadgen_id)     │
│  DO NOTHING                 │     └──────────────────────────────┘
└──────────────┬──────────────┘
               │
               ▼
       ┌───────────────┐
       │ Has Phone?    │
       └──┬────────┬───┘
    YES   │        │  NO
          │        │
          ▼        ▼
 ┌──────────────┐  ┌──────────────────────┐
 │ WhatsApp     │  │ Notify Backend       │
 │ Cloud API    │  │ POST /api/internal/  │
 │ Template Msg │  │ leads/notify-no-phone│
 │ → lead's     │  └──────────────────────┘
 │   phone      │
 └──────────────┘

Data at rest:
  PostgreSQL.leads { leadgen_id, full_name, phone, email, city, ad_name,
                     campaign_name, form_id, status, created_at }
```

---

## Workflow 02 — Attendance Summary

```
                    ┌──────────────┐
                    │  Clock: 18:00│  Mon–Sat (Asia/Kolkata)
                    │  Cron Trigger│
                    └──────┬───────┘
                           │
                           ▼
              ┌────────────────────────┐    ┌───────────────────┐
              │ Fetch Today's Check-ins│───▶│   PostgreSQL      │
              │ SELECT check_ins JOIN  │◀───│   check_ins       │
              │ employees WHERE date   │    │   employees       │
              │ = CURRENT_DATE         │    └───────────────────┘
              └────────────┬───────────┘
                           │  [ rows… ]
                           ▼
              ┌────────────────────────┐
              │ Aggregate & Build      │
              │ Report                 │
              │                        │
              │  group by department   │
              │  count present/absent/ │
              │  late                  │
              │  format WA message     │
              └────────────┬───────────┘
                           │  { report, whatsapp_message }
                           ▼
              ┌────────────────────────┐    ┌───────────────────┐
              │ Fetch Manager Phones   │───▶│   PostgreSQL      │
              │ SELECT DISTINCT        │◀───│   employees       │
              │ manager_phone FROM     │    │   (active=true)   │
              │ employees              │    └───────────────────┘
              └────────────┬───────────┘
                           │  [ { manager_phone }, … ]
                           ▼
              ┌────────────────────────┐
              │ Merge Report +         │
              │ Recipients             │
              │  → 1 item per manager  │
              └────────────┬───────────┘
                           │  (fan-out: N items)
                           ▼
              ┌────────────────────────┐    ┌───────────────────┐
              │ Send via WhatsApp      │───▶│ WhatsApp Cloud API│
              │ (runs N times,         │    │ POST /messages    │
              │  once per manager)     │    │ type: text        │
              └────────────┬───────────┘    └───────────────────┘
                           │
                           ▼
              ┌────────────────────────┐    ┌───────────────────┐
              │ Archive Report to DB   │───▶│   PostgreSQL      │
              │ INSERT INTO            │    │   attendance_     │
              │ attendance_reports     │    │   reports         │
              └────────────────────────┘    └───────────────────┘
```

---

## Workflow 03 — Expense Submission

```
Employee's Phone
      │
      │  Sends photo on WhatsApp with caption "₹450 taxi"
      │
      ▼
┌──────────────────────────────────┐
│  WhatsApp Cloud API              │
│  (Meta infrastructure)           │
└──────────────────┬───────────────┘
                   │  POST webhook with message object
                   ▼
┌──────────────────────────────────┐
│  WhatsApp Inbound Webhook (n8n)  │  ← /webhook/whatsapp-inbound
└──────────────────┬───────────────┘
                   │  raw body
                   ▼
┌──────────────────────────────────┐
│  Parse WhatsApp Payload          │
│  Extract: from, media_id,        │
│  mime_type, caption              │
└──────────────────┬───────────────┘
                   │
             ┌─────┴──────┐
             │ Is Image?  │
             └──┬─────────┘
          YES   │        NO → (stop, ignore)
                ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Get Media Download URL          │───▶│  Meta Graph API         │
│  GET /v19.0/{media_id}           │◀───│  GET /v19.0/{media_id}  │
│  → { url: "https://cdn…" }       │    │  → { url }              │
└──────────────────┬───────────────┘    └─────────────────────────┘
                   │  temporary CDN URL
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Download Receipt Image          │───▶│  WhatsApp CDN           │
│  GET <url> (authenticated)       │◀───│  → binary image data    │
│  → binary blob in n8n memory     │    └─────────────────────────┘
└──────────────────┬───────────────┘
                   │  binary data
                   ▼
┌──────────────────────────────────┐
│  Build R2 Object Key             │
│  expenses/{phone}/{date}/{ts}.jpg│
└──────────────────┬───────────────┘
                   │  { r2_key, binary }
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Upload to Cloudflare R2         │───▶│  Cloudflare R2          │
│  S3-compatible PUT               │    │  Bucket: expense-       │
│                                  │    │  receipts               │
└──────────────────┬───────────────┘    └─────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Lookup Employee by Phone        │───▶│  PostgreSQL             │
│  SELECT id, name FROM employees  │◀───│  employees table        │
│  WHERE phone = $1                │    └─────────────────────────┘
└──────────────────┬───────────────┘
                   │  { id, name }
                   ▼
┌──────────────────────────────────┐
│  Build Expense Record            │
│  Parse amount from caption regex │
│  Assemble DB row                 │
└──────────────────┬───────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Save Expense to PostgreSQL      │───▶│  PostgreSQL             │
│  INSERT INTO expense_submissions │    │  expense_submissions    │
│  status = 'pending_review'       │    └─────────────────────────┘
└──────────────────┬───────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Send WhatsApp Confirmation      │───▶│  WhatsApp Cloud API     │
│  "✅ Receipt received. Ref: #12" │    │  → employee's phone     │
└──────────────────────────────────┘    └─────────────────────────┘
```

---

## Workflow 04 — Social Media Scheduler

```
Node.js Backend
      │
      │  POST /webhook/schedule-post
      │  { caption, image_url, platforms: ['instagram','facebook','linkedin'] }
      │  Authorization: Bearer <SECRET>
      │
      ▼
┌──────────────────────────────────┐
│  Schedule Post Webhook (n8n)     │
└──────────────────┬───────────────┘
                   │
                   ▼
┌──────────────────────────────────┐
│  Validate & Normalise Payload    │
│  - Check required fields         │
│  - Trim caption to platform limit│
│  - Generate post_id              │
└──────────────────┬───────────────┘
                   │
         ┌─────────┼──────────┐
         │         │          │
         ▼         ▼          ▼
   [Instagram?] [Facebook?] [LinkedIn?]
         │         │          │
    YES  │    YES  │     YES  │
         │         │          │
         ▼         ▼          ▼
  ┌──────────┐ ┌──────────┐ ┌──────────────────┐
  │IG: Create│ │FB: Photo │ │LI: UGC Post      │
  │ Media    │ │to Page   │ │POST /v2/ugcPosts  │
  │Container │ │/photos   │ │author: org URN   │
  └────┬─────┘ └────┬─────┘ └──────┬───────────┘
       │             │              │
       ▼             │              │
  ┌──────────┐       │              │
  │IG: Pub-  │       │              │
  │lish Con- │       │              │
  │tainer    │       │              │
  └────┬─────┘       │              │
       │             │              │
       └──────┬──────┘──────────────┘
              │  (merge — wait for all branches)
              ▼
┌──────────────────────────────────┐
│  Merge Publish Results           │
│  Collect ig_post_id, fb_post_id, │
│  li_post_id from each branch     │
└──────────────────┬───────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Save Post Record to DB          │───▶│  PostgreSQL             │
│  INSERT INTO social_posts        │    │  social_posts           │
└──────────────────────────────────┘    └─────────────────────────┘

External API calls made:
  Instagram  → POST /v19.0/{ig_id}/media           (create container)
               POST /v19.0/{ig_id}/media_publish   (publish)
  Facebook   → POST /v19.0/{page_id}/photos
  LinkedIn   → POST /v2/ugcPosts
```

---

## Workflow 05 — Missed Call Alert

```
Caller dials Exotel number
      │
      │  Call rings but goes unanswered / busy
      │
      ▼
┌──────────────────────────────────┐
│  Exotel Platform                 │
│  Status: no-answer / busy        │
└──────────────────┬───────────────┘
                   │  POST (form-encoded)
                   │  { CallSid, From, To, Status, StartTime, … }
                   ▼
┌──────────────────────────────────┐
│  Exotel Missed Call Webhook (n8n)│  ← /webhook/exotel-missed-call
└──────────────────┬───────────────┘
                   │
                   ▼
┌──────────────────────────────────┐
│  Parse Exotel Payload            │
│  Normalise phone to E.164        │
│  (0XXXXXXXXXX → 91XXXXXXXXXX)    │
└──────────────────┬───────────────┘
                   │
             ┌─────┴──────────┐
             │ Is Missed Call?│
             └──┬─────────────┘
          YES   │   NO → (stop)
                ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Lookup Caller in DB             │───▶│  PostgreSQL             │
│  SELECT id, name, department     │◀───│  employees table        │
│  FROM employees                  │    └─────────────────────────┘
│  WHERE phone LIKE $1             │
└──────────────────┬───────────────┘
                   │  { id, name, department } or empty
                   ▼
┌──────────────────────────────────┐
│  Build WhatsApp Message          │
│  "Hi {name} 👋 We noticed you   │
│   called at {time}…"             │
└──────────────────┬───────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Send WhatsApp Callback Message  │───▶│  WhatsApp Cloud API     │
│  POST /messages type:text        │    │  → caller's phone       │
└──────────────────┬───────────────┘    └─────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Log Missed Call to DB           │───▶│  PostgreSQL             │
│  INSERT INTO missed_calls        │    │  missed_calls           │
└──────────────────┬───────────────┘    └─────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────┐    ┌─────────────────────────┐
│  Notify Backend Service          │───▶│  Node.js Backend        │
│  POST /api/internal/calls/missed │    │  → CRM task creation    │
│                                  │    │  → dashboard push event │
└──────────────────────────────────┘    └─────────────────────────┘

Total latency: typically < 3 seconds from missed call to WhatsApp delivery
```

---

## Cross-Workflow Data Relationships

```
                    ┌─────────────────────────────┐
                    │        PostgreSQL            │
                    │                             │
    WF01 ──writes──▶│  leads                      │
    WF02 ──reads───▶│  employees                  │
    WF02 ──reads───▶│  check_ins                  │
    WF02 ──writes──▶│  attendance_reports          │
    WF03 ──reads───▶│  employees                  │
    WF03 ──writes──▶│  expense_submissions         │
    WF04 ──writes──▶│  social_posts               │
    WF05 ──reads───▶│  employees                  │
    WF05 ──writes──▶│  missed_calls               │
                    │                             │
                    └─────────────────────────────┘

Shared credential across all workflows: "Business DB — PostgreSQL"
Shared credential for WF01, WF03, WF05: "WhatsApp Cloud API"
```
