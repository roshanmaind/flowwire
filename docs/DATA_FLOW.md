# Data Flow Diagrams

Mermaid diagrams showing how data moves through each of the five automation workflows.

---

## Workflow 01 — Lead Capture

```mermaid
flowchart TD
    A([Meta Ads Platform\nUser submits Lead Ad form]) -->|POST webhook| B

    B["🔗 Meta Ads Webhook\n/webhook/meta-ads-lead"]
    B --> C["⚙ Parse Lead Payload\nExtract leadgen_id from\nentry › changes › value"]

    C -->|leadgen_id| D["🌐 Fetch Lead Details\nGET /v19.0/{leadgen_id}\n?fields=field_data,ad_name,…"]
    D <-->|Graph API call| META[(Meta Graph API)]

    D -->|field_data array| E["⚙ Flatten Field Data\nConvert field_data list\nto flat key-value object"]

    E -->|full_name · phone · email · city| F["🗄 Save Lead to PostgreSQL\nINSERT INTO leads\nON CONFLICT leadgen_id DO NOTHING"]
    F <--> PG[(PostgreSQL\nleads table)]

    F --> G{Has Phone\nNumber?}

    G -->|YES| H["💬 Send WhatsApp Welcome\nTemplate: lead_welcome\nvia WhatsApp Cloud API"]
    G -->|NO| I["📡 Notify Backend\nPOST /api/internal/leads\n/notify-no-phone"]

    H <--> WA[(WhatsApp\nCloud API)]
    I <--> BE[(Node.js\nBackend)]
```

---

## Workflow 02 — Attendance Summary

```mermaid
flowchart TD
    A([⏰ Cron: 18:00 Mon–Sat\nAsia/Kolkata]) --> B

    B["📅 Daily 6 PM Trigger"]
    B --> C["🗄 Fetch Today's Check-ins\nSELECT check_ins JOIN employees\nWHERE date = CURRENT_DATE"]
    C <--> PG1[(PostgreSQL\ncheck_ins · employees)]

    C -->|N rows| D["⚙ Aggregate & Build Report\n• Group by department\n• Count present / absent / late\n• Format WhatsApp message"]

    D --> E["🗄 Fetch Manager Phones\nSELECT DISTINCT manager_phone\nFROM employees WHERE active"]
    E <--> PG2[(PostgreSQL\nemployees)]

    E -->|manager phone list| F["⚙ Merge Report + Recipients\n1 item per manager"]

    F -->|fan-out| G["💬 Send Report via WhatsApp\nPlain-text message\nruns once per manager"]
    G <--> WA[(WhatsApp\nCloud API)]

    G --> H["🗄 Archive Report to DB\nINSERT INTO attendance_reports"]
    H <--> PG3[(PostgreSQL\nattendance_reports)]
```

**Sample WhatsApp output:**

```
📋 Attendance Report — 15 Jan 2024
━━━━━━━━━━━━━━━━━━━━━━
✅ Present : 28  ❌ Absent : 4  ⏰ Late : 3

*Sales* (P:10 A:1 L:2)
  ✅ Aditya Kumar — In: 09:02 | Out: 18:30 | 9.47h
  ⏰ Priya Sharma — In: 09:45 | Out: 18:15 | 8.5h
```

---

## Workflow 03 — Expense Submission

```mermaid
flowchart TD
    EMP([👤 Employee\nSends receipt photo on WhatsApp\nCaption: ₹450 taxi]) -->|inbound message event| A

    A["🔗 WhatsApp Inbound Webhook\n/webhook/whatsapp-inbound"]
    A --> B["⚙ Parse WhatsApp Payload\nExtract: from · media_id\nmime_type · caption"]

    B --> C{Message\ntype = image?}
    C -->|NO| STOP([🛑 Stop — ignore\nnon-image messages])
    C -->|YES| D

    D["🌐 Get Media Download URL\nGET /v19.0/{media_id}\n→ temporary CDN URL"]
    D <--> META[(Meta Graph API)]

    D -->|cdn_url| E["🌐 Download Receipt Image\nGET cdn_url with Bearer token\n→ binary blob in n8n memory"]
    E <--> CDN[(WhatsApp CDN)]

    E -->|binary| F["⚙ Build R2 Object Key\nexpenses/{phone}/{date}/{ts}.jpg"]

    F -->|key + binary| G["☁ Upload to Cloudflare R2\nS3-compatible PUT\nContent-Type: image/jpeg"]
    G <--> R2[(Cloudflare R2\nexpense-receipts bucket)]

    G --> H["🗄 Lookup Employee by Phone\nSELECT id, name FROM employees\nWHERE phone = sender"]
    H <--> PG1[(PostgreSQL\nemployees)]

    H -->|employee record| I["⚙ Build Expense Record\nParse amount via regex\n₹450 taxi → amount: 450"]

    I --> J["🗄 Save Expense to PostgreSQL\nINSERT INTO expense_submissions\nstatus = pending_review"]
    J <--> PG2[(PostgreSQL\nexpense_submissions)]

    J --> K["💬 Send WhatsApp Confirmation\n✅ Receipt received.\nRef: #ID · Amount: ₹450"]
    K <--> WA[(WhatsApp\nCloud API)]
```

---

## Workflow 04 — Social Media Scheduler

```mermaid
flowchart TD
    BE([Node.js Backend\nMarketing dashboard triggers post]) -->|"POST /webhook/schedule-post\nAuthorization: Bearer SECRET"| A

    A["🔗 Schedule Post Webhook\n/webhook/schedule-post\n🔒 Header Auth protected"]
    A --> B["⚙ Validate & Normalise Payload\n• Validate caption + image_url\n• Trim to platform char limits\n• Generate post_id"]

    B --> C{Platforms\nselected?}

    C -->|instagram| D["🔀 Post to Instagram?"]
    C -->|facebook| G["🔀 Post to Facebook?"]
    C -->|linkedin| J["🔀 Post to LinkedIn?"]

    D -->|YES| E["🌐 IG: Create Media Container\nPOST /v19.0/{ig_id}/media\nimage_url + caption"]
    E <--> IGAPI[(Instagram\nGraph API)]
    E --> F["🌐 IG: Publish Container\nPOST /v19.0/{ig_id}/media_publish\ncreation_id"]
    F <--> IGAPI

    G -->|YES| H["🌐 FB: Post Photo to Page\nPOST /v19.0/{page_id}/photos\nurl + message + page_token"]
    H <--> FBAPI[(Facebook\nGraph API)]

    J -->|YES| K["🌐 LI: Create UGC Post\nPOST /v2/ugcPosts\nauthor: org URN · media"]
    K <--> LIAPI[(LinkedIn API)]

    F --> L["⚙ Merge Publish Results\nCollect ig_post_id\nfb_post_id · li_post_id"]
    H --> L
    K --> L

    L --> M["🗄 Save Post Record to DB\nINSERT INTO social_posts\nall platform post IDs"]
    M <--> PG[(PostgreSQL\nsocial_posts)]
```

**Instagram requires a two-step publish** — container creation then publish — which this workflow handles sequentially before merging with the other branches.

---

## Workflow 05 — Missed Call Alert

```mermaid
flowchart TD
    CALLER([📞 Caller dials Exotel number\nCall goes unanswered / busy]) --> EXOTEL

    EXOTEL[(Exotel Platform\nStatus: no-answer · busy · failed)]
    EXOTEL -->|"POST form-encoded\nCallSid · From · To · Status"| A

    A["🔗 Exotel Missed Call Webhook\n/webhook/exotel-missed-call"]
    A --> B["⚙ Parse Exotel Payload\nNormalise phone to E.164\n0XXXXXXXXXX → 91XXXXXXXXXX"]

    B --> C{Is Missed\nCall?}
    C -->|"status = answered\nor completed"| STOP([🛑 Stop])
    C -->|"no-answer · busy\nfailed · missed"| D

    D["🗄 Lookup Caller in DB\nSELECT id · name · department\nFROM employees WHERE phone"]
    D <--> PG1[(PostgreSQL\nemployees)]

    D -->|employee or empty| E["⚙ Build WhatsApp Message\nPersonalise with caller name\nand formatted call time"]

    E --> F["💬 Send WhatsApp Callback\n'Hi Rahul 👋 We noticed\nyou called at 02:45 PM…'"]
    F <--> WA[(WhatsApp\nCloud API → caller's phone)]

    F --> G["🗄 Log Missed Call to DB\nINSERT INTO missed_calls\ncall_sid · phones · wa_sent_at"]
    G <--> PG2[(PostgreSQL\nmissed_calls)]

    G --> H["📡 Notify Backend Service\nPOST /api/internal/calls/missed\n→ CRM task · dashboard event"]
    H <--> BE[(Node.js Backend)]
```

**End-to-end latency:** typically under 3 seconds from missed call detection to WhatsApp delivery.

---

## Cross-Workflow Data Relationships

```mermaid
erDiagram
    employees {
        int id PK
        text name
        text phone
        text manager_phone
        text department
        bool active
    }

    leads {
        int id PK
        text leadgen_id UK
        text full_name
        text phone
        text email
        text city
        text ad_name
        text campaign_name
        text status
        timestamptz created_at
    }

    check_ins {
        int id PK
        int employee_id FK
        timestamptz check_in_time
        timestamptz check_out_time
        decimal lat_in
        decimal lng_in
        text status
    }

    attendance_reports {
        int id PK
        date report_date
        int total_present
        int total_absent
        int total_late
        jsonb report_json
        timestamptz sent_at
    }

    expense_submissions {
        int id PK
        int employee_id FK
        text phone
        text caption
        decimal amount
        text receipt_url
        text r2_key
        text status
        timestamptz submitted_at
    }

    social_posts {
        int id PK
        text post_id UK
        text caption
        text image_url
        text[] platforms
        text ig_post_id
        text fb_post_id
        text li_post_id
        timestamptz published_at
    }

    missed_calls {
        int id PK
        text call_sid
        text caller_phone
        text called_phone
        int employee_id FK
        text status
        timestamptz started_at
        timestamptz wa_sent_at
    }

    employees ||--o{ check_ins : "logs"
    employees ||--o{ expense_submissions : "submits"
    employees ||--o{ missed_calls : "identified as"
```

---

## Credential Sharing Map

```mermaid
graph LR
    subgraph Credentials
        PGcred("🔑 Business DB\nPostgreSQL")
        WAcred("🔑 WhatsApp\nCloud API")
        METAcred("🔑 Meta Ads\nAccess Token")
        R2cred("🔑 Cloudflare R2")
        IGcred("🔑 Instagram\nGraph API")
        FBcred("🔑 Facebook\nGraph API")
        LIcred("🔑 LinkedIn API")
    end

    subgraph Workflows
        WF1(WF1 · Lead Capture)
        WF2(WF2 · Attendance)
        WF3(WF3 · Expense)
        WF4(WF4 · Social)
        WF5(WF5 · Missed Call)
    end

    PGcred --> WF1
    PGcred --> WF2
    PGcred --> WF3
    PGcred --> WF4
    PGcred --> WF5

    WAcred --> WF1
    WAcred --> WF2
    WAcred --> WF3
    WAcred --> WF5

    METAcred --> WF1
    METAcred --> WF3

    R2cred --> WF3

    IGcred --> WF4
    FBcred --> WF4
    LIcred --> WF4
```
