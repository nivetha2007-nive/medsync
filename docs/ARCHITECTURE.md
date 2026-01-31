# MedSync System Architecture

## Overview

MedSync is a full-stack digital pharmacy platform with AI-powered features built on a modern microservices-inspired architecture.

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                         Frontend                              │
│  (Next.js 15 + React 19 + TypeScript + Tailwind)            │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ User Portal  │  │ Admin Panel  │  │ Doctor Portal│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │  Pharmacist  │  │ Voice Assistant                         │
│  └──────────────┘  └──────────────┘                         │
└───────────────────────────┬──────────────────────────────────┘
                            │ REST API (Axios)
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                         Backend                               │
│              (FastAPI + Python 3.11+)                        │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              API Routes (v1)                          │   │
│  │  auth, medicines, prescriptions, cart, orders,       │   │
│  │  subscriptions, ai-consultant, consultations         │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           10-Layer Validation Pipeline                │   │
│  │  (Format → Upload → Duplicate → OCR → Quality →      │   │
│  │   Tampering → Face → Hash → Refill → Matching)       │   │
│  └──────────────────────────────────────────────────────┘   │
│                            │                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Business Logic Services                  │   │
│  │  validation, cart, order, subscription, ai, video    │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────┬──────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   MongoDB    │   │    MinIO     │   │    Redis     │
│  (Database)  │   │  (Storage)   │   │  (Cache)     │
└──────────────┘   └──────────────┘   └──────────────┘

        ┌───────────────────┴───────────────────┐
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Firebase   │   │  Google AI   │   │  Razorpay    │
│  (Phone OTP) │   │  (Gemini +   │   │  (Payments)  │
│              │   │   Vision)    │   │              │
└──────────────┘   └──────────────┘   └──────────────┘
```

## Technology Stack

### Frontend Layer
- **Framework**: Next.js 15 with App Router
- **Language**: TypeScript (strict mode)
- **UI Components**: shadcn/ui + Tailwind CSS
- **State Management**: 
  - Zustand (global auth state)
  - React Query (server state)
  - Zustand (cart state)
- **Authentication**: Firebase Phone Auth SDK
- **Video**: Agora SDK
- **Voice**: Web Speech API (browser-native)
- **Maps**: Google Maps JavaScript API

### Backend Layer
- **Framework**: FastAPI
- **Language**: Python 3.11+
- **Validation**: Pydantic v2
- **Authentication**: 
  - Firebase Admin SDK (User verification)
  - JWT (Admin/Doctor/Pharmacist)
- **AI/ML**:
  - Google Gemini 2.0 Flash (OCR)
  - Google Cloud Vision (Image analysis)
- **Image Processing**: OpenCV, PIL, imagehash
- **Background Tasks**: Celery + Redis

### Data Layer
- **Primary Database**: MongoDB 7.0
- **Object Storage**: MinIO
- **Cache**: Redis
- **Session Store**: Redis

### External Services
- **Phone Auth**: Firebase Authentication
- **AI OCR**: Google Gemini 2.0 Flash
- **Image Analysis**: Google Cloud Vision API
- **Maps**: Google Maps API
- **Payments**: Razorpay
- **Video**: Agora RTC

## Data Flow

### Prescription Upload Flow

```
User → Upload Image → Frontend
                         ↓
              FastAPI Endpoint
                         ↓
            Validation Pipeline (10 Layers)
                         ↓
         ┌───────────────┴───────────────┐
         ▼                               ▼
    MongoDB                          MinIO
  (Store hash_id,              (Temporary storage,
   medicines,                   deleted after
   refill rules)                validation)
         │
         ▼
    Cart Auto-Populate
```

### Order Flow with Subscription

```
User → Add to Cart → Checkout
                         ↓
              Select Subscription Type
                         ↓
         (Instant / Weekly / Monthly)
                         ↓
                    Payment
                         ↓
                  Create Order
                         ↓
         ┌───────────────┴───────────────┐
         ▼                               ▼
    orders collection           subscriptions collection
         │                               │
         ▼                               ▼
    Delivery                      Celery Daily Check
                                   (Auto-charge refills)
```

### AI Consultant Flow

```
User → Select Symptom → Upload Photo (optional)
                         ↓
              Answer 15-20 Questions
                         ↓
         AI Severity Scoring Algorithm
                         ↓
         ┌───────────────┴───────────────┐
         ▼               ▼               ▼
    Mild (0-25)    Moderate (26-50)  Serious/Critical (51-100)
         │               │               │
         ▼               ▼               ▼
   Home Care      Doctor Booking    Hospital Finder
   + OTC Meds                        (Google Maps)
```

## Security Architecture

### Authentication Flow

```
┌─────────────────────────────────────────────────────────┐
│  User (Patient)                                         │
│  ↓                                                       │
│  Firebase Phone OTP → UID → Backend Verification →      │
│  JWT Token (short-lived) → Stored in httpOnly cookie   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Admin / Doctor / Pharmacist                            │
│  ↓                                                       │
│  Email + Password → JWT (long-lived) →                  │
│  Refresh Token → Role-based access control              │
└─────────────────────────────────────────────────────────┘
```

### Privacy-First Design

**Prescription Validation Storage:**
- ✅ **Store**: `hash_id`, `image_hash`, `content_hash`, `medicines`, `dates`, `refill_count`
- ❌ **Delete**: Prescription image, `doctor_name`, `patient_name`, `hospital_name`

**Rationale**: No PII stored after validation completes (GDPR/HIPAA compliant).

## Scalability Considerations

### Horizontal Scaling
- **Frontend**: Vercel Edge Network (automatic)
- **Backend**: Multiple FastAPI instances behind load balancer
- **Database**: MongoDB sharding (user_id as shard key)
- **Storage**: MinIO distributed mode

### Caching Strategy
- **Redis**: 
  - User sessions (15 min TTL)
  - Medicine catalog (1 hour TTL)
  - Doctor availability (5 min TTL)

### Background Jobs
- **Celery Workers**:
  - Daily subscription checks
  - Refill auto-charges
  - Temp file cleanup
  - Email/SMS notifications (future)

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Production                          │
└─────────────────────────────────────────────────────────┘

Frontend (Vercel)
  ↓
CDN (Cloudflare/Vercel Edge)
  ↓
Backend (Railway/Render)
  ↓ ← Load Balancer
  ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ FastAPI      │  │ FastAPI      │  │ FastAPI      │
│ Instance 1   │  │ Instance 2   │  │ Instance 3   │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        └─────────────────┴─────────────────┘
                          ↓
        ┌─────────────────┴─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ MongoDB      │  │ MinIO/S3     │  │ Redis        │
│ Atlas        │  │ Cloud        │  │ Cloud        │
└──────────────┘  └──────────────┘  └──────────────┘
```

## Performance Targets

- **Prescription Validation**: < 5 seconds (10 layers)
- **API Response Time**: < 200ms (p95)
- **Cart Operations**: < 100ms
- **AI Consultant**: < 3 seconds (severity scoring)
- **Video Call Latency**: < 200ms (peer-to-peer)

## Monitoring & Observability

- **Frontend**: Vercel Analytics
- **Backend**: FastAPI + Prometheus
- **Logs**: Structured JSON logs (ELK stack future)
- **Errors**: Sentry (future)
- **Uptime**: UptimeRobot (future)

## Disaster Recovery

- **Database**: MongoDB Atlas automated backups (daily)
- **Storage**: MinIO versioning enabled
- **Code**: GitHub (all branches backed up)
- **Secrets**: Environment variables in secure vault

## Future Enhancements

1. **Microservices Split**:
   - Prescription Service
   - Order Service
   - Consultation Service
   - AI Service

2. **Event-Driven Architecture**:
   - Kafka/RabbitMQ for async events
   - CQRS pattern for read-heavy operations

3. **GraphQL API**:
   - Alternative to REST for flexible queries

4. **Mobile Apps**:
   - React Native (iOS + Android)

---

**Architecture Version**: 1.0  
**Last Updated**: January 31, 2026
