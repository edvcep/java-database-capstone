# Clinic Management System – Database Schema Design

This document outlines the hybrid database design for a clinic management system.
- **MySQL** is used for structured, transaction-safe core operations (patient records, appointment scheduling, billing).
- **MongoDB** complements the system for unstructured, flexible, or deeply nested data (doctor notes, prescriptions with metadata, chat logs, audit trails).

---

## MySQL Database Design

The following relational tables form the backbone of clinic operations. All tables use the `InnoDB` storage engine to ensure referential integrity and transaction safety.

### Table: clinic_locations
- id: INT, Primary Key, Auto Increment
- name: VARCHAR(100), Not Null
- address: VARCHAR(255), Not Null
- city: VARCHAR(100), Not Null
- phone: VARCHAR(20), Not Null
- is_active: BOOLEAN, Not Null, Default TRUE

### Table: admin
- id: INT, Primary Key, Auto Increment
- username: VARCHAR(50), Not Null, Unique
- email: VARCHAR(100), Not Null, Unique
- password_hash: VARCHAR(255), Not Null
- role: VARCHAR(30), Not Null (e.g., 'SUPER_ADMIN', 'RECEPTIONIST')
- created_at: TIMESTAMP, Default CURRENT_TIMESTAMP

### Table: patients
- id: INT, Primary Key, Auto Increment
- first_name: VARCHAR(50), Not Null
- last_name: VARCHAR(50), Not Null
- date_of_birth: DATE, Not Null
- email: VARCHAR(100), Not Null, Unique
- phone: VARCHAR(20), Not Null
- insurance_number: VARCHAR(50), Unique, Nullable
- is_active: BOOLEAN, Not Null, Default TRUE
- created_at: TIMESTAMP, Default CURRENT_TIMESTAMP

### Table: doctors
- id: INT, Primary Key, Auto Increment
- location_id: INT, Foreign Key → clinic_locations(id)
- first_name: VARCHAR(50), Not Null
- last_name: VARCHAR(50), Not Null
- specialization: VARCHAR(100), Not Null
- license_number: VARCHAR(50), Not Null, Unique
- email: VARCHAR(100), Not Null, Unique
- phone: VARCHAR(20), Not Null
- is_active: BOOLEAN, Not Null, Default TRUE

### Table: doctor_availabilities
- id: INT, Primary Key, Auto Increment
- doctor_id: INT, Foreign Key → doctors(id), On Delete Cascade
- day_of_week: TINYINT, Not Null (1 = Monday to 7 = Sunday)
- start_time: TIME, Not Null
- end_time: TIME, Not Null
- slot_duration_minutes: INT, Not Null, Default 30

### Table: appointments
- id: INT, Primary Key, Auto Increment
- doctor_id: INT, Foreign Key → doctors(id), On Delete Restrict
- patient_id: INT, Foreign Key → patients(id), On Delete Restrict
- location_id: INT, Foreign Key → clinic_locations(id)
- appointment_time: DATETIME, Not Null
- duration_minutes: INT, Not Null, Default 30
- status: INT, Not Null, Default 0 (0 = Scheduled, 1 = Completed, 2 = Cancelled, 3 = No-Show)
- created_at: TIMESTAMP, Default CURRENT_TIMESTAMP
- *Constraint:* `UNIQUE (doctor_id, appointment_time)` (prevents exact double bookings at the same timestamp)

### Table: payments
- id: INT, Primary Key, Auto Increment
- appointment_id: INT, Foreign Key → appointments(id), On Delete Restrict
- patient_id: INT, Foreign Key → patients(id), On Delete Restrict
- amount: DECIMAL(10, 2), Not Null
- currency: VARCHAR(3), Not Null, Default 'CHF'
- payment_method: VARCHAR(30), Not Null (e.g., 'CARD', 'CASH', 'INSURANCE')
- status: INT, Not Null, Default 0 (0 = Pending, 1 = Paid, 2 = Refunded)
- paid_at: DATETIME, Nullable

---

### Architectural Decisions & Justifications (MySQL)

1. **Deletion Behavior (`ON DELETE RESTRICT` & Soft Deletes):**
   - When a patient or doctor leaves the system, historical `appointments` and `payments` must **not** be deleted due to legal and medical record-retention requirements. Instead of a physical `DELETE`, the `is_active = FALSE` flag (Soft Delete) is used.
2. **Preventing Overlapping Appointments:**
   - A `UNIQUE` constraint on `(doctor_id, appointment_time)` prevents duplicate bookings at the exact same start time. However, since appointments can vary in duration, backend application logic must also check for time-window overlaps before executing an `INSERT` (`new_start < existing_end AND new_end > existing_start`).
3. **Email & Phone Format Validation:**
   - Database constraints (`UNIQUE`, `NOT NULL`) enforce structural integrity at the schema level. Format validations (such as regex checks for email addresses or international phone prefixes) are handled in the application layer to provide clearer error messages and easier updates.
4. **Doctor Availability (`doctor_availabilities`):**
   - Each doctor manages their own weekly working hours, which the booking engine uses to dynamically calculate available time slots.

---

## MongoDB Collection Design

MongoDB is used for unstructured medical documentation, prescriptions with variable medication lists, and communication logs.

### Collection: prescriptions

Prescriptions are typically linked to a specific appointment (`appointmentId`), but can also exist independently by setting `appointmentId: null` (e.g., for phone-based refill requests).

```json
{
  "_id": "ObjectId('664abc1234567890abcdef01')",
  "prescriptionNumber": "RX-2026-88412",
  "patientId": 1042,
  "doctorId": 18,
  "appointmentId": 519,
  "issuedAt": "2026-11-22T10:30:00Z",
  "validUntil": "2027-02-22T23:59:59Z",
  "status": "ACTIVE",
  "medications": [
    {
      "name": "Paracetamol",
      "activeIngredient": "Acetaminophen",
      "dosage": "500mg",
      "frequency": "Every 6 hours as needed",
      "durationDays": 5,
      "instructions": "Take with plenty of water after a meal."
    },
    {
      "name": "Pantoprazole",
      "activeIngredient": "Pantoprazole",
      "dosage": "20mg",
      "frequency": "Once daily in the morning",
      "durationDays": 14,
      "instructions": "Take 30 minutes before breakfast."
    }
  ],
  "doctorNotes": "Patient reports mild postoperative pain. No known drug allergies.",
  "refillCount": 2,
  "pharmacy": {
    "name": "Bahnhof Pharmacy",
    "location": "Zurich Main Station",
    "electronicDispatch": true
  },
  "attachments": [
    {
      "fileName": "lab_results_bloodwork.pdf",
      "fileUrl": "s3://clinic-docs/2026/11/lab_results_bloodwork.pdf",
      "uploadedAt": "2026-11-22T10:15:00Z"
    }
  ],
  "tags": ["pain-management", "post-op", "refill"],
  "schemaVersion": 1
}
```

### Collection: consultation_notes_and_chat

Complementing prescriptions, this collection stores doctor-patient chat threads, file attachments, and post-visit patient feedback.

```json
{
  "_id": "ObjectId('664abc9876543210fedcba99')",
  "appointmentId": 519,
  "patientId": 1042,
  "doctorId": 18,
  "messages": [
    {
      "senderRole": "PATIENT",
      "senderId": 1042,
      "timestamp": "2026-11-22T08:12:00Z",
      "text": "Good morning, I have uploaded my latest bloodwork results.",
      "attachmentUrl": "s3://clinic-docs/2026/11/lab_results_bloodwork.pdf",
      "read": true
    },
    {
      "senderRole": "DOCTOR",
      "senderId": 18,
      "timestamp": "2026-11-22T08:45:00Z",
      "text": "Thank you, we will review the values during your 10:00 AM appointment.",
      "read": true
    }
  ],
  "patientFeedback": {
    "rating": 5,
    "comment": "Very short wait time and clear explanations.",
    "submittedAt": "2026-11-22T11:05:00Z"
  },
  "schemaVersion": 1
}
```

---

### Architectural Decisions & Justifications (MongoDB)

1. **Referencing via ID (`patientId`, `doctorId`) Instead of Full Embedding:**
   - Core personal details such as name, phone number, or email address can change over time and reside in MySQL as the *Single Source of Truth*. Embedding the entire patient object inside every MongoDB document would lead to stale or inconsistent data whenever a patient updates their profile. Storing only reference IDs avoids duplication.
2. **Schema Evolution (`schemaVersion`):**
   - Including a `schemaVersion` field allows the application to gracefully handle future structural changes (e.g., new required fields for e-prescriptions or telehealth metadata) without requiring immediate migrations of historical documents.
3. **Arrays and Nested Objects:**
   - A single prescription can contain multiple medications (`medications` array), and chat histories grow continuously (`messages` array) within a single document, eliminating the need for complex multi-table `JOIN` queries.
