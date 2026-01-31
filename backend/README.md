# MedSync Backend - FastAPI

AI-powered pharmacy platform backend with prescription validation, medical triage, and fraud prevention.

## Tech Stack

- **Framework**: FastAPI (Python 3.11+)
- **Database**: MongoDB
- **Storage**: MinIO
- **AI/ML**: Google Gemini 2.0 Flash, Google Cloud Vision
- **Image Processing**: OpenCV, PIL, imagehash
- **Authentication**: Firebase Admin SDK, JWT
- **Payment**: Razorpay (test mode)

## Project Structure

```
backend/
├── app/
│   ├── api/v1/              # API routes
│   ├── core/                # Security, auth
│   ├── db/                  # Database connections
│   ├── schemas/             # Pydantic models
│   ├── services/            # Business logic
│   ├── validation/          # 10-layer validation pipeline
│   ├── utils/               # Utilities
│   ├── middleware/          # Middleware
│   └── tasks/               # Background tasks
├── tests/                   # Tests
├── scripts/                 # Utility scripts
└── docs/                    # Documentation
```

## Setup

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Environment Variables

Copy `.env.example` to `.env` and fill in values:

```bash
cp .env.example .env
```

### 3. Start Services

Start MongoDB and MinIO using Docker:

```bash
docker-compose up -d
```

### 4. Populate Database

```bash
python scripts/populate_medicines.py
python scripts/create_admin.py
python scripts/create_doctor.py
python scripts/create_pharmacist.py
```

### 5. Run Development Server

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

API will be available at: http://localhost:8000
API Documentation: http://localhost:8000/docs

## Features

### 10-Layer Prescription Validation Pipeline

1. **Layer 1**: File format validation
2. **Layer 2**: Upload to MinIO
3. **Layer 3**: Duplicate detection (image hash)
4. **Layer 4**: AI OCR extraction (Gemini)
5. **Layer 5**: Image quality check
6. **Layer 6**: Tampering detection
7. **Layer 7**: Face detection (privacy)
8. **Layer 8**: Content hash generation
9. **Layer 9**: Refill eligibility check
10. **Layer 10**: Medicine database matching

### Key Endpoints

- **Authentication**: `/api/v1/auth/*`
- **Medicines**: `/api/v1/medicines/*`
- **Prescriptions**: `/api/v1/prescriptions/*`
- **Cart**: `/api/v1/cart/*`
- **Orders**: `/api/v1/orders/*`
- **Subscriptions**: `/api/v1/subscriptions/*`
- **AI Consultant**: `/api/v1/ai-consultant/*`
- **Consultations**: `/api/v1/consultations/*`

## Testing

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=html

# Run specific test
pytest tests/validation/test_pipeline.py
```

## Deployment

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for deployment instructions.

## License

MIT
