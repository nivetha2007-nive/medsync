# MedSync - AI-Powered Digital Pharmacy Platform

**Secure e-commerce pharmacy with AI prescription validation, medical triage, and fraud prevention.**

## 🚀 Project Overview

MedSync is a comprehensive digital pharmacy platform featuring:

- ✅ **10-Layer Prescription Validation** with AI-powered OCR
- ✅ **Dual Upload Pipelines** (extract all medicines vs. match specific medicine)
- ✅ **Differential Cart Behavior** (RX locked, OTC editable)
- ✅ **Subscription Model** (Instant, Weekly, Monthly)
- ✅ **AI Medical Consultant** with severity scoring
- ✅ **Doctor Video Consultations** with digital prescription issuance
- ✅ **Voice Assistant** for hands-free navigation
- ✅ **4 User Roles** (User, Admin, Doctor, Pharmacist)

## 🛠️ Tech Stack

### Frontend
- Next.js 15 (App Router) + TypeScript
- React 19 + Tailwind CSS + shadcn/ui
- Zustand + React Query
- Firebase Phone Auth
- Agora SDK (Video)
- Web Speech API (Voice)

### Backend
- FastAPI + Python 3.11+
- MongoDB + MinIO
- Google Gemini 2.0 Flash (OCR)
- Google Cloud Vision (Tampering Detection)
- OpenCV + PIL + imagehash
- Razorpay (Payments)

## 📁 Project Structure

```
medsync/
├── frontend/          # Next.js 15 frontend
├── backend/           # FastAPI backend
├── docs/              # Documentation
├── .github/           # CI/CD workflows
└── docker-compose.yml # MongoDB, MinIO, Redis
```

## 🚦 Quick Start

### Prerequisites

- Node.js 18+
- Python 3.11+
- Docker (for MongoDB, MinIO)

### 1. Clone Repository

```bash
git clone https://github.com/yourusername/medsync.git
cd medsync
```

### 2. Setup Backend

```bash
cd backend
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your API keys
docker-compose up -d  # Start MongoDB, MinIO
python scripts/populate_medicines.py
uvicorn app.main:app --reload
```

Backend runs at: http://localhost:8000

### 3. Setup Frontend

```bash
cd frontend
npm install
cp .env.example .env.local
# Edit .env.local with your API keys
npm run dev
```

Frontend runs at: http://localhost:3000

## 📖 Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [Database Schema](docs/DATABASE_SCHEMA.md)
- [API Endpoints](docs/API_ENDPOINTS.md)
- [Validation Pipeline](docs/VALIDATION_PIPELINE.md)
- [Deployment Guide](docs/DEPLOYMENT.md)

## 🧪 Testing

### Backend
```bash
cd backend
pytest --cov=app
```

### Frontend
```bash
cd frontend
npm test
```

## 🔑 Key Features

### 10-Layer Prescription Validation

1. File format validation
2. Upload to MinIO
3. Duplicate detection (image hash)
4. AI OCR extraction (Gemini)
5. Image quality check
6. Tampering detection
7. Face detection (privacy)
8. Content hash generation
9. Refill eligibility check
10. Medicine database matching

**Privacy-First**: All PII (doctor name, patient name, hospital) deleted after validation.

### Differential Cart Behavior

- **COMPLETE_RX**: All quantities locked 🔒
- **COMPLETE_OTC**: All quantities editable ✏️
- **MIXED**: RX locked 🔒, OTC editable ✏️

### Subscription Model

- **Instant**: One-time delivery (₹50)
- **Weekly**: ₹249/week (10% discount, max 3 refills)
- **Monthly**: ₹799/month (10% discount, max 3 refills)

Refill rules enforced: Max 3 refills, 5-day gap, 180-day validity.

## 👥 User Roles

1. **User** (Phone OTP) - Browse, order, upload prescriptions, consult doctors
2. **Admin** (JWT) - Manage medicines, view orders, fraud logs, analytics
3. **Doctor** (JWT) - Video consultations, issue digital prescriptions
4. **Pharmacist** (JWT) - Review low-confidence prescriptions (<75%)

## 🚀 Deployment

- **Frontend**: Vercel
- **Backend**: Railway / Render
- **Database**: MongoDB Atlas
- **Storage**: MinIO Cloud / AWS S3

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for detailed instructions.

## 📄 License

MIT License - see [LICENSE](LICENSE) file.

## 🤝 Contributing

See [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md) for contribution guidelines.

## 📞 Support

For issues or questions, open a GitHub issue or contact: support@medsync.com

---

**Built with ❤️ for hackathons and healthcare innovation.**
