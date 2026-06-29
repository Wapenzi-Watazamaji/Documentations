# Binti Care — Mobile API Documentation

**Version:** 1.1 (Draft) — Revised: 3-role model (Mother, Clinician, Facility Admin), dynamic schema-driven forms, auto-instantiated MOH care pathways
**Scope:** Mobile application (mother/woman-facing) backend API
**Base URL:** `https://api.bintic.care/api` *(placeholder — update once deployed)*
**Protocol:** REST over HTTPS
**Authentication:** JWT Bearer tokens

---

## Table of Contents

1. [Conventions](#conventions)
2. [Core Entities](#core-entities)
3. [Authentication & User Management](#1-authentication--user-management)
4. [Personal Health Profile](#2-personal-health-profile)
5. [Menstrual & Cycle Tracker](#3-menstrual--cycle-tracker)
6. [Pregnancy Journey Tracker](#4-pregnancy-journey-tracker)
7. [Labour & Birth Monitor](#5-labour--birth-monitor)
8. [Postpartum & Baby Tracker](#6-postpartum--baby-tracker)
9. [Universal Referral Network & Facilities](#7-universal-referral-network--facilities)
10. [Reminders, Notifications & SMS](#8-reminders-notifications--sms)
11. [Education & Community Engagement](#9-education--community-engagement)
12. [AI Assistant & Personal Doctor Chat](#10-ai-assistant--personal-doctor-chat)
13. [Facility-Managed Patients & Manual Entry](#11-facility-managed-patients--manual-entry)
14. [Error Codes Reference](#error-codes-reference)

---

## Conventions

### Base headers

Unless otherwise noted, every authenticated request must include:

```http
Authorization: Bearer <jwt_access_token>
Content-Type: application/json
Accept-Language: en | sw
```

`Accept-Language` drives which language content (education posts, AI assistant responses, system messages) is returned in. Defaults to `en` if omitted.

### Standard response envelope

All responses follow this shape:

```json
{
  "success": true,
  "data": { },
  "meta": { },
  "error": null
}
```

On failure:

```json
{
  "success": false,
  "data": null,
  "meta": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Phone number is already registered",
    "fields": {
      "phoneNumber": "Already in use"
    }
  }
}
```

### Pagination

List endpoints accept `page` and `pageSize` query parameters and return:

```json
{
  "success": true,
  "data": [ ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 134,
    "totalPages": 7
  }
}
```

### Timestamps

All timestamps are ISO 8601 UTC strings, e.g. `"2026-06-29T08:30:00Z"`. The client is responsible for converting to local display time.

### Offline sync fields

Any `POST` endpoint that creates a record accepts an optional `clientGeneratedId` (UUID v4) and `clientCreatedAt` (ISO 8601). This allows the mobile app to create records while offline and sync them later without losing the true creation time or risking duplicate inserts.

```json
{
  "clientGeneratedId": "550e8400-e29b-41d4-a716-446655440000",
  "clientCreatedAt": "2026-06-29T06:12:00Z"
}
```

The server treats `clientGeneratedId` as an idempotency key — replaying the same request with the same ID will not create a duplicate record.

### Roles

| Role | Description |
|---|---|
| `MOTHER` | Primary mobile app user, a pregnant/postpartum/cycle-tracking woman |
| `CLINICIAN` | Doctor/nurse/midwife at a registered facility (web-only, referenced here for shared entities) |
| `FACILITY_ADMIN` | Manages a facility's staff, patients, and settings, including manual registration of patients without smartphones (web-only) |

This document covers **mobile-facing endpoints**, used primarily by users with role `MOTHER`. Endpoints shared with the web app (e.g. facility lookup, manual patient registration) are included where mobile depends on them or where the design decision affects the mobile data model.

---

## Core Entities

This section defines the canonical shape of every entity referenced throughout the API. Endpoint-specific request/response bodies reference these by name.

### User

```json
{
  "id": "usr_8f3a2b1c",
  "role": "MOTHER",
  "fullName": "Wanjiru Kamau",
  "phoneNumber": "+254712345678",
  "dateOfBirth": "2002-03-14",
  "gender": "FEMALE",
  "preferredLanguage": "en",
  "county": "Kilifi",
  "profilePhotoUrl": "https://cdn.bintic.care/photos/usr_8f3a2b1c.jpg",
  "createdAt": "2026-04-01T09:00:00Z",
  "updatedAt": "2026-06-20T11:15:00Z"
}
```

### Profile (extended mother-specific fields)

```json
{
  "userId": "usr_8f3a2b1c",
  "currentStage": "PREGNANT",
  "preferredUnitIds": ["fac_1029", "fac_1031"],
  "emergencySharingPreference": "ASK_FIRST",
  "contactPreference": "BOTH",
  "emergencyContact": {
    "name": "James Kamau",
    "relationship": "Husband",
    "phoneNumber": "+254721556002"
  },
  "personalDoctorId": "usr_doc_4471",
  "personalDoctorRequestStatus": "ASSIGNED",
  "qrPassportToken": "bc_qr_9f8e7d6c"
}
```

`personalDoctorRequestStatus`: `NONE` | `PENDING_FACILITY_REVIEW` | `ASSIGNED`. A mother does not pick a doctor directly — she requests one from her preferred facility, and the facility admin performs the actual assignment using the same mechanism used for general patient-to-clinician routing (see [Module 11](#11-facility-managed-patients--manual-entry)).

`currentStage` is one of: `CYCLE_TRACKING`, `PREGNANT`, `POSTPARTUM`.

### Consent

```json
{
  "id": "con_5521",
  "userId": "usr_8f3a2b1c",
  "granteeType": "FACILITY",
  "granteeId": "fac_1029",
  "granteeName": "Kilifi County Hospital",
  "active": true,
  "grantedAt": "2026-04-01T09:05:00Z",
  "revokedAt": null
}
```

`granteeType` is one of: `FACILITY`, `CLINICIAN`.

### CycleEntry

```json
{
  "id": "cyc_3391",
  "userId": "usr_8f3a2b1c",
  "startDate": "2026-06-01",
  "endDate": "2026-06-05",
  "flowLevel": "MODERATE",
  "clotLevel": "NONE",
  "flags": ["SOAKED_WITHIN_2_HOURS"],
  "pbacScore": 18,
  "createdAt": "2026-06-01T07:00:00Z"
}
```

`flowLevel`: `LIGHT` | `MODERATE` | `HEAVY` | `VERY_HEAVY`. `clotLevel`: `NONE` | `SMALL` | `LARGE`.

### PregnancyRecord

```json
{
  "id": "preg_2207",
  "userId": "usr_8f3a2b1c",
  "status": "ACTIVE",
  "lastMenstrualPeriod": "2025-11-20",
  "dueDate": "2026-08-27",
  "isFirstPregnancy": true,
  "gestationalAgeWeeks": 31,
  "gestationalAgeDays": 2,
  "endedAt": null,
  "outcome": null
}
```

`status`: `ACTIVE` | `ENDED`. `outcome` (once ended): `LIVE_BIRTH` | `STILLBIRTH` | `MISCARRIAGE`.

### FormTemplate

Defines the structure of any data-entry form in the system (pregnancy vitals, postpartum check-ins, baby vitals, etc.) as a configurable set of fields, rather than a fixed set of database columns. Facility admins can extend a template with extra fields for their own patients without a backend code change.

```json
{
  "id": "tmpl_preg_vitals_v2",
  "context": "PREGNANCY_VITALS",
  "version": 2,
  "facilityId": null,
  "isDefault": true,
  "fields": [
    { "key": "bloodPressureSystolic", "label": "Blood pressure (systolic)", "type": "NUMBER", "unit": "mmHg", "required": true },
    { "key": "bloodPressureDiastolic", "label": "Blood pressure (diastolic)", "type": "NUMBER", "unit": "mmHg", "required": true },
    { "key": "weightKg", "label": "Weight", "type": "NUMBER", "unit": "kg", "required": true },
    { "key": "temperatureCelsius", "label": "Temperature", "type": "NUMBER", "unit": "°C", "required": false },
    { "key": "symptoms", "label": "Symptoms", "type": "MULTI_SELECT", "options": ["SWELLING", "HEADACHE", "NAUSEA", "VISION_CHANGES", "REDUCED_FETAL_MOVEMENT", "VAGINAL_BLEEDING"], "flaggingOptions": ["VISION_CHANGES", "REDUCED_FETAL_MOVEMENT", "VAGINAL_BLEEDING"], "required": false },
    { "key": "fetalMovementCount", "label": "Fetal movement count", "type": "NUMBER", "required": false },
    { "key": "notes", "label": "Additional notes", "type": "TEXT", "required": false }
  ],
  "createdAt": "2026-01-10T00:00:00Z"
}
```

`context`: `CYCLE_ENTRY` | `CYCLE_SYMPTOM` | `PREGNANCY_VITALS` | `LABOUR_READING` | `MATERNAL_CHECKIN` | `BABY_VITALS` | `DEPRESSION_SCREENING` (any data-entry surface in the app). `facilityId: null` with `isDefault: true` marks the platform-wide default template for that context; a facility can create its own override by setting `facilityId` and `isDefault: false` — the active template for a patient is resolved as: facility-specific template if one exists, otherwise the platform default. Field `type`: `NUMBER` | `TEXT` | `SINGLE_SELECT` | `MULTI_SELECT` | `BOOLEAN` | `DATE`. `flaggingOptions` marks which selected values on a `MULTI_SELECT`/`SINGLE_SELECT` field should automatically flag the submission for clinician review.

### FormSubmission

The generic answer-record for any `FormTemplate`. All the previously-separate entry types (`VitalsEntry`, `CycleEntry`, `MaternalCheckin`, `BabyVitalsEntry`, etc.) are, under the hood, `FormSubmission` records scoped to a `context`. The API still exposes context-specific endpoints (e.g. `/api/pregnancy/vitals`) for convenience, but they all read/write this same underlying shape.

```json
{
  "id": "sub_7734",
  "templateId": "tmpl_preg_vitals_v2",
  "context": "PREGNANCY_VITALS",
  "subjectUserId": "usr_8f3a2b1c",
  "relatedEntityId": "preg_2207",
  "enteredBy": "usr_8f3a2b1c",
  "answers": {
    "bloodPressureSystolic": 122,
    "bloodPressureDiastolic": 80,
    "weightKg": 68.4,
    "temperatureCelsius": 36.7,
    "symptoms": ["REDUCED_FETAL_MOVEMENT"],
    "fetalMovementCount": 4,
    "notes": "Haven't felt baby move as much today"
  },
  "isFlagged": true,
  "createdAt": "2026-06-29T08:10:00Z"
}
```

`subjectUserId` is the mother/patient the submission is about. `relatedEntityId` links it to the relevant parent record (e.g. a `PregnancyRecord` ID, a `LabourSession` ID). `enteredBy` is normally equal to `subjectUserId` (self-reported); facility staff entering data manually on a patient's behalf will differ — see [Module 11](#11-facility-managed-patients--manual-entry). `isFlagged` is computed server-side from the template's `flaggingOptions` and any numeric thresholds configured on the template.

### CarePathwayTemplate

Represents a standard MOH care schedule (e.g. the 8-visit ANC schedule, the postnatal 48hr/1wk/6wk schedule, the infant vaccination schedule) that gets automatically instantiated into real scheduled visits for a patient once she reaches the relevant stage.

```json
{
  "id": "path_anc_moh_v1",
  "name": "Kenya MOH ANC Schedule",
  "appliesToStage": "PREGNANT",
  "milestones": [
    { "order": 1, "label": "1st ANC visit", "triggerWeek": 12, "purpose": "Booking visit, risk assessment" },
    { "order": 2, "label": "2nd ANC visit", "triggerWeek": 20, "purpose": "Routine check-up" },
    { "order": 8, "label": "8th ANC visit", "triggerWeek": 40, "purpose": "Pre-delivery check" }
  ]
}
```

`appliesToStage`: `PREGNANT` | `POSTPARTUM` | `NEWBORN`. Each `triggerWeek` is relative to gestational age (for `PREGNANT`) or age-in-weeks since birth (for `POSTPARTUM`/`NEWBORN`, e.g. the vaccination schedule).

### ScheduledVisit

A real, dated visit instance created automatically from a `CarePathwayTemplate` once a patient enters the relevant stage (e.g. the moment `POST /api/pregnancy/start` is called, the system instantiates all 8 ANC visits from `path_anc_moh_v1` against her due date).

```json
{
  "id": "visit_anc_6",
  "pathwayTemplateId": "path_anc_moh_v1",
  "milestoneOrder": 6,
  "subjectUserId": "usr_8f3a2b1c",
  "label": "6th ANC visit",
  "scheduledAt": "2026-06-26T10:30:00Z",
  "facilityId": "fac_1029",
  "status": "COMPLETED",
  "summary": "Weight 64kg, BP 118/76, no concerns noted"
}
```

`status`: `SCHEDULED` | `COMPLETED` | `MISSED` | `RESCHEDULED`. This entity replaces the previously separate `AncVisit` and postnatal-visit shapes — both ANC and postnatal/vaccination schedules are now `ScheduledVisit` records generated from the appropriate `CarePathwayTemplate`.

### LabourSession

```json
{
  "id": "lab_5512",
  "pregnancyId": "preg_2207",
  "facilityId": "fac_1029",
  "status": "ACTIVE",
  "activeLabourStartedAt": "2026-06-29T05:30:00Z",
  "closedAt": null,
  "outcome": null,
  "deliveryType": null
}
```

### LabourReading

```json
{
  "id": "read_8821",
  "labourSessionId": "lab_5512",
  "type": "DILATION",
  "value": 6,
  "unit": "cm",
  "recordedAt": "2026-06-29T09:48:00Z",
  "recordedBy": "usr_doc_4471"
}
```

`type`: `DILATION` | `FHR` | `MATERNAL_BP` | `CONTRACTIONS`.

### BabyProfile

```json
{
  "id": "baby_3301",
  "motherUserId": "usr_8f3a2b1c",
  "name": "Amani",
  "dateOfBirth": "2026-06-17",
  "timeOfBirth": "09:42",
  "sex": "FEMALE",
  "birthWeightKg": 3.2,
  "birthLengthCm": 49,
  "deliveryType": "VAGINAL",
  "placeOfBirth": "fac_1029"
}
```

### Facility

```json
{
  "id": "fac_1029",
  "name": "Kilifi County Hospital",
  "type": "PUBLIC",
  "county": "Kilifi",
  "address": "Off Mombasa-Malindi Road, Kilifi Town",
  "phoneNumber": "+254712000111",
  "latitude": -3.6309,
  "longitude": 39.8499,
  "servicesOffered": ["ANTENATAL_CARE", "DELIVERY", "NEONATAL_ICU"],
  "readiness": {
    "bloodBankStocked": true,
    "maternityBedsAvailable": true,
    "theatreAvailable": false,
    "neonatalIcuCapacity": true,
    "ambulanceOnStandby": true
  },
  "updatedAt": "2026-06-29T07:00:00Z"
}
```

### Referral

```json
{
  "id": "ref_4471",
  "patientUserId": "usr_8f3a2b1c",
  "fromFacilityId": "fac_2002",
  "toFacilityId": "fac_1029",
  "status": "ACCEPTED",
  "reason": "REDUCED_FETAL_MOVEMENT",
  "notes": "Patient reports significantly reduced movement since this morning",
  "createdAt": "2026-06-29T08:30:00Z",
  "acceptedAt": "2026-06-29T08:34:00Z",
  "completedAt": null
}
```

`status`: `PENDING` | `ACCEPTED` | `REJECTED` | `COMPLETED`. `reason` is a free-form enum matching the mobile emergency request reasons (see [Referral Network](#7-universal-referral-network--facilities)).

### Reminder

```json
{
  "id": "rem_2281",
  "userId": "usr_8f3a2b1c",
  "type": "ANC_VISIT",
  "title": "ANC visit — Week 34 check-up",
  "dueAt": "2026-07-06T09:00:00Z",
  "isDone": false,
  "createdAt": "2026-06-20T10:00:00Z"
}
```

`type`: `ANC_VISIT` | `VACCINATION` | `MEDICATION` | `CYCLE` | `CUSTOM`.

### Notification

```json
{
  "id": "notif_9931",
  "userId": "usr_8f3a2b1c",
  "category": "CLINICIAN_FEEDBACK",
  "title": "Dr. Achieng responded to your entry",
  "body": "Mild ankle swelling is common at this stage...",
  "isRead": false,
  "relatedEntityType": "VITALS_ENTRY",
  "relatedEntityId": "vit_7734",
  "createdAt": "2026-06-29T08:15:00Z"
}
```

---

## 1. Authentication & User Management

### `POST /api/auth/register`

Registers a new mobile user.

**Headers:** None beyond base headers (no auth token required — this is a public endpoint).

**Request body:**

```json
{
  "fullName": "Wanjiru Kamau",
  "phoneNumber": "+254712345678",
  "dateOfBirth": "2002-03-14",
  "gender": "FEMALE",
  "password": "SecurePass123",
  "consentAccepted": true
}
```

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "user": { "...": "User entity" },
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600
  }
}
```

**Errors:** `400 VALIDATION_ERROR` (weak password, invalid phone format), `409 PHONE_ALREADY_REGISTERED`.

---

### `POST /api/auth/login`

**Request body:**

```json
{
  "phoneNumber": "+254712345678",
  "password": "SecurePass123"
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "user": { "...": "User entity" },
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600
  }
}
```

**Errors:** `401 INVALID_CREDENTIALS`, `423 ACCOUNT_LOCKED` (after repeated failed attempts).

---

### `POST /api/auth/refresh`

**Request body:**

```json
{ "refreshToken": "eyJhbGciOiJIUzI1NiIs..." }
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600
  }
}
```

**Errors:** `401 INVALID_REFRESH_TOKEN`, `401 REFRESH_TOKEN_EXPIRED`.

---

### `POST /api/auth/logout`

**Headers:** `Authorization: Bearer <token>` required.

**Request body:** `{ "refreshToken": "eyJhbGciOiJIUzI1NiIs..." }`

**Response `204 No Content`**

---

### `POST /api/auth/register-sms-only`

This endpoint is **not called from the mobile app**. Patients without a smartphone are registered directly by a facility admin on the web app — see `POST /api/facility-admin/patients` in [Module 11](#11-facility-managed-patients--manual-entry). It is listed here only because mobile depends on the resulting account shape: an `accountType: SMS_ONLY` user can still appear in mobile-facing data (e.g. as a referral's patient record), even though she never logs into the app herself. All her reminders and notifications are delivered via SMS rather than push, since she has no app session.

---

### `POST /api/auth/forgot-password`

**Request body:** `{ "phoneNumber": "+254712345678" }`

**Response `200 OK`:** `{ "success": true, "data": { "otpSentTo": "+254712345678", "expiresInSeconds": 300 } }`

---

### `POST /api/auth/reset-password`

**Request body:**

```json
{
  "phoneNumber": "+254712345678",
  "otp": "482911",
  "newPassword": "NewSecurePass456"
}
```

**Response `200 OK`:** `{ "success": true, "data": null }`

**Errors:** `400 INVALID_OTP`, `410 OTP_EXPIRED`.

---

## 2. Personal Health Profile

### `GET /api/profile/me`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "user": { "...": "User entity" },
    "profile": { "...": "Profile entity" }
  }
}
```

---

### `PUT /api/profile/me`

**Request body** (all fields optional, partial update):

```json
{
  "fullName": "Wanjiru Kamau",
  "preferredLanguage": "sw",
  "county": "Kilifi"
}
```

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated User entity" } }`

---

### `POST /api/profile/me/photo`

**Headers:** `Content-Type: multipart/form-data`

**Request body:** form field `photo` (image file, max 5MB, jpg/png)

**Response `200 OK`:** `{ "success": true, "data": { "profilePhotoUrl": "https://cdn.bintic.care/photos/usr_8f3a2b1c.jpg" } }`

---

### `GET /api/profile/me/qr`

Generates or returns the current QR pregnancy passport token.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "qrToken": "bc_qr_9f8e7d6c",
    "qrImageUrl": "https://cdn.bintic.care/qr/bc_qr_9f8e7d6c.png",
    "expiresAt": null
  }
}
```

A `POST` to the same path (`POST /api/profile/me/qr/refresh`) regenerates the token, invalidating the old one.

---

### `GET /api/profile/lookup/{qrToken}`

Used by a facility scanning a mother's QR passport. Requires the requester to hold a `CLINICIAN` or `FACILITY_ADMIN` role token.

**Path parameters:** `qrToken` (string)

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "user": { "...": "User entity, limited fields" },
    "bloodType": "O+",
    "currentStage": "PREGNANT",
    "activeRiskFlags": ["REDUCED_FETAL_MOVEMENT"],
    "gestationalAgeWeeks": 31,
    "consentGranted": true
  }
}
```

If `consentGranted` is `false` (mother's emergency sharing preference is `ASK_FIRST` or `MANUAL` and she has not approved this facility), the response instead returns:

```json
{
  "success": true,
  "data": {
    "user": { "id": "usr_8f3a2b1c", "fullName": "Wanjiru Kamau" },
    "consentGranted": false,
    "consentRequestSent": true
  }
}
```

**Errors:** `404 QR_TOKEN_NOT_FOUND`, `410 QR_TOKEN_EXPIRED`.

---

### `PUT /api/profile/preferred-units`

**Request body:** `{ "facilityIds": ["fac_1029", "fac_1031"] }`

**Response `200 OK`:** `{ "success": true, "data": { "preferredUnitIds": ["fac_1029", "fac_1031"] } }`

---

### `GET /api/profile/preferred-units`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Facility entity" } ] }`

---

### `POST /api/profile/consent`

**Request body:**

```json
{
  "granteeType": "FACILITY",
  "granteeId": "fac_1029"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "Consent entity" } }`

---

### `GET /api/profile/consent`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Consent entity" } ] }`

---

### `DELETE /api/profile/consent/{consentId}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Consent entity, with active: false, revokedAt set" } }`

---

### `GET /api/profile/personal-doctor`

**Response `200 OK`** (doctor assigned):

```json
{
  "success": true,
  "data": {
    "status": "ASSIGNED",
    "doctor": {
      "id": "usr_doc_4471",
      "fullName": "Dr. Achieng Otieno",
      "specialty": "Obstetrics",
      "facilityId": "fac_1029",
      "bio": "10 years in maternal health, Kilifi County Hospital"
    }
  }
}
```

**Response `200 OK`** (request pending facility review):

```json
{ "success": true, "data": { "status": "PENDING_FACILITY_REVIEW", "requestedFacilityId": "fac_1029", "requestedAt": "2026-06-28T10:00:00Z" } }
```

**Response `200 OK`** (no request made):

```json
{ "success": true, "data": { "status": "NONE" } }
```

---

### `POST /api/profile/personal-doctor/request`

The mother requests a personal doctor from her preferred facility — she does not select a specific clinician. The facility admin reviews the request and performs the actual assignment (see `PUT /api/facility-admin/patients/{patientId}/assign-doctor` in [Module 11](#11-facility-managed-patients--manual-entry)).

**Request body:** `{ "facilityId": "fac_1029" }`

**Response `201 Created`:**

```json
{
  "success": true,
  "data": { "requestId": "docreq_2291", "status": "PENDING_FACILITY_REVIEW", "facilityId": "fac_1029" }
}
```

**Errors:** `400 NO_PREFERRED_FACILITY` if the mother has no facility relationship to request through.

---

### `PUT /api/profile/emergency-sharing-preference`

**Request body:** `{ "preference": "ASK_FIRST" }`

`preference`: `AUTO_SHARE` | `ASK_FIRST` | `MANUAL`.

**Response `200 OK`:** `{ "success": true, "data": { "preference": "ASK_FIRST" } }`

---

## 3. Menstrual & Cycle Tracker

### `GET /api/cycles/entries/form-template`

Returns the active `FormTemplate` for this patient's period-entry detail fields (flow level, clot level, flags). `startDate`/`endDate` are structural and not part of the template, since every period entry needs them regardless of facility customization.

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormTemplate entity, context: CYCLE_ENTRY" } }`

---

### `POST /api/cycles/entries`

**Request body:**

```json
{
  "startDate": "2026-06-01",
  "endDate": "2026-06-05",
  "templateId": "tmpl_cycle_entry_v1",
  "answers": {
    "flowLevel": "MODERATE",
    "clotLevel": "NONE",
    "flags": ["SOAKED_WITHIN_2_HOURS"]
  },
  "clientGeneratedId": "550e8400-e29b-41d4-a716-446655440000",
  "clientCreatedAt": "2026-06-01T07:00:00Z"
}
```

Default template fields: `flowLevel` (`SINGLE_SELECT`: `LIGHT` | `MODERATE` | `HEAVY` | `VERY_HEAVY`), `clotLevel` (`SINGLE_SELECT`: `NONE` | `SMALL` | `LARGE`), `flags` (`MULTI_SELECT`: `LEAKED_THROUGH_CLOTHING` | `CHANGED_AT_NIGHT` | `SOAKED_WITHIN_2_HOURS`). The server computes `pbacScore` from `flowLevel`/`clotLevel`/`flags` regardless of whether a facility has customized the template, using a fixed scoring table mapped against these standard keys — a facility that extends the template with additional custom fields does not affect PBAC scoring, which only ever reads these three keys if present.

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormSubmission entity, context: CYCLE_ENTRY, with startDate/endDate and computed pbacScore" } }`

---

### `GET /api/cycles/entries`

**Query params:** `page`, `pageSize`, `from` (date), `to` (date)

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity" } ], "meta": { "...": "pagination" } }`

---

### `GET /api/cycles/entries/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormSubmission entity" } }`

---

### `PUT /api/cycles/entries/{id}`

**Request body:** any subset of `startDate`, `endDate`, `answers`.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated FormSubmission entity" } }`

---

### `DELETE /api/cycles/entries/{id}`

**Response `204 No Content`**

---

### `POST /api/cycles/entries/{id}/pbac-items`

Logs an individual PBAC item (a single pad/tampon change or clot observation) tied to a specific day within a period entry. Useful when the simplified flow-level selector isn't precise enough and the team wants raw PBAC scoring.

**Request body:**

```json
{
  "date": "2026-06-02",
  "itemType": "PAD",
  "soakLevel": "FULLY_SOAKED",
  "pointValue": 5
}
```

`itemType`: `PAD` | `TAMPON` | `CLOT`. `soakLevel` (for PAD/TAMPON): `LIGHTLY_SOAKED` | `MODERATELY_SOAKED` | `FULLY_SOAKED`.

**Response `201 Created`:** `{ "success": true, "data": { "...": "PbacItem entity" } }`

---

### `GET /api/cycles/entries/{id}/pbac-items`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "PbacItem entity" } ] }`

---

### `GET /api/cycles/entries/{id}/pbac-score`

**Response `200 OK`:** `{ "success": true, "data": { "entryId": "sub_3391", "totalScore": 18, "isHmbRisk": false } }`

A score ≥ 100 (per standard PBAC clinical threshold) sets `isHmbRisk: true` and triggers the HMB flag (see below).

---

### `GET /api/cycles/symptoms/form-template`

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormTemplate entity, context: CYCLE_SYMPTOM" } }`

---

### `POST /api/cycles/symptoms`

**Request body:**

```json
{
  "date": "2026-06-12",
  "templateId": "tmpl_cycle_symptom_v1",
  "answers": {
    "mood": "LOW",
    "symptoms": ["CRAMPS", "FATIGUE"],
    "energyLevel": "LOW",
    "notes": ""
  },
  "clientGeneratedId": "660e8400-e29b-41d4-a716-446655440001"
}
```

Default template fields: `mood` (`SINGLE_SELECT`: `VERY_LOW` | `LOW` | `NEUTRAL` | `GOOD` | `VERY_GOOD`), `symptoms` (`MULTI_SELECT`: `CRAMPS` | `BLOATING` | `HEADACHE` | `FATIGUE` | `BREAST_TENDERNESS` | `ACNE` | `BACK_PAIN` | `NAUSEA`), `energyLevel` (`SINGLE_SELECT`: `LOW` | `MEDIUM` | `HIGH`), `notes` (`TEXT`).

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormSubmission entity, context: CYCLE_SYMPTOM" } }`

---

### `GET /api/cycles/symptoms`

**Query params:** `page`, `pageSize`, `from`, `to`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity" } ] }`

---

### `GET /api/cycles/predictions`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "nextPeriodPredictedDate": "2026-07-01",
    "ovulationWindowStart": "2026-06-15",
    "ovulationWindowEnd": "2026-06-19",
    "averageCycleLengthDays": 29,
    "currentCycleDay": 5
  }
}
```

---

### `GET /api/cycles/trends`

**Query params:** `months` (default 6)

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "cycleLengthHistory": [
      { "month": "2026-01", "averageLengthDays": 28 },
      { "month": "2026-02", "averageLengthDays": 30 }
    ],
    "insights": [
      { "type": "REGULARITY", "message": "Your cycles have been fairly regular over the last 6 months" },
      { "type": "DURATION_INCREASE", "message": "Your periods have lasted longer than usual twice this year" }
    ],
    "topSymptoms": [
      { "symptom": "CRAMPS", "count": 8 },
      { "symptom": "FATIGUE", "count": 6 }
    ]
  }
}
```

---

### `GET /api/cycles/hmb-status`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "isActive": true,
    "triggeredAt": "2026-06-05T08:00:00Z",
    "reasons": [
      "Changed pad within 2 hours on multiple days",
      "Passed large clots in your last 2 periods",
      "Period lasted longer than 7 days last month"
    ]
  }
}
```

---

### `POST /api/cycles/hmb-status/acknowledge`

**Request body:** `{ "action": "DISMISSED" }` or `{ "action": "TALK_TO_DOCTOR" }`

**Response `200 OK`:** `{ "success": true, "data": { "isActive": false } }`

---

## 4. Pregnancy Journey Tracker

### `POST /api/pregnancy/start`

**Request body:**

```json
{
  "dateInputType": "LMP",
  "lastMenstrualPeriod": "2025-11-20",
  "isFirstPregnancy": true
}
```

`dateInputType`: `LMP` | `DUE_DATE`. If `DUE_DATE`, send `dueDate` instead of `lastMenstrualPeriod`; the server back-calculates the other.

**Response `201 Created`:** `{ "success": true, "data": { "...": "PregnancyRecord entity" } }`

**Side effect:** the user's `currentStage` in their Profile is set to `PREGNANT`, and cycle tracking is suspended.

---

### `GET /api/pregnancy/current`

**Response `200 OK`:** `{ "success": true, "data": { "...": "PregnancyRecord entity" } }`

**Errors:** `404 NO_ACTIVE_PREGNANCY`

---

### `PUT /api/pregnancy/current`

**Request body:** `{ "dueDate": "2026-08-30" }` *(clinician-revised due date)*

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated PregnancyRecord entity" } }`

---

### `POST /api/pregnancy/end`

**Request body:**

```json
{
  "endedAt": "2026-08-27T14:20:00Z",
  "outcome": "LIVE_BIRTH"
}
```

**Response `200 OK`:** `{ "success": true, "data": { "...": "PregnancyRecord entity, status: ENDED" } }`

**Side effect:** triggers transition to postpartum module; if `outcome == LIVE_BIRTH`, the client should subsequently call `POST /api/postpartum/baby/profile`.

---

### `GET /api/pregnancy/week-info`

**Query params:** none (derives current week from active pregnancy)

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "weekNumber": 31,
    "trimester": 3,
    "babySizeComparison": "About the size of a coconut",
    "developmentNote": "Your baby's bones are hardening, though the skull stays soft for delivery.",
    "imageUrl": "https://cdn.bintic.care/pregnancy/week31.png"
  }
}
```

---

### `GET /api/pregnancy/vitals/form-template`

Returns the active `FormTemplate` for this patient's pregnancy vitals — either her facility's custom template if one has been configured, or the platform default. The mobile client renders its form dynamically from this response rather than hardcoding fields.

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormTemplate entity, context: PREGNANCY_VITALS" } }`

---

### `POST /api/pregnancy/vitals`

**Request body:**

```json
{
  "templateId": "tmpl_preg_vitals_v2",
  "answers": {
    "bloodPressureSystolic": 122,
    "bloodPressureDiastolic": 80,
    "weightKg": 68.4,
    "temperatureCelsius": 36.7,
    "symptoms": ["REDUCED_FETAL_MOVEMENT"],
    "fetalMovementCount": 4,
    "notes": "Haven't felt baby move as much today"
  },
  "clientGeneratedId": "770e8400-e29b-41d4-a716-446655440002",
  "clientCreatedAt": "2026-06-29T08:10:00Z"
}
```

`answers` is a free-form object whose keys must match the `key` values defined in the `templateId`'s `fields` array (see the `FormTemplate` entity). The server validates `answers` against the template's field definitions (required fields present, values match expected `type`) and computes `isFlagged` from the template's `flaggingOptions`, rather than the client deciding what counts as a symptom worth flagging.

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormSubmission entity, context: PREGNANCY_VITALS" } }`

**Errors:** `400 TEMPLATE_VALIDATION_ERROR` — returned with `error.fields` listing which answer keys failed validation against the template.

---

### `GET /api/pregnancy/vitals`

**Query params:** `page`, `pageSize`, `flaggedOnly` (boolean)

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity" } ] }`

---

### `GET /api/pregnancy/vitals/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormSubmission entity" } }`

---

### `PUT /api/pregnancy/vitals/{id}`

**Request body:** `{ "answers": { ... } }` — any subset of the template's fields.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated FormSubmission entity" } }`

---

### `GET /api/pregnancy/patients/{patientId}/vitals`

**Role required:** `CLINICIAN` or `FACILITY_ADMIN`. Included here because the mobile app's "Clinician feedback thread" reads the clinician's responses generated against this endpoint.

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity" } ] }`

---

### `POST /api/pregnancy/vitals/{id}/feedback`

**Role required:** `CLINICIAN`.

**Request body:** `{ "message": "Mild ankle swelling is common at this stage, but let's keep an eye on it." }`

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "id": "fb_4471",
    "vitalsEntryId": "vit_7734",
    "clinicianId": "usr_doc_4471",
    "message": "Mild ankle swelling is common at this stage, but let's keep an eye on it.",
    "createdAt": "2026-06-24T10:00:00Z"
  }
}
```

---

### `GET /api/pregnancy/vitals/{id}/feedback`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Feedback entity" } ] }`

---

### `GET /api/pregnancy/anc-visits`

Returns this patient's ANC visit schedule. These `ScheduledVisit` records are **not created manually by the mobile client** — the full 8-visit schedule is automatically instantiated from the `path_anc_moh_v1` `CarePathwayTemplate` the moment `POST /api/pregnancy/start` is called, with each visit's `scheduledAt` calculated from the template's `triggerWeek` values against her due date. A facility may add an extra ad-hoc visit outside the standard schedule using `POST /api/pregnancy/anc-visits/manual` (see below).

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "id": "visit_anc_6",
      "pathwayTemplateId": "path_anc_moh_v1",
      "milestoneOrder": 6,
      "label": "6th ANC visit",
      "scheduledAt": "2026-06-26T10:30:00Z",
      "status": "COMPLETED",
      "facilityId": "fac_1029",
      "summary": "Weight 64kg, BP 118/76, no concerns noted"
    }
  ]
}
```

`status`: `SCHEDULED` | `COMPLETED` | `MISSED` | `RESCHEDULED`.

---

### `POST /api/pregnancy/anc-visits/manual`

Adds an ad-hoc visit outside the standard MOH schedule (e.g. an unscheduled follow-up). **Role required:** `CLINICIAN` or `FACILITY_ADMIN`.

**Request body:**

```json
{
  "scheduledAt": "2026-07-06T09:00:00Z",
  "facilityId": "fac_1029",
  "purpose": "Follow-up after reduced fetal movement report"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "ScheduledVisit entity, pathwayTemplateId: null" } }`

---

### `PUT /api/pregnancy/anc-visits/{id}`

**Request body:** `{ "status": "COMPLETED", "summary": "..." }` or `{ "scheduledAt": "2026-07-10T09:00:00Z" }` (reschedule)

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated ScheduledVisit entity" } }`

---

### `GET /api/pregnancy/nutrition-guidance`

**Query params:** `category` (optional: `IRON` | `FOLIC_ACID` | `HYDRATION` | `FOODS_TO_AVOID` | `HEALTHY_SNACKS`)

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "id": "nut_201",
      "category": "IRON",
      "title": "Why iron-rich foods matter this trimester",
      "summary": "Iron needs increase significantly...",
      "trimesterRelevance": [2, 3],
      "iconUrl": "https://cdn.bintic.care/nutrition/iron.png"
    }
  ]
}
```

---

### `GET /api/pregnancy/risk-score`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "score": 42,
    "level": "MEDIUM",
    "calculatedAt": "2026-06-29T08:30:00Z",
    "clinicianOverride": null,
    "factors": [
      {
        "label": "Reduced fetal movement reported",
        "weight": 25,
        "severity": "WARNING",
        "description": "Patient reported feeling the baby move significantly less than usual"
      },
      {
        "label": "Blood pressure within normal range",
        "weight": 0,
        "severity": "SUCCESS",
        "description": "Consistent readings across the last 4 entries"
      }
    ]
  }
}
```

`clinicianOverride` (if set by a clinician via the web app):

```json
"clinicianOverride": {
  "level": "LOW",
  "reason": "Reduced fetal movement resolved after follow-up call",
  "overriddenBy": "usr_doc_4471",
  "overriddenAt": "2026-06-29T09:00:00Z"
}
```

---

### `GET /api/pregnancy/risk-score/history`

**Response `200 OK`:** `{ "success": true, "data": [ { "calculatedAt": "...", "score": 38, "level": "LOW" } ] }`

---

## 5. Labour & Birth Monitor

These endpoints are primarily written to by clinicians on the web app, but mobile reads from them to show the mother a simplified, read-only labour status view (per the earlier decision that detailed labour monitoring lives on web).

### `POST /api/labour/sessions`

**Role required:** `CLINICIAN`.

**Request body:**

```json
{
  "pregnancyId": "preg_2207",
  "facilityId": "fac_1029",
  "activeLabourStartedAt": "2026-06-29T05:30:00Z"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "LabourSession entity" } }`

---

### `GET /api/labour/sessions/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "LabourSession entity" } }`

---

### `PUT /api/labour/sessions/{id}/close`

**Role required:** `CLINICIAN`.

**Request body:**

```json
{
  "closedAt": "2026-06-29T13:10:00Z",
  "outcome": "LIVE_BIRTH",
  "deliveryType": "VAGINAL"
}
```

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated LabourSession entity, status: CLOSED" } }`

---

### `POST /api/labour/sessions/{id}/readings/dilation`

**Role required:** `CLINICIAN`.

**Request body:** `{ "value": 6, "recordedAt": "2026-06-29T09:48:00Z" }`

**Response `201 Created`:** `{ "success": true, "data": { "...": "LabourReading entity, type: DILATION" } }`

---

### `POST /api/labour/sessions/{id}/readings/fhr`

**Request body:** `{ "value": 152, "recordedAt": "2026-06-29T09:48:00Z" }`

**Response `201 Created`:** `{ "success": true, "data": { "...": "LabourReading entity, type: FHR" } }`

---

### `POST /api/labour/sessions/{id}/readings/maternal-vitals`

**Request body:** `{ "bloodPressureSystolic": 124, "bloodPressureDiastolic": 82, "recordedAt": "2026-06-29T09:48:00Z" }`

**Response `201 Created`:** `{ "success": true, "data": { "...": "LabourReading entity, type: MATERNAL_BP" } }`

---

### `POST /api/labour/sessions/{id}/readings/contractions`

**Request body:** `{ "frequencyPer10Min": 4, "durationSeconds": 50, "recordedAt": "2026-06-29T09:48:00Z" }`

**Response `201 Created`:** `{ "success": true, "data": { "...": "LabourReading entity, type: CONTRACTIONS" } }`

---

### `GET /api/labour/sessions/{id}/partograph`

Returns all readings pre-formatted for charting, including the computed WHO alert/action line reference points.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "dilationReadings": [
      { "hoursElapsed": 0, "value": 4, "recordedAt": "2026-06-29T05:30:00Z" },
      { "hoursElapsed": 3.7, "value": 6, "recordedAt": "2026-06-29T09:48:00Z" }
    ],
    "fhrReadings": [
      { "hoursElapsed": 0, "value": 145 },
      { "hoursElapsed": 3.7, "value": 152 }
    ],
    "alertLine": { "startHour": 0, "startCm": 4, "slopeCmPerHour": 1 },
    "actionLine": { "startHour": 4, "startCm": 4, "slopeCmPerHour": 1 },
    "hasActionLineCrossed": true
  }
}
```

---

### `GET /api/labour/sessions/{id}/alerts`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "id": "alrt_8821",
      "type": "ACTION_LINE_CROSSED",
      "severity": "CRITICAL",
      "message": "Labour progress is now 4 hours behind the expected rate",
      "acknowledgedAt": null,
      "createdAt": "2026-06-29T09:50:00Z"
    }
  ]
}
```

`type`: `ACTION_LINE_CROSSED` | `FETAL_DISTRESS` | `PPH_RISK` | `PREECLAMPSIA_RISK` | `SEPSIS_RISK`. `severity`: `CRITICAL` | `WARNING`.

---

### `POST /api/labour/sessions/{id}/alerts/{alertId}/acknowledge`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Alert entity, acknowledgedAt set" } }`

---

### `POST /api/labour/sessions/{id}/alerts/{alertId}/escalate`

**Request body:** `{ "escalateTo": "REFERRAL" }`

**Response `200 OK`:** `{ "success": true, "data": { "referralId": "ref_4471" } }` *(creates a Referral, see Module 7)*

---

### `GET /api/labour/resuscitation-protocol`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "steps": [
      { "order": 1, "title": "Dry, warm, and position the baby", "timerSeconds": null },
      { "order": 2, "title": "Assess breathing and heart rate", "timerSeconds": null },
      { "order": 3, "title": "Begin positive pressure ventilation", "timerSeconds": 30, "instructions": "Ventilate at 40-60 breaths per minute. Reassess heart rate after 30 seconds." },
      { "order": 4, "title": "Reassess heart rate", "timerSeconds": null },
      { "order": 5, "title": "Chest compressions, if indicated", "timerSeconds": 60, "instructions": "3 compressions to 1 ventilation, at a rate of 120 events per minute." },
      { "order": 6, "title": "Consider escalation", "timerSeconds": null }
    ]
  }
}
```

---

### `POST /api/labour/sessions/{id}/resuscitation-log`

**Request body:**

```json
{
  "stepOrder": 3,
  "completedAt": "2026-06-17T09:43:15Z",
  "vitalsAtStep": { "heartRateBpm": 92, "spo2Percent": 88, "respiratoryEffort": "GASPING" }
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "ResuscitationLogEntry entity" } }`

---

## 6. Postpartum & Baby Tracker

### `GET /api/postpartum/maternal-checkins/form-template`

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormTemplate entity, context: MATERNAL_CHECKIN" } }`

---

### `POST /api/postpartum/maternal-checkins`

**Request body:**

```json
{
  "templateId": "tmpl_maternal_checkin_v1",
  "answers": {
    "physicalSymptoms": ["BLEEDING_REDUCED", "BREAST_TENDERNESS"],
    "mood": "NEUTRAL",
    "notes": ""
  },
  "clientGeneratedId": "880e8400-e29b-41d4-a716-446655440003",
  "clientCreatedAt": "2026-06-29T08:00:00Z"
}
```

Default template fields: `physicalSymptoms` (`MULTI_SELECT`: `BLEEDING_REDUCED` | `HEAVY_BLEEDING` | `PAIN_AT_SITE` | `SWELLING` | `FEVER_OR_CHILLS` | `BREAST_TENDERNESS` | `FEELING_WELL`, with `flaggingOptions: ["HEAVY_BLEEDING", "FEVER_OR_CHILLS"]`), `mood` (`SINGLE_SELECT`, same enum as cycle symptoms), `notes` (`TEXT`).

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormSubmission entity, context: MATERNAL_CHECKIN" } }`

---

### `GET /api/postpartum/maternal-checkins`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity" } ] }`

---

### `GET /api/postpartum/maternal-checkins/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormSubmission entity" } }`

---

### `POST /api/postpartum/depression-screening`

**Request body:**

```json
{
  "responses": [
    { "questionId": "q1", "answerValue": 0 },
    { "questionId": "q2", "answerValue": 2 },
    { "questionId": "q10", "answerValue": 0 }
  ]
}
```

Each `answerValue` is 0–3, following the standard EPDS-style scoring (lower = better). `q10` (self-harm ideation question) is checked specially — any value other than 0 triggers an immediate flag, independent of total score.

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "id": "epds_3301",
    "totalScore": 6,
    "suggestsSupportBeneficial": false,
    "immediateConcernFlag": false,
    "completedAt": "2026-06-29T08:05:00Z"
  }
}
```

A `totalScore` ≥ 13 (clinical threshold) sets `suggestsSupportBeneficial: true`. `immediateConcernFlag: true` if `q10 > 0`, regardless of total score, and triggers an immediate clinician notification.

---

### `GET /api/postpartum/depression-screening/history`

**Response `200 OK`:** `{ "success": true, "data": [ { "completedAt": "...", "totalScore": 6 } ] }`

---

### `GET /api/postpartum/depression-screening/flag`

**Response `200 OK`:** `{ "success": true, "data": { "isActive": false } }`

---

### `POST /api/postpartum/baby/profile`

**Request body:**

```json
{
  "name": "Amani",
  "dateOfBirth": "2026-06-17",
  "timeOfBirth": "09:42",
  "sex": "FEMALE",
  "birthWeightKg": 3.2,
  "birthLengthCm": 49,
  "deliveryType": "VAGINAL",
  "placeOfBirth": "fac_1029"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "BabyProfile entity" } }`

---

### `GET /api/postpartum/baby/profile`

**Response `200 OK`:** `{ "success": true, "data": { "...": "BabyProfile entity" } }`

---

### `PUT /api/postpartum/baby/profile`

**Request body:** any subset of creation fields.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated BabyProfile entity" } }`

---

### `GET /api/postpartum/baby/vitals/form-template`

**Response `200 OK`:** `{ "success": true, "data": { "...": "FormTemplate entity, context: BABY_VITALS" } }`

---

### `POST /api/postpartum/baby/vitals`

**Request body:**

```json
{
  "templateId": "tmpl_baby_vitals_v1",
  "answers": {
    "temperatureCelsius": 36.8,
    "spo2Percent": 98,
    "feedingType": "BREASTFEEDING",
    "feedCountToday": 6,
    "symptoms": ["ALL_NORMAL"],
    "notes": ""
  },
  "clientGeneratedId": "990e8400-e29b-41d4-a716-446655440004"
}
```

Default template fields: `temperatureCelsius` (`NUMBER`), `spo2Percent` (`NUMBER`), `feedingType` (`SINGLE_SELECT`: `BREASTFEEDING` | `FORMULA` | `BOTH`), `feedCountToday` (`NUMBER`), `symptoms` (`MULTI_SELECT`: `DIFFICULTY_BREATHING` | `UNUSUALLY_SLEEPY` | `POOR_FEEDING` | `JAUNDICE` | `FEVER` | `COLD_TO_TOUCH` | `ALL_NORMAL`, with `flaggingOptions` covering all except `ALL_NORMAL`), `notes` (`TEXT`).

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormSubmission entity, context: BABY_VITALS" } }`

---

### `GET /api/postpartum/baby/vitals`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "FormSubmission entity" } ] }`

---

### `GET /api/postpartum/baby/vitals/alerts`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    { "id": "balrt_201", "type": "LOW_TEMPERATURE", "message": "Temperature has been slightly low for 2 readings", "createdAt": "2026-06-29T05:00:00Z" }
  ]
}
```

---

### `POST /api/postpartum/baby/milestones`

**Request body:**

```json
{
  "category": "MOVEMENT",
  "title": "Lifted head during tummy time",
  "achievedAt": "2026-07-15",
  "note": "",
  "photoUrl": null
}
```

`category`: `GROWTH` | `MOVEMENT` | `FEEDING` | `SLEEP` | `FIRST_MOMENTS`.

**Response `201 Created`:** `{ "success": true, "data": { "...": "Milestone entity" } }`

---

### `GET /api/postpartum/baby/milestones`

**Query params:** `category` (optional filter)

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Milestone entity" } ] }`

---

### `POST /api/postpartum/baby/vaccinations`

Used by clinicians to mark a scheduled vaccination as given; mobile reads the resulting schedule.

**Request body:** `{ "vaccineId": "vac_bcg", "givenAt": "2026-06-17T10:00:00Z", "facilityId": "fac_1029", "batchNumber": "BCG-2026-0091" }`

**Response `201 Created`:** `{ "success": true, "data": { "...": "VaccinationRecord entity" } }`

---

### `GET /api/postpartum/baby/vaccinations/schedule`

This schedule is **not built manually** — the full infant immunization schedule (BCG, OPV, DPT-HepB-Hib, Pneumococcal, Rotavirus, Measles-Rubella, etc.) is automatically instantiated as `ScheduledVisit` records from a dedicated `path_vaccination_moh_v1` `CarePathwayTemplate` (`appliesToStage: NEWBORN`) the moment `POST /api/postpartum/baby/profile` is called, using the baby's date of birth against each milestone's `triggerWeek`.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    { "id": "visit_vac_bcg", "vaccineId": "vac_bcg", "name": "BCG", "ageMilestone": "At birth", "status": "GIVEN", "givenAt": "2026-06-17T10:00:00Z" },
    { "id": "visit_vac_dpt2", "vaccineId": "vac_dpt2", "name": "DPT-HepB-Hib (2nd dose)", "ageMilestone": "10 weeks", "status": "UPCOMING", "scheduledAt": "2026-08-26T00:00:00Z" }
  ]
}
```

`status`: `GIVEN` | `UPCOMING` | `OVERDUE` (maps to the underlying `ScheduledVisit.status` values `COMPLETED` | `SCHEDULED` | `MISSED`).

---

### `PUT /api/postpartum/baby/vaccinations/{id}/mark-given`

**Request body:** `{ "givenAt": "2026-08-26T09:00:00Z", "facilityId": "fac_1029", "batchNumber": "DPT-2026-0042" }`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated ScheduledVisit entity, status: COMPLETED" } }`

---

### `GET /api/postpartum/clinic-visits/schedule`

Combined mother + baby postnatal visit schedule (48 hours, 1 week, 6 weeks per WHO guidance), instantiated as `ScheduledVisit` records from `path_postnatal_moh_v1` (`appliesToStage: POSTPARTUM`) the moment a `PregnancyRecord` is ended with `outcome: LIVE_BIRTH`.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    { "id": "visit_pn_1", "label": "48-hour check", "scheduledAt": "2026-06-19T09:00:00Z", "covers": ["MOTHER", "BABY"], "status": "COMPLETED" },
    { "id": "visit_pn_2", "label": "6-week check", "scheduledAt": "2026-07-29T09:00:00Z", "covers": ["MOTHER", "BABY"], "status": "SCHEDULED" }
  ]
}
```

---

## 7. Universal Referral Network & Facilities

### `POST /api/facilities`

**Role required:** `FACILITY_ADMIN` (facility registration).

**Request body:**

```json
{
  "name": "Kilifi County Hospital",
  "type": "PUBLIC",
  "county": "Kilifi",
  "address": "Off Mombasa-Malindi Road, Kilifi Town",
  "phoneNumber": "+254712000111",
  "email": "info@kilificountyhospital.go.ke",
  "latitude": -3.6309,
  "longitude": 39.8499,
  "servicesOffered": ["ANTENATAL_CARE", "DELIVERY", "NEONATAL_ICU"]
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "Facility entity" } }`

---

### `GET /api/facilities`

**Query params:** `county`, `type`, `page`, `pageSize`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Facility entity" } ], "meta": { "...": "pagination" } }`

---

### `GET /api/facilities/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Facility entity" } }`

---

### `PUT /api/facilities/{id}`

**Role required:** `FACILITY_ADMIN`.

**Request body:** any subset of facility fields.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated Facility entity" } }`

---

### `PUT /api/facilities/{id}/availability`

**Role required:** `FACILITY_ADMIN`.

**Request body:**

```json
{
  "bloodBankStocked": true,
  "maternityBedsAvailable": true,
  "theatreAvailable": false,
  "neonatalIcuCapacity": true,
  "ambulanceOnStandby": true
}
```

**Response `200 OK`:** `{ "success": true, "data": { "...": "Facility entity, readiness updated" } }`

This is the endpoint the mobile referral map and facility detail screens read from in real time.

---

### `POST /api/facilities/{id}/staff`

**Role required:** `FACILITY_ADMIN`.

**Request body:** `{ "userId": "usr_doc_4471", "role": "CLINICIAN", "specialty": "Obstetrics" }`

**Response `201 Created`:** `{ "success": true, "data": { "...": "StaffMember entity" } }`

---

### `GET /api/facilities/{id}/staff`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "StaffMember entity" } ] }`

---

### `DELETE /api/facilities/{id}/staff/{userId}`

**Response `204 No Content`**

---

### `GET /api/facilities/nearby`

**Query params:** `latitude` (required), `longitude` (required), `transportMode` (optional: `WALKING` | `MATATU` | `BODA` | `CAR`), `type` (optional filter), `radiusKm` (default 25)

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    {
      "facility": { "...": "Facility entity" },
      "distanceKm": 4.2,
      "estimatedTimeToCareMinutes": 12,
      "transportModeUsed": "CAR"
    }
  ]
}
```

Results are sorted by `estimatedTimeToCareMinutes` ascending, not raw distance — this is the field the mobile UI surfaces prominently per the "time to nearest care" design decision.

---

### `POST /api/referrals`

Creates a referral or emergency request.

**Request body:**

```json
{
  "toFacilityId": "fac_1029",
  "fromFacilityId": "fac_2002",
  "reason": "REDUCED_FETAL_MOVEMENT",
  "notes": "Patient reports significantly reduced movement since this morning",
  "isEmergency": true,
  "offlineQueued": false
}
```

`reason`: `HEAVY_BLEEDING` | `SEVERE_PAIN` | `REDUCED_FETAL_MOVEMENT` | `LABOUR_STARTED` | `SOMETHING_FEELS_WRONG` | `ROUTINE_TRANSFER` | `SPECIALIST_REFERRAL`. If `offlineQueued: true`, the client is signaling this request was created while offline and is being sent now that connectivity returned; the server uses `clientCreatedAt` (required in this case) as the true request time.

**Response `201 Created`:**

```json
{
  "success": true,
  "data": { "...": "Referral entity" }
}
```

**Side effect:** if the mother has an emergency contact configured, an SMS notification is automatically sent to them (see Module 8).

---

### `GET /api/referrals/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Referral entity" } }`

---

### `GET /api/referrals`

**Query params:** `status`, `facilityId`, `direction` (`INCOMING` | `OUTGOING`, relative to the authenticated facility), `page`, `pageSize`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Referral entity" } ] }`

---

### `PUT /api/referrals/{id}/accept`

**Role required:** `CLINICIAN` or `FACILITY_ADMIN` at the receiving facility.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Referral entity, status: ACCEPTED" } }`

This triggers the mobile status tracker to advance from "Facility reviewing your request" to "Accepted — they're preparing for your arrival," and consent-gated data sharing executes (see `GET /api/referrals/{id}/patient-summary` below).

---

### `PUT /api/referrals/{id}/reject`

**Request body:** `{ "reason": "No theatre capacity available" }`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Referral entity, status: REJECTED" } }`

---

### `PUT /api/referrals/{id}/complete`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Referral entity, status: COMPLETED, completedAt set" } }`

---

### `GET /api/referrals/{id}/patient-summary`

Returns the clinician-readable "emergency brief" — a condensed summary, not the full record. Requires either active consent or the mother's `emergencySharingPreference` to be `AUTO_SHARE`.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "patient": { "fullName": "Wanjiru Kamau", "age": 24, "bloodType": "O+" },
    "gestationalAgeWeeks": 31,
    "activeRiskFlags": ["REDUCED_FETAL_MOVEMENT"],
    "reasonForVisit": "REDUCED_FETAL_MOVEMENT",
    "recentVitals": { "bloodPressure": "122/80", "lastRecordedAt": "2026-06-29T08:10:00Z" },
    "allergies": [],
    "emergencyContact": { "name": "James Kamau", "phoneNumber": "+254721556002" }
  }
}
```

**Response `200 OK`** (consent not yet granted, preference is `ASK_FIRST`):

```json
{
  "success": true,
  "data": { "consentGranted": false, "consentRequestSent": true }
}
```

---

### `GET /api/referrals/{id}/summary-document`

**Response `200 OK`:** `{ "success": true, "data": { "documentUrl": "https://cdn.bintic.care/referrals/ref_4471_summary.pdf" } }`

---

## 8. Reminders, Notifications & SMS

### `GET /api/reminders`

**Query params:** `upcomingOnly` (boolean), `type` (filter)

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Reminder entity" } ] }`

---

### `POST /api/reminders`

**Request body:**

```json
{
  "type": "ANC_VISIT",
  "title": "ANC visit — Week 34 check-up",
  "dueAt": "2026-07-06T09:00:00Z"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "Reminder entity" } }`

---

### `PUT /api/reminders/{id}`

**Request body:** `{ "dueAt": "2026-07-08T09:00:00Z" }`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated Reminder entity" } }`

---

### `DELETE /api/reminders/{id}`

**Response `204 No Content`**

---

### `PUT /api/reminders/{id}/mark-done`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Reminder entity, isDone: true" } }`

---

### `POST /api/devices/register`

Registers a push notification device token.

**Request body:** `{ "deviceToken": "fcm_abc123...", "platform": "ANDROID" }`

**Response `201 Created`:** `{ "success": true, "data": { "tokenId": "dev_4471" } }`

---

### `DELETE /api/devices/{tokenId}`

**Response `204 No Content`**

---

### `GET /api/notifications`

**Query params:** `page`, `pageSize`, `unreadOnly` (boolean)

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "Notification entity" } ] }`

---

### `PUT /api/notifications/{id}/read`

**Response `200 OK`:** `{ "success": true, "data": { "...": "Notification entity, isRead: true" } }`

---

### `POST /api/notifications/sms/send`

**Internal endpoint** — triggered server-side (e.g. on emergency referral creation), not called directly by the mobile client. Documented here for backend reference.

**Request body:** `{ "toPhoneNumber": "+254721556002", "templateId": "emergency_contact_notify", "variables": { "motherName": "Wanjiru Kamau", "facilityName": "Kilifi County Hospital" } }`

**Response `200 OK`:** `{ "success": true, "data": { "smsId": "sms_9911", "status": "SENT" } }`

---

### `POST /api/notifications/sms/inbound-webhook`

**Internal endpoint** — receives inbound SMS replies from the SMS gateway (e.g. Africa's Talking). Used for SMS-only patients (registered manually by a facility admin, with no smartphone) to respond to simple check-in prompts like "Reply 1 if baby moved today, 2 if not," since they have no app to log entries in directly. A reply is converted server-side into a `FormSubmission` against the relevant context, with `enteredBy` set to the patient herself (the reply came from her own phone number) even though a staff member's prompt triggered it.

**Request body** (gateway-defined shape, example):

```json
{
  "from": "+254712004552",
  "text": "1",
  "linkedReminderId": "rem_2281"
}
```

**Response `200 OK`:** `{ "success": true, "data": null }`

---

### `GET /api/notifications/sms/preferences`

**Response `200 OK`:** `{ "success": true, "data": { "contactPreference": "BOTH" } }`

`contactPreference`: `APP_NOTIFICATIONS` | `SMS` | `BOTH`.

---

### `PUT /api/notifications/sms/preferences`

**Request body:** `{ "contactPreference": "SMS" }`

**Response `200 OK`:** `{ "success": true, "data": { "contactPreference": "SMS" } }`

---

## 9. Education & Community Engagement

### `POST /api/education/content`

**Role required:** `CLINICIAN` or `FACILITY_ADMIN`.

**Request body:**

```json
{
  "title": "Why hydration matters more during pregnancy",
  "category": "HYDRATION",
  "body": "Full article content...",
  "trimesterRelevance": [2, 3],
  "facilityId": "fac_1029"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "EducationContent entity" } }`

---

### `GET /api/education/content`

**Query params:** `category`, `page`, `pageSize`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "EducationContent entity" } ] }`

---

### `GET /api/education/content/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "EducationContent entity" } }`

---

### `PUT /api/education/content/{id}`

**Role required:** `CLINICIAN` or `FACILITY_ADMIN`.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated EducationContent entity" } }`

---

### `DELETE /api/education/content/{id}`

**Response `204 No Content`**

---

### `POST /api/education/events`

**Request body:**

```json
{
  "title": "Free antenatal screening day",
  "facilityId": "fac_1029",
  "eventDate": "2026-07-05T09:00:00Z",
  "description": "Free BP and weight screening for all registered mothers"
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "EducationEvent entity" } }`

---

### `GET /api/education/events`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "EducationEvent entity" } ] }`

---

### `GET /api/education/events/{id}`

**Response `200 OK`:** `{ "success": true, "data": { "...": "EducationEvent entity" } }`

---

### `GET /api/education/feed`

Personalized feed for the logged-in mother, mixing content and events relevant to her current stage.

**Response `200 OK`:** `{ "success": true, "data": [ { "type": "CONTENT", "...": "EducationContent entity" }, { "type": "EVENT", "...": "EducationEvent entity" } ] }`

---

## 10. AI Assistant & Personal Doctor Chat

### `GET /api/assistant/conversations`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": [
    { "id": "conv_201", "preview": "Asked about nausea remedies", "flagged": false, "createdAt": "2026-06-28T19:00:00Z" }
  ]
}
```

---

### `GET /api/assistant/conversations/{id}`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "id": "conv_201",
    "messages": [
      { "role": "USER", "text": "I'm feeling nauseous in the mornings", "createdAt": "2026-06-28T19:00:00Z" },
      { "role": "ASSISTANT", "text": "That's common in early pregnancy...", "createdAt": "2026-06-28T19:00:05Z" }
    ]
  }
}
```

---

### `POST /api/assistant/conversations/{id}/messages`

**Request body:** `{ "text": "Is this normal at 12 weeks?" }`

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "userMessage": { "role": "USER", "text": "Is this normal at 12 weeks?", "createdAt": "2026-06-29T08:00:00Z" },
    "assistantMessage": { "role": "ASSISTANT", "text": "Yes, this is very common around 12 weeks...", "createdAt": "2026-06-29T08:00:04Z" }
  }
}
```

The assistant's response is generated using the user's current profile, pregnancy/cycle/postpartum stage, and recent flagged entries as context, per the `accessHealthRecords` setting below.

---

### `GET /api/assistant/settings`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "voiceResponsesEnabled": true,
    "voiceStyle": "WARM_AND_CALM",
    "language": "en",
    "conversationStyle": "DETAILED",
    "accessHealthRecords": true
  }
}
```

`voiceStyle`: `WARM_AND_CALM` | `CLEAR_AND_CONFIDENT`. `conversationStyle`: `DETAILED` | `QUICK`.

---

### `PUT /api/assistant/settings`

**Request body:** any subset of the above fields.

**Response `200 OK`:** `{ "success": true, "data": { "...": "Updated settings" } }`

---

### `DELETE /api/assistant/conversations`

Clears all conversation history.

**Response `204 No Content`**

---

### `GET /api/doctor-chat/threads/{doctorId}`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "doctorId": "usr_doc_4471",
    "messages": [
      { "sender": "DOCTOR", "text": "How are you feeling today?", "createdAt": "2026-06-29T07:00:00Z" },
      { "sender": "MOTHER", "text": "A bit tired but okay", "createdAt": "2026-06-29T07:05:00Z" }
    ]
  }
}
```

---

### `POST /api/doctor-chat/threads/{doctorId}/messages`

**Request body:** `{ "text": "Should I be worried about the swelling?", "attachedEntryId": "sub_7734" }`

`attachedEntryId` is optional — lets the mother attach a specific vitals/cycle entry to the message.

**Response `201 Created`:** `{ "success": true, "data": { "...": "DoctorChatMessage entity" } }`

---

## 11. Facility-Managed Patients & Manual Entry

These endpoints are called from the **web app by a `FACILITY_ADMIN` or `CLINICIAN`**, not from mobile. They are documented here because they directly create or modify the same `User`, `Profile`, and `FormSubmission` records that mobile reads and writes, and the mobile data model assumes their existence (e.g. an SMS-only patient with no app session, or a doctor assignment that originated from a facility-side decision rather than a mobile action).

### `POST /api/facility-admin/patients`

Manually registers a patient who lacks a smartphone, or onboards an existing walk-in patient directly from the facility side. **Role required:** `FACILITY_ADMIN`.

**Request body:**

```json
{
  "fullName": "Esther Nyambura",
  "phoneNumber": "+254712004552",
  "dateOfBirth": "2007-09-02",
  "gender": "FEMALE",
  "accountType": "SMS_ONLY",
  "currentStage": "PREGNANT",
  "facilityId": "fac_1029"
}
```

`accountType`: `APP` (she will log into the mobile app herself) | `SMS_ONLY` (no app session; all reminders and check-in prompts are delivered by SMS, and her replies are converted to `FormSubmission` records via the inbound SMS webhook).

**Response `201 Created`:** `{ "success": true, "data": { "user": { "...": "User entity" }, "profile": { "...": "Profile entity" } } }`

---

### `GET /api/facility-admin/patients`

**Query params:** `facilityId`, `assignedClinicianId` (filter, optional, use `"unassigned"` to find patients with no clinician), `accountType` (filter), `page`, `pageSize`

**Response `200 OK`:** `{ "success": true, "data": [ { "...": "User + Profile summary" } ], "meta": { "...": "pagination" } }`

---

### `PUT /api/facility-admin/patients/{patientId}/assign-clinician`

General-purpose patient-to-clinician routing, used both for ordinary caseload assignment and for fulfilling a personal-doctor request.

**Request body:** `{ "clinicianId": "usr_doc_4471", "reason": "PERSONAL_DOCTOR_REQUEST" }`

`reason`: `GENERAL_ASSIGNMENT` | `PERSONAL_DOCTOR_REQUEST`. When `reason` is `PERSONAL_DOCTOR_REQUEST`, the server also updates the patient's `Profile.personalDoctorId` and `personalDoctorRequestStatus` (set to `ASSIGNED`), which mobile picks up via `GET /api/profile/personal-doctor`.

**Response `200 OK`:** `{ "success": true, "data": { "patientId": "usr_8f3a2b1c", "assignedClinicianId": "usr_doc_4471" } }`

---

### `POST /api/facility-admin/patients/{patientId}/entries/{context}`

Allows facility staff to submit a `FormSubmission` on behalf of a patient — for example, a clinician taking BP manually during a walk-in visit for a patient with no smartphone, or backfilling a missed entry. `context` is one of the `FormTemplate` context values (e.g. `PREGNANCY_VITALS`).

**Request body:**

```json
{
  "templateId": "tmpl_preg_vitals_v2",
  "answers": { "bloodPressureSystolic": 130, "bloodPressureDiastolic": 86, "weightKg": 65.0 }
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormSubmission entity, enteredBy: the staff member's user ID, subjectUserId: the patient's user ID" } }`

The distinction between `enteredBy` and `subjectUserId` is what lets the clinician dashboard and mobile timeline both correctly show "logged by your care team" versus "logged by you," exactly as previously handled for CHP-assisted entries, now generalized to any facility-staff-entered record.

---

### `POST /api/facility-admin/form-templates`

Creates or extends a `FormTemplate` for the facility's own patients — e.g. adding an extra question specific to a high-risk patient population. **Role required:** `FACILITY_ADMIN`.

**Request body:**

```json
{
  "context": "PREGNANCY_VITALS",
  "facilityId": "fac_1029",
  "basedOnTemplateId": "tmpl_preg_vitals_v2",
  "additionalFields": [
    { "key": "homeBloodSugarMmol", "label": "Home blood sugar reading", "type": "NUMBER", "unit": "mmol/L", "required": false }
  ]
}
```

**Response `201 Created`:** `{ "success": true, "data": { "...": "FormTemplate entity, facilityId: fac_1029, isDefault: false" } }`

From this point on, patients linked to `fac_1029` receive this extended template from `GET /api/pregnancy/vitals/form-template` instead of the platform default — no mobile app update or backend redeploy required.

---

## Error Codes Reference

| HTTP Status | Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | One or more fields failed validation; see `error.fields` |
| 400 | `TEMPLATE_VALIDATION_ERROR` | `answers` object failed validation against the active `FormTemplate`; see `error.fields` |
| 400 | `NO_PREFERRED_FACILITY` | Action requires a facility relationship the mother hasn't established yet |
| 401 | `UNAUTHORIZED` | Missing or invalid access token |
| 401 | `INVALID_CREDENTIALS` | Login phone/password mismatch |
| 401 | `INVALID_REFRESH_TOKEN` | Refresh token invalid or revoked |
| 401 | `REFRESH_TOKEN_EXPIRED` | Refresh token past expiry |
| 403 | `FORBIDDEN` | Authenticated but lacks permission for this resource |
| 403 | `CONSENT_REQUIRED` | Action blocked pending the mother's consent |
| 404 | `NOT_FOUND` | Generic resource not found |
| 404 | `NO_ACTIVE_PREGNANCY` | No active `PregnancyRecord` exists for this user |
| 404 | `QR_TOKEN_NOT_FOUND` | QR passport token does not match any user |
| 409 | `PHONE_ALREADY_REGISTERED` | Registration attempted with an existing phone number |
| 410 | `QR_TOKEN_EXPIRED` | QR passport token has expired or been refreshed |
| 410 | `OTP_EXPIRED` | Password reset OTP expired |
| 423 | `ACCOUNT_LOCKED` | Too many failed login attempts |
| 429 | `RATE_LIMITED` | Too many requests in a short window |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

---

*End of mobile API documentation.*