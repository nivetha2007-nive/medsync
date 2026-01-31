# MedSync - AI-Powered Digital Pharmacy Platform

## Project Overview
MedSync is a secure e-commerce pharmacy platform with AI-powered prescription validation, medical triage, and blockchain-backed fraud prevention. Built with Next.js (frontend) and FastAPI (backend).

---

## Tech Stack

### Frontend
- **Framework**: Next.js 15 (App Router)
- **Language**: TypeScript
- **UI Library**: React 19
- **Styling**: Tailwind CSS + shadcn/ui components
- **State Management**: React Context API + Zustand
- **Forms**: React Hook Form + Zod validation
- **HTTP Client**: Axios
- **Real-time**: Socket.io (for notifications)
- **Video**: Agora SDK / WebRTC
- **Maps**: Google Maps JavaScript API
- **Voice**: Web Speech API (browser native)

### Backend
- **Framework**: FastAPI (Python 3.11+)
- **Database**: MongoDB (primary), MinIO (temporary file storage)
- **Authentication**: 
  - Firebase Phone Auth (Users)
  - JWT (Admin/Doctor/Pharmacist)
- **AI/ML**: 
  - Google Gemini 2.0 Flash API (OCR extraction)
  - Google Cloud Vision API (image analysis, tampering detection)
- **Blockchain**: Hyperledger Besu / Polygon Edge (optional for MVP, use MongoDB hashes)
- **Image Processing**: OpenCV, PIL, imagehash
- **Payment**: Razorpay (test mode)

---

## User Roles & Authentication

### 1. User (Patient)
- **Auth**: Firebase Phone OTP (one phone = one account)
- **Permissions**: Browse medicines, upload prescriptions, order, consult doctors, AI triage
- **Unique ID**: `USR{YYYYMMDD}{RANDOM8}` (e.g., USR20260131A1B2C3D4)

### 2. Admin
- **Auth**: JWT (email + password)
- **Permissions**: Manage medicines, view orders, fraud logs, user management, analytics
- **Login**: `/admin/login`

### 3. Pharmacist
- **Auth**: JWT (email + password)
- **Permissions**: Review low-confidence prescriptions (<75%), approve/reject, manual medicine entry
- **Login**: `/pharmacist/login`

### 4. Doctor
- **Auth**: JWT (email + password)
- **Permissions**: View consultations, video calls, issue digital prescriptions
- **Login**: `/doctor/login`

---

## Core Features

### Feature 1: Two Prescription Upload Pipelines

#### Pipeline 1: Homepage Upload (Extract All Medicines)
**User Flow:**
1. User clicks "Upload Prescription" on homepage
2. Uploads prescription image (JPG/PNG/PDF, max 10MB)
3. Backend runs **10-layer validation** (3-5 seconds)
4. Gemini AI extracts **ALL medicines** from prescription
5. **All extracted medicines auto-populate cart**
6. User confirms and proceeds to checkout

**Route**: `POST /api/prescriptions/upload` (no medicine_id in request)

#### Pipeline 2: Search → Upload (Match Specific Medicine)
**User Flow:**
1. User searches "Paracetamol" → clicks on medicine
2. Medicine detail page asks "Do you have a prescription?"
3. User uploads prescription
4. Backend runs **same 10-layer validation**
5. Gemini AI extracts **ALL medicines**
6. Backend matches **ONLY "Paracetamol"** from extracted list
7. **Only matched medicine added to cart** (not all medicines)
8. If not found in prescription → Show error "Paracetamol not found in prescription"

**Route**: `POST /api/prescriptions/upload?medicine_id=MED001`

**KEY**: Same validation engine, different post-processing logic.

---

### Feature 2: 10-Layer Prescription Validation Pipeline

**Execution Time**: 3-5 seconds total

#### Layer 1: File Format Validation (0.2s)
- Check file type: JPG, PNG, PDF only
- Validate MIME type
- Check file size < 10MB
- Detect corruption
- **Output**: `PASS` or `FAIL`

#### Layer 2: File Upload to MinIO (0.3s)
- Upload to temporary storage (MinIO bucket: `prescriptions-temp`)
- Generate presigned URL (expires in 1 hour)
- Store temp file path
- **Output**: `file_url`, `temp_file_id`

#### Layer 3: Duplicate Detection (Image Hash) (0.4s)
- Calculate perceptual hash using `imagehash` library (pHash algorithm)
- Search MongoDB `prescriptions` collection for matching `image_hash`
- If duplicate found:
  - Check if **same user** → Proceed to Layer 9 (refill eligibility check)
  - Check if **different user** → Flag as fraud, REJECT
- If no duplicate → Continue to Layer 4
- **Output**: `is_duplicate`, `original_prescription_id`, `original_user_id`

#### Layer 4: AI OCR Extraction (Gemini 2.0 Flash) (1.5s)
- Send image to Gemini API with prompt:
```
Extract from this prescription:
1. Doctor name
2. Hospital name
3. Patient name
4. Prescription date (YYYY-MM-DD)
5. List of medicines with:
   - Medicine name
   - Dosage (e.g., 500mg)
   - Frequency (e.g., twice daily)
   - Quantity (e.g., 30 tablets)
Return as JSON.
```
- Parse Gemini response
- **Output**: 
```json
{
  "doctor_name": "Dr. Ajay Sharma",
  "hospital_name": "Apollo Hospital",
  "patient_name": "Rahul Kumar",
  "prescription_date": "2026-01-25",
  "medicines": [
    {"name": "Paracetamol", "dosage": "500mg", "frequency": "twice daily", "quantity": 30},
    {"name": "Azithromycin", "dosage": "500mg", "frequency": "once daily", "quantity": 5}
  ],
  "confidence": 0.92
}
```

#### Layer 5: Image Quality Check (0.3s)
- Use OpenCV to analyze:
  - Resolution (min 1200x800 pixels)
  - Brightness (40-220 on 0-255 scale)
  - Contrast (min 40 on 0-100 scale)
  - Blur score (Laplacian variance > 100)
- Calculate quality score (0-1)
- If quality < 0.75 → Send to pharmacist manual review queue
- **Output**: `quality_score`, `needs_manual_review`

#### Layer 6: Tampering Detection (0.5s)
- Method 1: JPEG artifact analysis (detect Photoshop edits)
- Method 2: Error Level Analysis (ELA) - detect spliced regions
- Method 3: Pixel consistency check (abrupt color transitions)
- Calculate tampering score (0-1)
- If tampering > 0.5 → REJECT prescription
- **Output**: `tampering_score`, `tampering_detected`

#### Layer 7: Face Detection (Privacy) (0.2s)
- Use OpenCV Haar Cascade to detect human faces
- If face detected → REJECT (privacy violation)
- **Output**: `face_detected`

#### Layer 8: Content Hash Generation (0.1s)
- Combine: `doctor_name + patient_name + prescription_date + medicine_names`
- Hash using SHA-256
- Store as `content_hash` for logical duplicate detection
- **Output**: `content_hash`

#### Layer 9: Refill Eligibility Check (0.3s)
- Triggered if: Layer 3 found duplicate for same user
- Check MongoDB `prescriptions` collection:
  - `refill_count < 3` (max 3 refills allowed)
  - `days_since_last_order >= 5` (minimum 5-day gap)
  - `prescription_date + 180 days > today` (6-month validity)
- If eligible → Allow refill, increment `refill_count`
- If not eligible → REJECT with reason
- **Output**: `refill_eligible`, `refill_reason`

#### Layer 10: Medicine Database Matching (0.2s)
- For each extracted medicine, search MongoDB `medicines` collection
- Match by fuzzy name search (handle typos, e.g., "Paracetomol" → "Paracetamol")
- Check if medicine type is "RX" or "OTC"
- Determine prescription type:
  - All medicines are OTC → `COMPLETE_OTC`
  - All medicines are RX → `COMPLETE_RX`
  - Mixed → `MIXED`
- Check warehouse stock availability
- **Output**:
```json
{
  "matched_medicines": [
    {"medicine_id": "MED001", "name": "Paracetamol", "type": "OTC", "stock": 500},
    {"medicine_id": "MED003", "name": "Azithromycin", "type": "RX", "stock": 200}
  ],
  "prescription_type": "MIXED",
  "unmatched_medicines": []
}
```

#### Final Validation Result:
```json
{
  "status": "VALID" | "REJECTED" | "MANUAL_REVIEW",
  "confidence": 0.92,
  "prescription_type": "COMPLETE_RX" | "COMPLETE_OTC" | "MIXED",
  "hash_id": "HASH20260131ABCD1234",
  "image_hash": "a1b2c3d4e5f6...",
  "content_hash": "sha256_hash_value",
  "refill_count": 0,
  "max_refills": 3,
  "prescription_date": "2026-01-25",
  "expiry_date": "2026-07-25",
  "extracted_medicines": [...],
  "matched_medicines": [...],
  "validation_layers": {
    "layer1": "PASS",
    "layer2": "PASS",
    "layer3": "NO_DUPLICATE",
    "layer4": {"confidence": 0.92, "extracted_count": 2},
    "layer5": {"quality_score": 0.88},
    "layer6": {"tampering_score": 0.12},
    "layer7": "NO_FACE",
    "layer8": "HASH_GENERATED",
    "layer9": "NOT_APPLICABLE",
    "layer10": {"matched": 2, "unmatched": 0}
  }
}
```

#### Post-Validation Storage (Privacy-First):
**Store in MongoDB:**
- `hash_id`, `user_id`, `image_hash`, `content_hash`
- `matched_medicines` (array of medicine_ids + quantities)
- `prescription_type`, `refill_count`, `max_refills`
- `prescription_date`, `expiry_date`
- `validation_result` (status, confidence)

**DELETE immediately:**
- Prescription image from MinIO
- `doctor_name`, `hospital_name`, `patient_name` (PII)
- Original extracted text

---

### Feature 3: Differential Cart Behavior by Prescription Type

#### Prescription Type 1: COMPLETE_RX
- All medicines are prescription-required (type = "RX")
- **Cart Behavior:**
  - All quantities **LOCKED** (user cannot edit)
  - Cannot delete individual medicines
  - Must order all medicines together
  - Refill rules enforced (max 3, 5-day gap, 180-day validity)
- **UI**: Show lock icon 🔒 on quantity selector
- **Backend**: Reject any quantity modification attempts

#### Prescription Type 2: COMPLETE_OTC
- All medicines are over-the-counter (type = "OTC")
- **Cart Behavior:**
  - All quantities **EDITABLE** (user can change)
  - Can delete individual medicines
  - Can add more OTC medicines without prescription
  - No refill limits
- **UI**: Normal quantity selector with +/- buttons
- **Backend**: Allow quantity modifications

#### Prescription Type 3: MIXED
- Some medicines RX, some OTC in same prescription
- **Cart Behavior:**
  - RX medicines: Quantities **LOCKED** 🔒, cannot delete
  - OTC medicines: Quantities **EDITABLE**, can delete
  - Refill rules apply to entire prescription (because of RX presence)
  - If user deletes all OTC items, RX items remain locked
- **UI**: Show lock icon only on RX medicines
- **Backend**: Validate each medicine's type before allowing modification

**Example MIXED Cart:**
```
Cart Items:
1. Paracetamol 500mg (OTC) - Qty: 30 [+] [-] [Delete] ✅ Editable
2. Azithromycin 500mg (RX) - Qty: 5 🔒 Cannot edit ❌
3. Vitamin D3 (OTC) - Qty: 60 [+] [-] [Delete] ✅ Editable
```

---

### Feature 4: Subscription Model

#### Subscription Types

##### 1. Instant Delivery (No Subscription)
- **Price**: Medicine cost + ₹50 delivery
- **Delivery**: Within 2 hours
- **Refund**: 24-hour window if not delivered
- **Use Case**: One-time acute illness, emergency

##### 2. Weekly Subscription
- **Price**: ₹249/week (10% medicine discount)
- **Frequency**: Every 7 days
- **Auto-charge**: On Day 7, auto-charge for next week
- **Refill Rules**: Max 3 refills, 5-day gap enforced
- **Cancellation**: Cancel within 4 days for full refund, after 4 days no refund
- **Auto-end**: Stops when prescription expires or max refills reached
- **Use Case**: Weekly medications (e.g., Insulin, TB drugs)

##### 3. Monthly Subscription
- **Price**: ₹799/month (10% medicine discount)
- **Frequency**: Every 30 days
- **Auto-charge**: On Day 30, auto-charge for next month
- **Refill Rules**: Max 3 refills, 5-day gap enforced
- **Cancellation**: Cancel within 15 days for full refund, after 15 days no refund
- **Auto-end**: Stops when prescription expires or max refills reached
- **Use Case**: Chronic disease management (diabetes, hypertension)

#### Refill Enforcement Logic
**Conditions for Refill:**
```python
refill_eligible = (
    prescription.refill_count < 3 AND
    days_since_last_order >= 5 AND
    today <= prescription.expiry_date
)
```

**Subscription Auto-End Scenarios:**
1. Prescription expired (180 days passed)
2. Max 3 refills completed
3. User manually cancels
4. Payment failure (retry 3 times, then pause)

**Database Schema (subscriptions collection):**
```json
{
  "_id": ObjectId("..."),
  "subscription_id": "SUB20260131ABCD",
  "user_id": "USR20260125XYZT",
  "hash_id": "HASH20260125MNOP",
  "type": "WEEKLY" | "MONTHLY" | "INSTANT",
  "status": "ACTIVE" | "PAUSED" | "CANCELLED" | "COMPLETED",
  "refill_count": 1,
  "max_refills": 3,
  "next_charge_date": "2026-02-07",
  "created_at": "2026-01-31T10:00:00Z",
  "cancelled_at": null,
  "end_reason": null
}
```

---

### Feature 5: AI Medical Consultant

#### 5-Step Assessment Flow

##### Step 1: Symptom Selection
- Show 10 common issues: Fever, Skin Rash, Cut/Wound, Burn, Headache, Joint Pain, Cough, Eye Issues, Stomach Pain, Other
- User selects one or types custom symptom
- **Output**: `symptom_type`

##### Step 2: Photo Upload (Optional)
- Allow upload of injury/symptom photo (JPG/PNG, max 5MB)
- Send to Google Cloud Vision API for analysis:
  - Detect bleeding, swelling, discoloration, burn degree
  - Return tags: `bleeding`, `swelling`, `burn_2nd_degree`, etc.
- **Output**: `photo_url`, `vision_analysis`

##### Step 3: Pain & Severity Questions
- Ask 15-20 targeted questions based on symptom:
  - Pain level (0-10 scale)
  - Duration (hours/days)
  - Location on body (visual body map)
  - Associated symptoms (nausea, fever, etc.)
  - Medical history (diabetes, allergies, etc.)
- **Output**: `pain_level`, `duration`, `location`, `associated_symptoms`, `medical_history`

##### Step 4: AI Severity Scoring
- Combine answers + photo analysis
- Calculate severity score (0-100):
  - **0-25**: Mild (home care)
  - **26-50**: Moderate (doctor consultation recommended)
  - **51-75**: Serious (urgent care within 24 hours)
  - **76-100**: Critical (emergency room immediately)
- Use simple rule-based logic for MVP (can train ML model later)
- **Output**: `severity_score`, `care_level`

##### Step 5: Recommendations & Actions
- Generate AI report with:
  - **Findings**: Summary of symptoms and photo analysis
  - **Severity**: Score + care level
  - **Recommendations**:
    - Home care instructions (if mild)
    - OTC medicine suggestions (e.g., "Paracetamol 500mg for pain")
    - Doctor consultation (if moderate/serious)
    - Nearest hospital with specialization (if critical)
  - **Warning Signs**: When to escalate care
- **Actions**:
  - Show nearby hospitals on Google Maps (with turn-by-turn directions)
  - One-click add OTC medicines to cart
  - Fast-track doctor booking with pre-filled consultation form
- **Output**: `ai_report`, `otc_suggestions`, `nearby_hospitals`, `doctor_recommendation`

**Database Schema (ai_consultations collection):**
```json
{
  "_id": ObjectId("..."),
  "consultation_id": "AICON20260131ABCD",
  "user_id": "USR20260125XYZT",
  "symptom_type": "Burn",
  "photo_url": "https://minio.../burn_photo.jpg",
  "vision_analysis": {"tags": ["burn_2nd_degree", "swelling"], "confidence": 0.88},
  "pain_level": 7,
  "duration": "2 hours",
  "location": "Right hand palm",
  "associated_symptoms": ["swelling", "redness"],
  "medical_history": ["none"],
  "severity_score": 72,
  "care_level": "URGENT_CARE",
  "ai_report": {
    "findings": "2nd-degree burn on right palm with swelling...",
    "recommendations": "Seek urgent care within 2 hours. Apply cool water...",
    "warning_signs": ["Increased pain", "Pus formation", "Fever"]
  },
  "otc_suggestions": ["MED009", "MED010"],
  "nearby_hospitals": [
    {"name": "Apollo Hospital", "distance": "2.3 km", "maps_url": "..."}
  ],
  "doctor_recommended": true,
  "created_at": "2026-01-31T10:30:00Z"
}
```

---

### Feature 6: Doctor Consultation

#### Entry Points

##### Entry 1: Homepage → General Consultation
- User clicks "Consult Doctor" button
- Shows doctor booking form (no pre-filled data)
- User selects specialty, language preference, time slot

##### Entry 2: Pipeline 2 → No Prescription Flow
- User searches medicine → Medicine detail page
- Clicks "I don't have a prescription" → Redirects to doctor booking
- Pre-fill consultation form with medicine name in "Reason for consultation"

##### Entry 3: AI Consultant → Recommended Doctor
- After AI report shows "Consult doctor recommended"
- Click "Book Specialist" button
- Pre-fill form with:
  - Symptom type from AI assessment
  - Severity score
  - AI report attached for doctor to review

#### Booking Flow

##### Step 1: Doctor Matching (AI-Powered)
- Backend filters doctors by:
  - Specialty (e.g., "Dermatology" for burns)
  - Language (user's preferred language from profile)
  - Availability (next 24-48 hours)
- Sort by: Rating, experience, availability
- Show top 3 doctors with profiles

##### Step 2: Time Slot Selection
- Show doctor's available slots (15-minute intervals)
- User selects date + time
- Fee: ₹199 per consultation

##### Step 3: Payment & Confirmation
- Mock payment for MVP (auto-success)
- Create consultation booking in database
- Send confirmation (show on screen, skip email/SMS for MVP)

##### Step 4: Video Call
- At scheduled time, user and doctor join video room
- Use Agora SDK or WebRTC (peer-to-peer)
- 30-minute session maximum
- Screen shows: Video, chat, countdown timer

##### Step 5: Digital Prescription Issuance
- After consultation, doctor opens "Issue Prescription" form
- Doctor enters:
  - Diagnosis
  - Medicines (search from database, select quantity)
  - Dosage instructions
  - Validity period (default 180 days)
- Click "Issue Prescription"
- Backend creates digital prescription:
  - Stores in `prescriptions` collection
  - Marks as `doctor_issued: true` (skips 10-layer validation)
  - Auto-uploads to user's account
  - Auto-populates user's cart
- User gets notification: "Prescription ready, view in cart"

**Database Schema (consultations collection):**
```json
{
  "_id": ObjectId("..."),
  "consultation_id": "CONSULT20260131ABCD",
  "user_id": "USR20260125XYZT",
  "doctor_id": "DOC001",
  "entry_point": "AI_CONSULTANT" | "HOMEPAGE" | "NO_PRESCRIPTION",
  "ai_consultation_id": "AICON20260131ABCD",
  "reason": "2nd-degree burn on right palm",
  "scheduled_time": "2026-01-31T15:00:00Z",
  "duration_minutes": 20,
  "status": "SCHEDULED" | "COMPLETED" | "CANCELLED",
  "video_room_id": "room_abc123",
  "prescription_issued": true,
  "prescription_id": "HASH20260131MNOP",
  "fee": 199,
  "created_at": "2026-01-31T10:45:00Z"
}
```

---

### Feature 7: Voice Assistant (TTS/STT)

#### Implementation
- **Technology**: Web Speech API (browser-native, no external API needed)
- **Component**: Floating mic button (global component on all authenticated pages)
- **Functionality**:
  1. User clicks mic button → Starts listening
  2. User speaks command (e.g., "Search for Paracetamol")
  3. Speech-to-Text converts audio → text
  4. Intent recognition (simple keyword matching for MVP):
     - "Search for [medicine]" → Navigate to `/medicines?q=[medicine]`
     - "Add to cart" → Add currently viewed medicine to cart (if on medicine detail page)
     - "Upload prescription" → Navigate to `/upload`
     - "Book doctor" → Navigate to `/doctor/book`
     - "View cart" → Navigate to `/cart`
     - "AI consultant" → Navigate to `/ai-consultant`
  5. Text-to-Speech responds: "Searching for Paracetamol" or "Added to cart"

**Supported Languages:**
- English (default)
- Hindi
- Tamil
- Telugu
- Bengali

**Code Structure:**
```typescript
// components/VoiceAssistant.tsx
const VoiceAssistant = () => {
  const recognition = new webkitSpeechRecognition();
  
  const handleVoiceCommand = (transcript: string) => {
    if (transcript.includes("search for")) {
      const medicine = extractMedicine(transcript);
      router.push(`/medicines?q=${medicine}`);
      speak("Searching for " + medicine);
    }
    // ... more intent handlers
  };
  
  return <FloatingMicButton onClick={startListening} />;
};
```

---

## Database Schema (MongoDB)

### Collection 1: users
```json
{
  "_id": ObjectId("..."),
  "user_id": "USR20260131A1B2C3D4",
  "phone": "+919876543210",
  "email": "user@example.com",
  "username": "rahul_kumar",
  "role": "USER",
  "profile": {
    "full_name": "Rahul Kumar",
    "age": 32,
    "gender": "Male",
    "address": "123 MG Road, Bangalore",
    "preferred_language": "English"
  },
  "firebase_uid": "firebase_unique_id",
  "created_at": ISODate("2026-01-31T00:00:00Z"),
  "updated_at": ISODate("2026-01-31T00:00:00Z")
}
```

### Collection 2: medicines
```json
{
  "_id": ObjectId("..."),
  "medicine_id": "MED001",
  "name": "Paracetamol",
  "generic_name": "Acetaminophen",
  "strength": "500mg",
  "form": "Tablet",
  "type": "OTC" | "RX",
  "manufacturer": "Cipla",
  "price": 25,
  "mrp": 30,
  "discount": 17,
  "pack_size": "15 tablets",
  "category": "Pain Relief",
  "description": "...",
  "usage": "...",
  "warnings": [],
  "side_effects": [],
  "stock": 500,
  "is_prescription_required": false,
  "image_url": null
}
```

### Collection 3: prescriptions
```json
{
  "_id": ObjectId("..."),
  "hash_id": "HASH20260131ABCD1234",
  "user_id": "USR20260125XYZT",
  "image_hash": "perceptual_hash_value",
  "content_hash": "sha256_hash_value",
  "prescription_type": "COMPLETE_RX" | "COMPLETE_OTC" | "MIXED",
  "medicines": [
    {"medicine_id": "MED001", "quantity": 30, "type": "OTC"},
    {"medicine_id": "MED003", "quantity": 5, "type": "RX"}
  ],
  "prescription_date": "2026-01-25",
  "expiry_date": "2026-07-25",
  "refill_count": 0,
  "max_refills": 3,
  "last_order_date": null,
  "validation_result": {
    "status": "VALID",
    "confidence": 0.92,
    "layers": {...}
  },
  "doctor_issued": false,
  "doctor_id": null,
  "created_at": ISODate("2026-01-31T00:00:00Z")
}
```

### Collection 4: cart
```json
{
  "_id": ObjectId("..."),
  "user_id": "USR20260125XYZT",
  "items": [
    {
      "medicine_id": "MED001",
      "name": "Paracetamol",
      "quantity": 30,
      "price": 25,
      "type": "OTC",
      "is_locked": false,
      "hash_id": "HASH20260131ABCD1234"
    },
    {
      "medicine_id": "MED003",
      "name": "Azithromycin",
      "quantity": 5,
      "price": 120,
      "type": "RX",
      "is_locked": true,
      "hash_id": "HASH20260131ABCD1234"
    }
  ],
  "total_amount": 750,
  "updated_at": ISODate("2026-01-31T11:00:00Z")
}
```

### Collection 5: orders
```json
{
  "_id": ObjectId("..."),
  "order_id": "ORD20260131ABCD",
  "user_id": "USR20260125XYZT",
  "hash_id": "HASH20260131ABCD1234",
  "items": [...],
  "total_amount": 750,
  "subscription_type": "WEEKLY" | "MONTHLY" | "INSTANT",
  "status": "PENDING" | "CONFIRMED" | "DELIVERED" | "CANCELLED",
  "payment_status": "PAID" | "PENDING" | "FAILED",
  "delivery_address": "...",
  "created_at": ISODate("2026-01-31T11:15:00Z"),
  "delivered_at": null
}
```

### Collection 6: subscriptions
```json
{
  "_id": ObjectId("..."),
  "subscription_id": "SUB20260131ABCD",
  "user_id": "USR20260125XYZT",
  "hash_id": "HASH20260131ABCD",
  "type": "WEEKLY" | "MONTHLY",
  "status": "ACTIVE" | "PAUSED" | "CANCELLED" | "COMPLETED",
  "refill_count": 1,
  "max_refills": 3,
  "next_charge_date": "2026-02-07",
  "created_at": ISODate("2026-01-31T11:15:00Z"),
  "cancelled_at": null
}
```

### Collection 7: ai_consultations
```json
{
  "_id": ObjectId("..."),
  "consultation_id": "AICON20260131ABCD",
  "user_id": "USR20260125XYZT",
  "symptom_type": "Burn",
  "severity_score": 72,
  "care_level": "URGENT_CARE",
  "ai_report": {...},
  "otc_suggestions": ["MED009", "MED010"],
  "nearby_hospitals": [...],
  "doctor_recommended": true,
  "created_at": ISODate("2026-01-31T10:30:00Z")
}
```

### Collection 8: consultations
```json
{
  "_id": ObjectId("..."),
  "consultation_id": "CONSULT20260131ABCD",
  "user_id": "USR20260125XYZT",
  "doctor_id": "DOC001",
  "entry_point": "AI_CONSULTANT",
  "ai_consultation_id": "AICON20260131ABCD",
  "scheduled_time": "2026-01-31T15:00:00Z",
  "status": "SCHEDULED" | "COMPLETED" | "CANCELLED",
  "prescription_issued": true,
  "prescription_id": "HASH20260131MNOP",
  "fee": 199,
  "created_at": ISODate("2026-01-31T10:45:00Z")
}
```

### Collection 9: doctors
```json
{
  "_id": ObjectId("..."),
  "doctor_id": "DOC001",
  "email": "doctor@example.com",
  "profile": {
    "full_name": "Dr. Priya Sharma",
    "specialty": "Dermatology",
    "experience_years": 10,
    "languages": ["English", "Hindi", "Tamil"],
    "rating": 4.8,
    "consultations_completed": 450
  },
  "availability": [
    {"day": "Monday", "slots": ["10:00", "10:15", "10:30", ...]},
  ],
  "created_at": ISODate("2026-01-01T00:00:00Z")
}
```

### Collection 10: pharmacists
```json
{
  "_id": ObjectId("..."),
  "pharmacist_id": "PHARM001",
  "email": "pharmacist@example.com",
  "profile": {
    "full_name": "Amit Patel",
    "license_number": "PH12345"
  },
  "created_at": ISODate("2026-01-01T00:00:00Z")
}
```

### Collection 11: admins
```json
{
  "_id": ObjectId("..."),
  "admin_id": "ADMIN001",
  "email": "admin@medsync.com",
  "password_hash": "bcrypt_hash",
  "role": "SUPER_ADMIN",
  "created_at": ISODate("2026-01-01T00:00:00Z")
}
```

### Collection 12: manual_review_queue
```json
{
  "_id": ObjectId("..."),
  "review_id": "REV20260131ABCD",
  "hash_id": "HASH20260131ABCD",
  "user_id": "USR20260125XYZT",
  "prescription_image_url": "https://minio.../prescription.jpg",
  "ai_extracted_medicines": [...],
  "confidence": 0.68,
  "quality_score": 0.74,
  "status": "PENDING" | "APPROVED" | "REJECTED",
  "pharmacist_id": null,
  "pharmacist_notes": null,
  "reviewed_at": null,
  "created_at": ISODate("2026-01-31T11:00:00Z")
}
```

---

## API Endpoints

### Authentication
```
POST /api/auth/user/register       - User registration (Phone OTP)
POST /api/auth/user/login          - User login (Phone OTP)
POST /api/auth/admin/login         - Admin login (JWT)
POST /api/auth/doctor/login        - Doctor login (JWT)
POST /api/auth/pharmacist/login    - Pharmacist login (JWT)
POST /api/auth/logout              - Logout (clear tokens)
GET  /api/auth/me                  - Get current user profile
```

### Medicines
```
GET  /api/medicines                - Get all medicines (paginated, filterable)
GET  /api/medicines/:id            - Get single medicine detail
POST /api/medicines                - Create medicine (Admin only)
PUT  /api/medicines/:id            - Update medicine (Admin only)
DELETE /api/medicines/:id          - Delete medicine (Admin only)
GET  /api/medicines/search?q=      - Search medicines by name
```

### Prescriptions (10-Layer Validation)
```
POST /api/prescriptions/upload                   - Upload prescription (Pipeline 1 & 2)
       Query params: ?medicine_id=MED001 (optional, for Pipeline 2)
GET  /api/prescriptions/:hash_id                 - Get prescription details
GET  /api/prescriptions/user/:user_id            - Get user's prescriptions
POST /api/prescriptions/check-refill/:hash_id    - Check refill eligibility
```

### Cart
```
GET  /api/cart/:user_id            - Get user's cart
POST /api/cart/add                 - Add item to cart
PUT  /api/cart/update/:item_id     - Update cart item (validates locked status)
DELETE /api/cart/remove/:item_id   - Remove item from cart (validates locked status)
DELETE /api/cart/clear/:user_id    - Clear entire cart
```

### Orders
```
POST /api/orders/create            - Create order (with subscription selection)
GET  /api/orders/:order_id         - Get order details
GET  /api/orders/user/:user_id     - Get user's orders
PUT  /api/orders/:order_id/cancel  - Cancel order
```

### Subscriptions
```
GET  /api/subscriptions/:subscription_id         - Get subscription details
GET  /api/subscriptions/user/:user_id            - Get user's subscriptions
POST /api/subscriptions/cancel/:subscription_id  - Cancel subscription
POST /api/subscriptions/reorder/:hash_id         - One-click reorder (check eligibility)
```

### AI Consultant
```
POST /api/ai-consultant/start               - Start AI consultation
POST /api/ai-consultant/upload-photo        - Upload symptom photo
POST /api/ai-consultant/submit-assessment   - Submit full assessment
GET  /api/ai-consultant/report/:id          - Get AI report
POST /api/ai-consultant/nearby-hospitals    - Get nearby hospitals (Google Maps)
```

### Doctor Consultation
```
GET  /api/consultations/doctors/match       - AI doctor matching
POST /api/consultations/book                - Book consultation
GET  /api/consultations/:consultation_id    - Get consultation details
GET  /api/consultations/user/:user_id       - Get user's consultations
POST /api/consultations/:id/join            - Join video room (generate token)
POST /api/consultations/:id/issue-prescription - Doctor issues digital Rx
```

### Manual Review (Pharmacist)
```
GET  /api/pharmacist/review-queue           - Get pending reviews
GET  /api/pharmacist/review/:review_id      - Get review details
POST /api/pharmacist/review/:review_id/approve - Approve prescription
POST /api/pharmacist/review/:review_id/reject  - Reject prescription
POST /api/pharmacist/review/:review_id/manual-entry - Manually add medicines
```

### Admin
```
GET  /api/admin/orders                      - Get all orders
GET  /api/admin/users                       - Get all users
GET  /api/admin/prescriptions               - Get all prescriptions
GET  /api/admin/fraud-logs                  - Get fraud detection logs
GET  /api/admin/analytics                   - Get analytics data
```

---

## Frontend Routing (Next.js App Router)

### User Pages
```
app/
├── page.tsx                              → Landing/Homepage
├── auth/
│   ├── login/page.tsx                    → User Login (Phone OTP)
│   └── register/page.tsx                 → User Registration
├── medicines/
│   ├── page.tsx                          → Browse Medicines
│   └── [id]/page.tsx                     → Medicine Detail
├── dashboard/
│   ├── page.tsx                          → User Dashboard
│   ├── orders/page.tsx                   → Order History
│   ├── prescriptions/page.tsx            → My Prescriptions
│   ├── subscriptions/page.tsx            → My Subscriptions
│   └── profile/page.tsx                  → Edit Profile
├── cart/page.tsx                         → Shopping Cart
├── checkout/page.tsx                     → Checkout
├── upload/page.tsx                       → Upload Prescription
├── ai-consultant/
│   ├── page.tsx                          → AI Consultant (5-step)
│   └── report/[id]/page.tsx              → AI Report
├── doctor/
│   ├── book/page.tsx                     → Book Doctor
│   ├── consultation/[id]/page.tsx        → Video Call Room
│   └── consultations/page.tsx            → My Consultations
```

### Admin Pages
```
app/admin/
├── login/page.tsx                        → Admin Login
├── dashboard/page.tsx                    → Admin Dashboard
├── orders/
│   ├── page.tsx                          → All Orders
│   └── [id]/page.tsx                     → Order Detail
├── medicines/
│   ├── page.tsx                          → Manage Medicines
│   ├── create/page.tsx                   → Add Medicine
│   └── [id]/edit/page.tsx                → Edit Medicine
├── prescriptions/
│   ├── page.tsx                          → All Prescriptions
│   └── [id]/page.tsx                     → Prescription Detail
├── users/
│   ├── page.tsx                          → All Users
│   └── [id]/page.tsx                     → User Detail
├── fraud-logs/page.tsx                   → Fraud Logs
└── analytics/page.tsx                    → Analytics
```

### Pharmacist Pages
```
app/pharmacist/
├── login/page.tsx                        → Pharmacist Login
├── dashboard/page.tsx                    → Pharmacist Dashboard
├── review-queue/page.tsx                 → Review Queue
├── review/[id]/page.tsx                  → Review Prescription
└── history/page.tsx                      → Review History
```

### Doctor Pages
```
app/doctor/
├── login/page.tsx                        → Doctor Login
├── dashboard/page.tsx                    → Doctor Dashboard
├── consultations/
│   ├── page.tsx                          → All Consultations
│   └── [id]/page.tsx                     → Consultation Detail
├── video-call/[id]/page.tsx              → Video Call Room
├── prescriptions/
│   ├── page.tsx                          → Issued Prescriptions
│   └── create/page.tsx                   → Issue Prescription Form
└── profile/page.tsx                      → Doctor Profile
```

---

## Key Implementation Notes

### 1. Validation Pipeline Reusability
- **DO NOT** duplicate validation code for Pipeline 1 and Pipeline 2
- Create single validation function: `validatePrescription(image, medicine_id=None)`
- **Pipeline 1**: Call `validatePrescription(image)` → Returns **ALL** medicines
- **Pipeline 2**: Call `validatePrescription(image, "MED001")` → Returns **ONLY** matched medicine

### 2. Cart Locking Enforcement
- **Frontend**: Disable quantity controls for locked items (visual only)
- **Backend**: **MUST** validate every cart update request:
```python
if item.type == "RX" and item.is_locked:
    raise HTTPException(403, "Cannot modify locked prescription medicine")
```

### 3. Refill Detection Flow
1. Layer 3 finds duplicate → Check if same user
2. If same user → Skip Layers 4-8, jump to Layer 9
3. Layer 9 checks refill eligibility → Allow or deny
4. **Never re-run OCR for refill attempts** (use stored data)

### 4. Privacy-First Storage
**After validation passes:**
- **Store**: `hash_id`, `image_hash`, `content_hash`, `medicines`, dates, refill rules
- **DELETE**: Prescription image, doctor name, patient name, hospital name
- **Never store PII after validation completes**

### 5. Subscription Auto-End
Run daily cron job to check subscriptions:
```python
if subscription.next_charge_date == today:
    if prescription.expired or refill_count >= 3:
        subscription.status = "COMPLETED"
    else:
        charge_payment()
        subscription.refill_count += 1
```

### 6. Doctor-Issued Prescriptions
- Skip 10-layer validation (doctor is trusted source)
- Mark as `doctor_issued: true`
- Still store `hash_id` for refill tracking
- Still enforce refill rules (max 3, 5-day gap, 180-day validity)

### 7. Voice Assistant Intent Matching
Use simple keyword matching for MVP:
- "search" OR "find" → Search medicine
- "add to cart" → Add item
- "upload" → Upload prescription
- "doctor" OR "consult" → Book doctor
- "AI" OR "triage" OR "symptoms" → AI consultant
- Can replace with NLP model later

---

## Code Style Guidelines

### TypeScript (Frontend)
- Use functional components with hooks
- Use TypeScript strict mode
- Define interfaces for all data types
- Use Zod for form validation
- Use React Query for API calls
- Use Zustand for global state

### Python (Backend)
- Use type hints for all functions
- Use Pydantic models for request/response validation
- Use async/await for all routes
- Use dependency injection for auth
- Follow PEP 8 style guide

### Naming Conventions
- MongoDB collections: lowercase plural (e.g., `medicines`, `prescriptions`)
- API routes: lowercase with hyphens (e.g., `/api/ai-consultant`)
- Frontend routes: lowercase with hyphens (e.g., `/ai-consultant`)
- Component files: PascalCase (e.g., `VoiceAssistant.tsx`)
- Function names: camelCase (e.g., `validatePrescription`)

---

## External API Keys Needed

```bash
# Frontend (.env.local)
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=
NEXT_PUBLIC_AGORA_APP_ID=

# Backend (.env)
MONGODB_URI=mongodb://localhost:27017/medsync
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=
MINIO_SECRET_KEY=

GEMINI_API_KEY=
GOOGLE_CLOUD_VISION_API_KEY=

FIREBASE_ADMIN_CREDENTIALS=./firebase-admin-key.json

JWT_SECRET=your_super_secret_key
JWT_ALGORITHM=HS256

RAZORPAY_KEY_ID=
RAZORPAY_KEY_SECRET=
```

---

## Testing Scenarios

### Prescription Upload Tests
1. Upload valid prescription → Should extract all medicines
2. Upload blurry prescription → Should go to manual review
3. Upload tampered prescription → Should reject
4. Upload duplicate (same user) → Should check refill eligibility
5. Upload duplicate (different user) → Should flag as fraud
6. Upload prescription with photo of person → Should reject (face detected)

### Cart Tests
1. Add RX medicine → Should lock quantity
2. Add OTC medicine → Should allow quantity edit
3. Try to edit locked RX quantity → Should reject
4. Add mixed prescription → RX locked, OTC editable
5. Delete OTC from mixed cart → Should allow
6. Try to delete RX from cart → Should reject

### Refill Tests
1. Attempt refill before 5 days → Should reject
2. Attempt 4th refill → Should reject (max 3)
3. Attempt refill after 180 days → Should reject (expired)
4. Valid refill attempt → Should allow, increment count

### AI Consultant Tests
1. Submit burn symptom + photo → Should calculate severity score
2. Mild severity → Should recommend home care
3. Critical severity → Should show nearest ER
4. Click "Book Doctor" → Should pre-fill consultation form

### Doctor Consultation Tests
1. Book doctor from homepage → General consultation
2. Book doctor from "No prescription" flow → Pre-fill medicine name
3. Book doctor from AI report → Pre-fill symptom + attach report
4. Doctor issues digital Rx → Should auto-upload to user cart

---

## Development Workflow

### Setup:
1. Clone repo
2. Install dependencies: `npm install` (frontend), `pip install -r requirements.txt` (backend)
3. Set up `.env` files
4. Start MongoDB, MinIO
5. Run dummy data script: `python populate_medicines.py`

### Run Dev Servers:
- **Frontend**: `npm run dev` (http://localhost:3000)
- **Backend**: `uvicorn main:app --reload` (http://localhost:8000)

### Test Flow:
1. Create user account (phone OTP)
2. Upload prescription (test validation pipeline)
3. Add to cart, checkout
4. Test AI consultant
5. Book doctor consultation
6. Test subscription model

### Deploy (for hackathon):
- **Frontend**: Vercel
- **Backend**: Railway / Render
- **Database**: MongoDB Atlas
- **File Storage**: MinIO Cloud / AWS S3

---

## Common Pitfalls to Avoid

1. **DO NOT** duplicate validation code for 2 pipelines
2. **DO NOT** allow frontend-only cart locking (must validate backend)
3. **DO NOT** re-run OCR for refill attempts (use stored data)
4. **DO NOT** store prescription images after validation
5. **DO NOT** skip refill eligibility checks
6. **DO NOT** hard-code prescription types (calculate dynamically)
7. **DO NOT** forget to delete PII after validation
8. **DO NOT** allow RX quantity edits in cart (strict enforcement)

---

## Success Metrics (Demo)

1. ✅ User can upload prescription and get cart auto-filled in <5 seconds
2. ✅ Cart correctly locks RX medicines, allows OTC editing
3. ✅ Refill system prevents >3 refills, <5 day gaps, expired prescriptions
4. ✅ AI consultant generates severity score and hospital recommendations
5. ✅ Doctor can issue digital prescription that auto-populates user cart
6. ✅ Voice assistant navigates to correct pages
7. ✅ All 4 roles (User/Admin/Doctor/Pharmacist) can authenticate and access respective dashboards

---

**END OF COPILOT INSTRUCTIONS**
