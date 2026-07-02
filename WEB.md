# Binti Care — Web App API Documentation

**Version:** 1.0 (Draft)
**Scope:** Web application backend API — used by `CLINICIAN` and `FACILITY_ADMIN` roles
**Base URL:** `https://api.bintic.care/api` *(same backend as mobile — placeholder, update once deployed)*
**Protocol:** REST over HTTPS
**Authentication:** JWT Bearer tokens
**Companion document:** [Binti Care — Mobile API Documentation](./mobilebackend.md) — the canonical source for entities and endpoints shared across both platforms. This document does not repeat those definitions in full; see [Shared Entities (Reference)](#shared-entities-reference) below for a quick index of what lives where.

---

## Table of Contents

1. [Conventions](#conventions)
2. [Shared Entities (Reference)](#shared-entities-reference)
3. [Web-Only Entities](#web-only-entities)
4. [Authentication & Facility Onboarding](#1-authentication--facility-onboarding)
5. [Clinician Dashboard](#2-clinician-dashboard)
6. [Pregnancy Monitoring](#3-pregnancy-monitoring)
7. [Labour & Birth Monitoring](#4-labour--birth-monitoring)
8. [Postpartum & Baby Monitoring](#5-postpartum--baby-monitoring)
9. [Referral Network — Facility Side](#6-referral-network--facility-side)
10. [Facility & Staff Management](#7-facility--staff-management)
11. [Population & Reporting](#8-population--reporting)
12. [Education Content Management](#9-education-content-management)
13. [Facility-Managed Patients](#10-facility-managed-patients)
14. [Error Codes Reference](#error-codes-reference)

---

## Conventions

Identical to the mobile API: same response envelope (`{success, data, meta, error}`), same pagination shape, same ISO 8601 UTC timestamp convention, same JWT Bearer auth. See the [mobile documentation's Conventions section](./mobilebackend.md#conventions) for the full detail. Two additions specific to web:

### Headers

```http
Authorization: Bearer <jwt_access_token>
Content-Type: application/json
X-Facility-Context: fac_1029
```

`X-Facility-Context` is required on every endpoint scoped to a facility (which is most of this document). It tells the backend which facility the logged-in `CLINICIAN` or `FACILITY_ADMIN` is currently acting on behalf of. A staff member who somehow holds roles at two facilities switches context by changing this header rather than re-authenticating.

### Roles recap

| Role | Can do |
|---|---|
| `CLINICIAN` | View assigned patients, log vitals/labour readings, respond to entries, manage alerts, accept/handle referrals for their facility |
| `FACILITY_ADMIN` | Everything a `CLINICIAN` can do, plus: manage staff, manage facility profile/readiness, assign patients to clinicians, generate reports, manage education content, register the facility itself |

There is no CHP role in this system. Patients without a smartphone are registered and maintained directly by a `FACILITY_ADMIN` (see [Module 10](#10-facility-managed-patients)).

---

## Shared Entities (Reference)

These are defined in full in the mobile documentation and used as-is by the web app. Linked here for convenience — **do not redefine these independently in code**; both platforms must agree on one shape per entity.

| Entity | Defined in mobile doc | Web app's relationship to it |
|---|---|---|
| `User` | [Core Entities](./mobilebackend.md#user) | Web reads/writes `CLINICIAN` and `FACILITY_ADMIN` users too, not just `MOTHER` |
| `Profile` | [Core Entities](./mobilebackend.md#profile-extended-mother-specific-fields) | Web reads this when viewing a patient; writes `personalDoctorId`, `personalDoctorRequestStatus`, `preferredUnitIds` indirectly via assignment endpoints |
| `Consent` | [Core Entities](./mobilebackend.md#consent) | Web reads to determine whether a referral or QR scan can reveal patient data |
| `FormTemplate` | [Core Entities](./mobilebackend.md#formtemplate) | Web is the **only** place templates are created/extended (see [Module 10](#10-facility-managed-patients)) |
| `FormSubmission` | [Core Entities](./mobilebackend.md#formsubmission) | Web reads all submission types for review/feedback; can write on a patient's behalf |
| `CarePathwayTemplate` / `ScheduledVisit` | [Core Entities](./mobilebackend.md#carepathwaytemplate) | Web reads to display ANC/postnatal/vaccination schedules; can create manual ad-hoc visits |
| `LabourSession` / `LabourReading` | [Core Entities](./mobilebackend.md#laboursession) | Web **owns all writes** — every labour reading, alert, and resuscitation log entry is created from the web app |
| `BabyProfile` | [Core Entities](./mobilebackend.md#babyprofile) | Web reads; mobile owns creation |
| `Facility` | [Core Entities](./mobilebackend.md#facility) | Web owns all writes (registration, profile edits, readiness toggles) |
| `Referral` | [Core Entities](./mobilebackend.md#referral) | Mobile creates emergency requests; web handles accept/reject/complete and the receiving-side review |
| `Reminder` / `Notification` | [Core Entities](./mobilebackend.md#reminder) | Web can create reminders on a patient's behalf; notifications are read-only from web's perspective |

---

## Web-Only Entities

### StaffMember

```json
{
  "id": "staff_2210",
  "facilityId": "fac_1029",
  "userId": "usr_doc_4471",
  "role": "CLINICIAN",
  "specialty": "Obstetrics",
  "assignedPatientCount": 28,
  "status": "ACTIVE",
  "invitedAt": "2026-01-15T00:00:00Z",
  "joinedAt": "2026-01-16T09:00:00Z"
}
```

`status`: `ACTIVE` | `INVITE_PENDING` | `DEACTIVATED`. A `StaffMember` with `status: INVITE_PENDING` has no corresponding login-capable `User` yet — the invite email/SMS contains a signup link that, once completed, creates the `User` and flips this to `ACTIVE`.

### RiskAssessment

Represents the clinician-facing detail behind a patient's risk score — the system-calculated factors plus any clinical override. Mobile's `GET /api/pregnancy/risk-score` returns a simplified read of this same record; web is where overrides are written.

```json
{
  "id": "risk_4471",
  "pregnancyId": "preg_2207",
  "systemScore": 42,
  "systemLevel": "MEDIUM",
  "factors": [
    { "label": "Reduced fetal movement reported", "weight": 25, "severity": "WARNING" },
    { "label": "Blood pressure within normal range", "weight": 0, "severity": "SUCCESS" }
  ],
  "calculatedAt": "2026-06-29T08:30:00Z",
  "override": {
    "level": "LOW",
    "reason": "Reduced fetal movement resolved after follow-up call",
    "overriddenBy": "usr_doc_4471",
    "overriddenAt": "2026-06-29T09:00:00Z"
  }
}
```

`override` is `null` until a clinician explicitly sets one. The **effective** risk level shown anywhere in the system (mobile, web alerts, dashboards) is `override.level` if present, otherwise `systemLevel`.

### PopulationSnapshot

The aggregate shape returned by the trends/reporting endpoints. Not a persisted entity in the traditional sense — computed on read from underlying `FormSubmission`, `ScheduledVisit`, and `Referral` records, but documented here as a stable response contract.

```json
{
  "facilityId": "fac_1029",
  "periodStart": "2026-01-01",
  "periodEnd": "2026-06-30",
  "ancAttendanceRate": [
    { "month": "2026-01", "ratePercent": 72 },
    { "month": "2026-06", "ratePercent": 88 }
  ],
  "topRiskFlags": [
    { "flag": "REDUCED_FETAL_MOVEMENT", "count": 34 },
    { "flag": "ELEVATED_BP", "count": 27 }
  ],
  "referralVolumeByMonth": [
    { "month": "2026-01", "count": 12 },
    { "month": "2026-06", "count": 21 }
  ],
  "totals": {
    "totalPatients": 312,
    "averageAncVisitsPerPatient": 6.4,
    "adolescentPatientPercent": 22,
    "referralAcceptanceRatePercent": 94
  }
}
```

### Report

```json
{
  "id": "rpt_5512",
  "facilityId": "fac_1029",
  "type": "MONTHLY_FACILITY_SUMMARY",
  "dateRangeStart": "2026-05-01",
  "dateRangeEnd": "2026-05-31",
  "format": "PDF",
  "status": "READY",
  "fileUrl": "https://cdn.bintic.care/reports/rpt_5512.pdf",
  "generatedAt": "2026-06-27T10:00:00Z"
}
```

`type`: `MONTHLY_FACILITY_SUMMARY` | `ANC_ATTENDANCE` | `REFERRAL_ACTIVITY` | `RISK_FLAG_SUMMARY` | `MOH_RMNCAH_SUBMISSION`. `format`: `PDF` | `CSV` | `EXCEL`. `status`: `GENERATING` | `READY` | `FAILED`.

---

## 1. Authentication & Facility Onboarding

### `POST /api/auth/login`

Same endpoint and shape as the mobile documentation's [`POST /api/auth/login`](./mobilebackend.md#post-apiauthlogin) — a `CLINICIAN` or `FACILITY_ADMIN` logs in with the same mechanism a `MOTHER` does. The returned `User.role` determines which app shell the frontend renders.

**Request body:** `{ "phoneNumber": "+254712000111", "password": "SecurePass123" }` *(or `email` instead of `phoneNumber` — both are accepted login identifiers for staff accounts)*

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "user": { "...": "User entity, role: CLINICIAN or FACILITY_ADMIN" },
    "staffMemberships": [
      { "facilityId": "fac_1029", "facilityName": "Kilifi County Hospital", "role": "CLINICIAN" }
    ],
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600
  }
}
```

`staffMemberships` lists every facility this user holds a role at — almost always exactly one, but the shape allows for multi-facility staff. The frontend uses this to set the initial `X-Facility-Context`.

---

### `POST /api/facilities/register`

Registers a new maternal unit and creates its first `FACILITY_ADMIN` account in one step. **No auth token required** — this is the entry point for a brand-new facility with no existing account.

**Request body:**

```json
{
  "facility": {
    "name": "Kilifi County Hospital",
    "type": "PUBLIC",
    "county": "Kilifi",
    "address": "Off Mombasa-Malindi Road, Kilifi Town",
    "phoneNumber": "+254712000111",
    "email": "info@kilificountyhospital.go.ke",
    "latitude": -3.6309,
    "longitude": 39.8499,
    "servicesOffered": ["ANTENATAL_CARE", "DELIVERY", "NEONATAL_ICU"]
  },
  "adminAccount": {
    "fullName": "Faith Achieng",
    "email": "faith.achieng@kilificountyhospital.go.ke",
    "phoneNumber": "+254712000112",
    "password": "SecurePass123"
  }
}
```

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "facility": { "...": "Facility entity, status: PENDING_VERIFICATION" },
    "adminUser": { "...": "User entity, role: FACILITY_ADMIN" },
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
  }
}
```

A newly registered `Facility` has `status: PENDING_VERIFICATION` and does not appear in mobile's `GET /api/facilities/nearby` results until an Anthropic/PPH-Foundation-side (or future MOH-side) reviewer flips it to `status: VERIFIED`. This prevents unverified facilities from receiving live emergency referrals. *(This verification workflow is out of scope for the hackathon build but documented here so the field exists in the schema from day one.)*

**Errors:** `409 FACILITY_EMAIL_ALREADY_REGISTERED`, `400 VALIDATION_ERROR`.

---

### `GET /api/auth/me/landing-summary`

Powers the role-based landing screen shown immediately after login — a quick "what's waiting for you" view before committing to a destination.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "activeAlertCount": 4,
    "activeLabourSessionCount": 2,
    "pendingReferralCount": 2
  }
}
```

---

## 2. Clinician Dashboard

### `GET /api/dashboard/summary`

Powers the dashboard home stat cards.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "assignedPatientCount": 28,
    "assignedPatientCountDeltaThisWeek": 3,
    "activeAlertCount": 4,
    "ancVisitsToday": 6,
    "ancVisitsCompletedToday": 2,
    "pendingReferralCount": 2
  }
}
```

For a `FACILITY_ADMIN`, the same endpoint returns facility-wide totals instead of personal caseload — see `GET /api/facility-admin/overview` in [Module 10](#10-facility-managed-patients) for the admin-specific version with additional fields (unassigned patients, staff workload).

---

### `GET /api/dashboard/alerts`

**Query params:** `severity` (`CRITICAL` | `WARNING`), `page`, `pageSize`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "id": "alrt_8821",
      "patientUserId": "usr_8f3a2b1c",
      "patientName": "Wanjiru Kamau",
      "type": "REDUCED_FETAL_MOVEMENT",
      "severity": "WARNING",
      "message": "Reduced fetal movement reported",
      "sourceSubmissionId": "sub_7734",
      "createdAt": "2026-06-29T08:10:00Z",
      "acknowledgedAt": null
    }
  ]
}
```

This aggregates flagged `FormSubmission` records, labour `Alert` records, and depression-screening flags into one unified alert feed scoped to the logged-in clinician's assigned patients (or facility-wide for a `FACILITY_ADMIN`).

---

### `PUT /api/dashboard/alerts/{id}/acknowledge`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Alert entity, acknowledgedAt set" } }`

---

### `GET /api/dashboard/anc-visits-today`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    { "scheduledVisitId": "visit_anc_6", "patientName": "Faith Mwende", "scheduledAt": "2026-06-29T11:45:00Z", "purpose": "Blood pressure screening", "status": "SCHEDULED" }
  ]
}
```

---

### `GET /api/patients`

The patient search/filter view — searches across all patients registered at the current `X-Facility-Context`, not just the logged-in clinician's own caseload.

**Query params:**

| Param | Type | Notes |
|---|---|---|
| `query` | string | Matches name, phone number, or patient ID |
| `stage` | enum | `CYCLE_TRACKING` \| `PREGNANT` \| `POSTPARTUM` |
| `riskLevel` | enum | `LOW` \| `MEDIUM` \| `HIGH` |
| `assignment` | enum | `ASSIGNED_TO_ME` \| `UNASSIGNED` \| `ALL` |
| `page`, `pageSize` | int | Pagination |

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "userId": "usr_8f3a2b1c",
      "fullName": "Wanjiru Kamau",
      "age": 24,
      "patientCode": "10293",
      "phoneNumber": "+254712345678",
      "stage": "PREGNANT",
      "stageDetail": "Week 31",
      "riskLevel": "MEDIUM",
      "assignedClinicianName": "Dr. Achieng Otieno",
      "lastActivityAt": "2026-06-29T08:10:00Z",
      "preferredFacilityName": "Kilifi County Hospital"
    }
  ],
  "meta": { "...": "pagination" }
}
```

---

### `GET /api/patients/{userId}/timeline`

The cross-module timeline view — every `FormSubmission`, `ScheduledVisit`, clinician feedback message, and alert for one patient, merged into a single chronological feed.

**Query params:** `filter` (`ALL` | `VITALS` | `VISITS` | `FLAGS`), `page`, `pageSize`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "type": "FORM_SUBMISSION",
      "isFlagged": true,
      "title": "Reduced fetal movement reported",
      "summary": "Patient logged reduced fetal movement via mobile app",
      "occurredAt": "2026-06-29T08:10:00Z",
      "sourceId": "sub_7734",
      "actions": ["RESPOND"]
    },
    {
      "type": "SCHEDULED_VISIT",
      "isFlagged": false,
      "title": "ANC visit completed",
      "summary": "Visit 6 of 8 — routine check-up and BP screening. No concerns noted.",
      "occurredAt": "2026-06-26T10:30:00Z",
      "sourceId": "visit_anc_6",
      "actions": []
    },
    {
      "type": "CLINICIAN_FEEDBACK",
      "isFlagged": false,
      "title": "Message from clinician",
      "summary": "Mild ankle swelling is common at this stage, but let's keep an eye on it.",
      "occurredAt": "2026-06-24T10:00:00Z",
      "sourceId": "fb_4471",
      "actions": []
    }
  ],
  "meta": { "...": "pagination" }
}
```

`type`: `FORM_SUBMISSION` | `SCHEDULED_VISIT` | `CLINICIAN_FEEDBACK` | `LABOUR_EVENT` | `REFERRAL_EVENT`. `actions` is a hint to the frontend about which buttons to render for that timeline item (e.g. `RESPOND` shows the inline feedback box).

---

### `GET /api/patients/{userId}/overview`

Powers the patient detail sidebar — pregnancy summary, care team, emergency contact.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "patient": { "...": "User + Profile" },
    "pregnancySummary": {
      "dueDate": "2026-08-27",
      "gestationalAge": "31 weeks, 2 days",
      "ancVisitsCompleted": 6,
      "ancVisitsTotal": 8,
      "lastBloodPressure": "122/80",
      "lastWeightKg": 68.4
    },
    "careTeam": [
      { "userId": "usr_doc_4471", "fullName": "Dr. Achieng Otieno", "role": "Assigned clinician" }
    ],
    "emergencyContact": { "name": "James Kamau", "relationship": "Husband", "phoneNumber": "+254721556002" }
  }
}
```

`pregnancySummary` is `null` if the patient's current stage is not `PREGNANT` — the frontend renders a `postpartumSummary` or `cycleSummary` block instead, following the same general shape.

---

## 3. Pregnancy Monitoring

### `GET /api/patients/{userId}/pregnancy-vitals`

Equivalent to mobile's [`GET /api/pregnancy/patients/{patientId}/vitals`](./mobilebackend.md#get-apipregnancypatientspatientidvitals) — documented again here under the web-facing path for clarity, same underlying `FormSubmission` records.

**Query params:** `filter` (`ALL` | `FLAGGED` | `VITALS_ONLY` | `SYMPTOMS_ONLY`), `page`, `pageSize`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity, with patient-facing answers and any attached Feedback entities" } ] }`

---

### `POST /api/patients/{userId}/pregnancy-vitals/{submissionId}/feedback`

Same endpoint as mobile's [`POST /api/pregnancy/vitals/{id}/feedback`](./mobilebackend.md#post-apipregnancyvitalsidfeedback). Repeated here because the web UI's "respond to entry" inline box also supports a review-only action with no message:

**Request body (with message):** `{ "message": "Mild ankle swelling is common at this stage, but let's keep an eye on it." }`

**Request body (review only, no message):** `{ "markReviewedOnly": true }`

**Response `201 Created`** (with message): `{ "success": true, "data": { "...": "Feedback entity" } }`

**Response `200 OK`** (review only): `{ "success": true, "data": { "submissionId": "sub_7734", "reviewedAt": "2026-06-29T08:30:00Z", "reviewedBy": "usr_doc_4471" } }`

---

### `GET /api/patients/{userId}/risk-assessment`

**Response `200 OK`:** `{ "success": true, "data": { "...": "RiskAssessment entity" } }`

---

### `PUT /api/patients/{userId}/risk-assessment/override`

Sets or clears a clinician's manual override on the system-calculated risk score.

**Request body** (set an override):

```json
{
  "level": "LOW",
  "reason": "Reduced fetal movement resolved after follow-up call, downgrading to low risk"
}
```

**Request body** (clear an override, revert to system score): `{ "level": null }`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated RiskAssessment entity" } }`

This is the only way a risk level shown to the patient on mobile differs from the raw system calculation — see the `override` field on the `RiskAssessment` entity above for how the two values coexist.

---

## 4. Labour & Birth Monitoring

This module owns all writes to the `LabourSession` and `LabourReading` entities defined in the [mobile documentation's Core Entities](./mobilebackend.md#laboursession) — mobile only reads a simplified status view. Full endpoint detail for session creation, readings, the partograph data shape, alerts, and the resuscitation protocol is already documented in the mobile doc's [Module 5: Labour & Birth Monitor](./mobilebackend.md#5-labour--birth-monitor); every endpoint listed there is called from the web app in practice. This section adds the facility-wide views that only make sense on web.

### `GET /api/labour-sessions/active`

Facility-wide list of all currently active labour sessions, used by the Labour Alerts Panel.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "labourSessionId": "lab_5512",
      "patientName": "Joyce Kemunto",
      "room": "Room 3",
      "activeLabourStartedAt": "2026-06-29T05:30:00Z",
      "hoursElapsed": 4.33,
      "currentDilationCm": 6,
      "status": "ACTION_LINE_CROSSED",
      "assignedClinicianName": "Dr. Achieng Otieno"
    }
  ]
}
```

`status`: `NORMAL` | `WATCH` | `ACTION_LINE_CROSSED` | `POSTPARTUM_WATCH`. This is a computed summary status, distinct from `LabourSession.status` (`ACTIVE`/`CLOSED`) — it reflects the clinical urgency level shown in the alerts panel.

---

### `GET /api/labour-sessions/alerts-summary`

Powers the Labour Alerts module's stat row (active sessions, critical count, watch-list count) and the severity-sorted combined alert feed across all sessions.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "activeSessionCount": 3,
    "criticalAlertCount": 1,
    "watchAlertCount": 2,
    "alerts": [
      {
        "labourSessionId": "lab_5512",
        "patientName": "Joyce Kemunto",
        "room": "Room 3",
        "severity": "CRITICAL",
        "message": "Dilation crossed the action line",
        "assignedClinicianName": "Dr. Achieng Otieno"
      }
    ]
  }
}
```

---

### Room/bed assignment

### `PUT /api/labour-sessions/{id}/room`

**Request body:** `{ "room": "Room 3" }`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated LabourSession entity, with room field" } }`

`room` is a simple display-string field on `LabourSession`, not yet a fully modeled bed-management entity — sufficient for the hackathon build's room-based session listing.

---

## 5. Postpartum & Baby Monitoring

> [!NOTE]
> **Architectural Note: Separate Entities**
> During the postnatal period, the mother and the baby (or babies) are tracked as completely **separate clinical entities** linked by a `PregnancyRecord`. 
> - **MotherPostnatalContext**: Tracks her physical recovery (lochia, c-section wound), breast health (mastitis), and mental health (EPDS screenings).
> - **BabyProfile (1-to-Many)**: Tracks the newborn's pediatric milestones, growth percentiles, and feeding schedules. 
> This strict separation ensures clean data handling for twins/multiples and prevents overlapping clinical milestones.
### `GET /api/postpartum-alerts/summary`

Facility-wide postpartum and newborn alert feed, separated into maternal and newborn streams.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "postpartumPatientCount": 11,
    "criticalAlertCount": 1,
    "watchAlertCount": 3,
    "maternalAlerts": [
      {
        "patientUserId": "usr_gc_1102",
        "patientName": "Grace Chepkoech",
        "dayPostpartum": 0,
        "severity": "CRITICAL",
        "message": "Heavy postpartum bleeding reported",
        "sourceSubmissionId": "sub_9821",
        "createdAt": "2026-06-29T07:58:00Z"
      }
    ],
    "newbornAlerts": [
      {
        "babyId": "baby_3301",
        "babyName": "Amani",
        "motherName": "Mary Njoki",
        "dayOfLife": 12,
        "severity": "WARNING",
        "message": "Temperature slightly low, 2 readings",
        "createdAt": "2026-06-29T05:00:00Z"
      }
    ]
  }
}
```

---

### `GET /api/postpartum-patients`

**Query params:** `filter` (`ALL` | `FLAGGED` | `THIS_WEEK`), `page`, `pageSize`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "patientUserId": "usr_mn_1102",
      "patientName": "Mary Njoki",
      "dayPostpartum": 12,
      "babyName": "Amani",
      "babySex": "FEMALE",
      "status": "WATCH",
      "assignedClinicianName": "Dr. Achieng Otieno"
    }
  ],
  "meta": { "...": "pagination" }
}
```

`status`: `NORMAL` | `WATCH` | `CRITICAL` — computed from active alerts on the patient and her baby, same severity model used in labour alerts.

---

## 6. Referral Network — Facility Side

The `Referral` entity and its core lifecycle endpoints (`POST /api/referrals`, accept/reject/complete, patient-summary) are fully documented in the [mobile doc's Module 7](./mobilebackend.md#7-universal-referral-network--facilities) — mobile creates referrals, web (this module) is where a facility reviews and acts on them. This section covers the web-specific inbox and tracking views layered on top of that same data.

### `GET /api/referrals/inbox`

Incoming referral requests for the current `X-Facility-Context`, sorted with emergencies first.

**Query params:** `status` (default: `PENDING`), `page`, `pageSize`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "referralId": "ref_4471",
      "patientName": "Patience Moraa",
      "patientAge": 32,
      "fromFacilityName": "Malindi Sub-County Hospital",
      "reason": "SPECIALIST_REFERRAL",
      "reasonDisplay": "Suspected preeclampsia",
      "notes": "BP 158/102 on last reading. Requesting transfer for specialist obstetric care.",
      "isEmergency": true,
      "estimatedArrivalMinutes": 25,
      "requestedAt": "2026-06-29T08:30:00Z"
    }
  ],
  "meta": { "...": "pagination" }
}
```

---

### `GET /api/referrals/outgoing`

Referrals this facility has sent **to** other facilities — the mirror view of the inbox.

**Query params:** `status`, `page`, `pageSize`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "referralId": "ref_2291",
      "patientName": "Faith Mwende",
      "toFacilityName": "Coast General Teaching & Referral Hospital",
      "reason": "SPECIALIST_REFERRAL",
      "reasonDisplay": "Suspected gestational diabetes, needs specialist review",
      "status": "ACCEPTED",
      "sentAt": "2026-06-28T09:00:00Z"
    }
  ],
  "meta": { "...": "pagination" }
}
```

---

### `POST /api/referrals/outgoing`

A facility-initiated outgoing referral (as opposed to mobile's emergency-request flow) — used for routine specialist transfers or a patient's own request to move facilities. Same underlying `Referral` entity and creation contract as mobile's `POST /api/referrals`, with `isEmergency: false` in the typical case.

**Request body:**

```json
{
  "patientUserId": "usr_fm_3301",
  "toFacilityId": "fac_2002",
  "reason": "SPECIALIST_REFERRAL",
  "notes": "Suspected gestational diabetes, needs specialist review",
  "isEmergency": false
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "Referral entity" } }`

---

## 7. Facility & Staff Management

The `Facility` entity and its core read/write endpoints (`GET/PUT /api/facilities/{id}`, `PUT /api/facilities/{id}/availability`, staff add/remove) are fully documented in the [mobile doc's Module 7](./mobilebackend.md#7-universal-referral-network--facilities), since mobile reads facility data too. This section covers the web-only staff workload and invite-flow views.

### `GET /api/facility-admin/staff`

**Role required:** `FACILITY_ADMIN`.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "staffId": "staff_2210",
      "userId": "usr_doc_4471",
      "fullName": "Dr. Achieng Otieno",
      "role": "CLINICIAN",
      "specialty": "Obstetrics",
      "assignedPatientCount": 28,
      "capacity": 40,
      "status": "ACTIVE"
    },
    {
      "staffId": "staff_3302",
      "userId": null,
      "fullName": null,
      "invitedEmail": "jane.muthoni@kilificountyhospital.go.ke",
      "role": "CLINICIAN",
      "status": "INVITE_PENDING"
    }
  ]
}
```

`capacity` is a configurable soft cap per staff member used to drive the workload bar shown in the admin UI (e.g. "37 / 40 — near capacity"); it does not block assignment, it only flags the admin's attention.

---

### `POST /api/facility-admin/staff/invite`

**Request body:**

```json
{
  "email": "jane.muthoni@kilificountyhospital.go.ke",
  "role": "CLINICIAN",
  "specialty": "Midwifery"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "StaffMember entity, status: INVITE_PENDING" } }`

**Side effect:** an invite email (or SMS, if no email provided) is sent with a signup link scoped to this facility and role.

---

### `POST /api/facility-admin/staff/{staffId}/resend-invite`

**Response `200 OK`:** `{ "success": true, "data": { "resentAt": "2026-06-29T09:00:00Z" } }`

---

### `PUT /api/facility-admin/staff/{staffId}/capacity`

**Request body:** `{ "capacity": 35 }`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated StaffMember entity" } }`

---

### `PUT /api/facility-admin/staff/{staffId}/deactivate`

**Response `200 OK`:** `{ "success": true, "data": { "...": "StaffMember entity, status: DEACTIVATED" } }`

Deactivating a staff member does not delete their historical records (feedback messages, readings they recorded) — it only revokes login and removes them from future assignment pools. Their existing assigned patients must be reassigned via `PUT /api/facility-admin/patients/{patientId}/assign-clinician` (see the [mobile doc's Module 11](./mobilebackend.md#11-facility-managed-patients--manual-entry)).

---

## 8. Population & Reporting

### `GET /api/facility-admin/overview`

Admin-specific version of the dashboard summary, with facility-wide fields a `CLINICIAN`'s personal dashboard doesn't need.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "totalPatientCount": 312,
    "totalPatientCountDeltaThisWeek": 14,
    "unassignedPatientCount": 7,
    "activeClinicianCount": 5,
    "facilityWideAlertCount": 5,
    "weekAtAGlance": {
      "ancVisitsCompleted": 48,
      "ancVisitsScheduled": 52,
      "deliveries": 9,
      "referralsAccepted": 4,
      "referralsSentOut": 3,
      "postnatalFollowUpsDue": 6
    }
  }
}
```

---

### `GET /api/population-trends`

**Role required:** `CLINICIAN` or `FACILITY_ADMIN`.

**Query params:** `months` (default 6)

**Response `200 OK`:** `{ "success": true, "data": { "...": "PopulationSnapshot entity" } }`

---

### `GET /api/reports`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Report entity" } ], "meta": { "...": "pagination" } }`

---

### `POST /api/reports`

**Role required:** `FACILITY_ADMIN`.

**Request body:**

```json
{
  "type": "MOH_RMNCAH_SUBMISSION",
  "dateRangeStart": "2026-01-01",
  "dateRangeEnd": "2026-03-31",
  "format": "EXCEL"
}
```

**Response `202 Accepted`:** `{ "success": true, "data": { "...": "Report entity, status: GENERATING" } }`

Report generation is asynchronous — the client polls `GET /api/reports/{id}` (or listens for a `REPORT_READY` notification) until `status` becomes `READY` and `fileUrl` is populated.

---

### `GET /api/reports/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Report entity" } }`

---

## 9. Education Content Management

The `EducationContent` and `EducationEvent` entities and the patient-facing read endpoints (`GET /api/education/feed`, etc.) are documented in the [mobile doc's Module 9](./mobilebackend.md#9-education--community-engagement). The create/edit/publish endpoints listed there (`POST/PUT/DELETE /api/education/content`, `POST /api/education/events`) are exactly the ones the web content-management screen calls — no separate web-specific versions exist. This section covers the one additional web-only view: content performance.

### `GET /api/education/content/{id}/stats`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "contentId": "edu_201",
    "viewCount": 412,
    "status": "PUBLISHED",
    "publishedAt": "2026-06-26T00:00:00Z"
  }
}
```

`status`: `DRAFT` | `PUBLISHED`. The content list screen on web shows this status and view count inline per item; mobile's feed only ever shows `PUBLISHED` content.

---

## 10. Facility-Managed Patients

This is the web-side home of everything documented in the [mobile doc's Module 11: Facility-Managed Patients & Manual Entry](./mobilebackend.md#11-facility-managed-patients--manual-entry) — `POST /api/facility-admin/patients`, `GET /api/facility-admin/patients`, `PUT /api/facility-admin/patients/{patientId}/assign-clinician`, `POST /api/facility-admin/patients/{patientId}/entries/{context}`, `POST /api/facility-admin/form-templates`, and `GET /api/facility-admin/patients/{patientId}/full-history`. Refer there for full request/response detail on each. This section adds the bulk-assignment workflow specific to the web admin UI's patient-assignment screen.

**Emergency / referral access to full patient history:** the "View patient history" button shown on a patient's record screen calls `GET /api/facility-admin/patients/{patientId}/full-history`, consent-gated exactly like the QR-scan path mobile uses (`GET /api/profile/lookup/{qrToken}/full-history`). Both paths return the identical full cross-module record shape and both are recorded in the patient's access log — see the mobile doc's [Personal Health Profile module](./mobilebackend.md#2-personal-health-profile) for the complete consent and access-log model shared by both access methods.

### `GET /api/facility-admin/patients/unassigned`

Convenience filtered view (equivalent to `GET /api/facility-admin/patients?assignedClinicianId=unassigned`), returning the shape the assignment screen's table renders directly.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "patientUserId": "usr_cw_2210",
      "fullName": "Carol Wambui",
      "stage": "PREGNANT",
      "stageDetail": "Week 9",
      "registeredAt": "2026-06-29T06:00:00Z",
      "isReferralFromOtherFacility": false
    },
    {
      "patientUserId": "usr_pm_3398",
      "fullName": "Patience Moraa",
      "stage": "PREGNANT",
      "stageDetail": "Week 38",
      "registeredAt": "2026-06-29T08:30:00Z",
      "isReferralFromOtherFacility": true,
      "referralFromFacilityName": "Malindi Sub-County Hospital"
    }
  ]
}
```

---

### `POST /api/facility-admin/patients/bulk-assign`

**Request body:**

```json
{
  "patientUserIds": ["usr_cw_2210", "usr_en_1100"],
  "clinicianId": "usr_doc_4471"
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": { "assignedCount": 2, "clinicianId": "usr_doc_4471" }
}
```

Internally this calls `PUT /api/facility-admin/patients/{patientId}/assign-clinician` with `reason: GENERAL_ASSIGNMENT` once per patient — exposed as a single bulk endpoint purely to keep the frontend's "select multiple rows, assign to..." interaction to one network call.

---

## Error Codes Reference

In addition to every code listed in the [mobile doc's Error Codes Reference](./mobilebackend.md#error-codes-reference) (which apply identically here, since both platforms share one backend), the web app surfaces a few additional codes:

| HTTP Status | Code | Meaning |
|---|---|---|
| 400 | `FACILITY_CONTEXT_REQUIRED` | Request is missing the required `X-Facility-Context` header |
| 403 | `FACILITY_CONTEXT_MISMATCH` | The logged-in user holds no role at the facility specified in `X-Facility-Context` |
| 409 | `FACILITY_EMAIL_ALREADY_REGISTERED` | Facility registration attempted with an email already in use |
| 409 | `STAFF_CAPACITY_EXCEEDED` | Assignment would push a clinician past a hard capacity limit, if one is configured as enforced rather than advisory |
| 422 | `REPORT_GENERATION_FAILED` | Async report generation failed; `Report.status` will be `FAILED` |

---

*End of web app API documentation.*