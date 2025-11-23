# AI-Powered Interview Platform

> Full-stack AI interview platform that automates technical recruitment through intelligent video interviews and CV screening

[![Next.js](https://img.shields.io/badge/Next.js-15-black)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.104-009688)](https://fastapi.tiangolo.com/)

---

## 🎯 Overview

An enterprise-grade platform that leverages AI to conduct video interviews, screen resumes, and rank candidates automatically. Built with Next.js 15 and FastAPI, supporting 1000+ concurrent users.

**Key Results:**
- 🚀 70% reduction in candidate screening time
- 📊 500+ CVs processed simultaneously
- ⚡ 80% reduction in manual review effort

---

## 🛠️ Tech Stack

### Frontend
- **Framework**: Next.js 15 (App Router), React 18
- **Language**: TypeScript 5.0
- **Styling**: Tailwind CSS
- **State**: Zustand, React Query (TanStack Query)
- **Forms**: React Hook Form + Zod validation
- **Media**: WebRTC for video recording

### Backend
- **Framework**: FastAPI (Python 3.11+)
- **Database**: PostgreSQL 15
- **Cache**: Redis 7.0
- **AI**: OpenAI GPT-4, Google Gemini
- **ML**: LangChain, Sentence Transformers, ChromaDB
- **Queue**: Celery (async tasks)

### DevOps
- **Containers**: Docker, Docker Compose
- **CI/CD**: GitHub Actions
- **Monitoring**: Sentry, Structured logging

---

## ✨ Key Features

### 🤖 AI-Powered Interviews
- Conversational AI with GPT-4 and Gemini
- Text-to-speech question delivery
- Real-time video/audio recording
- Automatic transcription and analysis

### 📄 Intelligent CV Screening
- Automated PDF/DOCX parsing
- Semantic job-candidate matching
- Batch processing (500+ CVs)
- AI-powered ranking

### 👥 Multi-tenant Architecture
- Three user types: Candidates, HR Managers, Guests
- Role-based access control (RBAC)
- Time-limited guest interview links
- Organization-level data isolation

### 📊 Real-time Analytics
- Performance metrics and dashboards
- Recruitment pipeline visualization
- Candidate insights with Chart.js
- Export to PDF/Excel

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│         Next.js 15 Frontend                  │
│  (SSR, TypeScript, Tailwind, Zustand)       │
└──────────────────┬──────────────────────────┘
                   │ REST API
┌──────────────────▼──────────────────────────┐
│         FastAPI Microservices                │
│  (Domain-Driven Design, Async)              │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  PostgreSQL │ Redis │ ChromaDB │ AI APIs    │
└─────────────────────────────────────────────┘
```

**Design Patterns:**
- Repository Pattern for data access
- Dependency Injection (FastAPI)
- Multi-provider AI abstraction
- Event-driven architecture

---

## 👤 User Workflows

### Candidate Journey
```
Register → Upload CV → AI Analysis → Camera Setup →
Live Interview → AI Questions → Video Recording → Results
```

### HR Manager Journey
```
Create Job → Set Requirements → Auto-screen CVs →
Review Rankings → Generate Interview Links → Evaluate Results
```

### Guest Interview
```
Receive Link → Basic Info → Tech Check → Interview → Complete
```

---

## 📸 Screenshots

> Add your screenshots here following the structure in `screenshots/` folder

### Candidate Portal
<!-- ![Dashboard](./screenshots/candidate/dashboard/candidate-dashboard-main-01.png) -->

### HR Portal
<!-- ![HR Dashboard](./screenshots/hr/dashboard/hr-dashboard-main-01.png) -->

### Live Interview
<!-- ![Interview](./screenshots/candidate/interview/candidate-interview-live-01.png) -->

---

## 📈 Impact & Performance

| Metric | Value |
|--------|-------|
| **Screening Time Reduction** | 70% |
| **Concurrent Users** | 1000+ |
| **CV Processing** | 500+ simultaneous |
| **API Response Time (p95)** | < 300ms |
| **Page Load Time** | < 2 seconds |
| **System Uptime** | 99.9% |

---

## 🔒 Security

- JWT authentication with refresh tokens
- Role-based access control (RBAC)
- Encryption at rest (AES-256) and in transit (TLS 1.3)
- Rate limiting (100 req/min per user)
- GDPR-compliant data handling

---

## 🚀 Technical Highlights

### Frontend
- **Server-Side Rendering** for fast page loads
- **Protected Routes** with middleware authentication
- **Optimistic Updates** for instant UI feedback
- **Code Splitting** for performance
- **WebRTC** for browser-based recording

### Backend
- **Microservices** architecture
- **Async Processing** with Celery
- **Multi-level Caching** (Redis)
- **AI Fallback** mechanisms
- **Database Optimization** (connection pooling, indexes)

---

## 🎓 Skills Demonstrated

✅ Full-stack development (Next.js + FastAPI)
✅ AI/ML integration (OpenAI, Gemini, LangChain)
✅ Real-time features (WebRTC)
✅ State management (Zustand, React Query)
✅ Database design & optimization
✅ Microservices architecture
✅ DevOps (Docker, CI/CD)
✅ Security best practices

---

## 👨‍💻 Developer

**Tanvir Rahman Anik**
Machine Learning Engineer @ Interactive Cares
📧 tranik.cse@gmail.com
🔗 [LinkedIn](https://linkedin.com/in/TR.Anik) | [GitHub](https://github.com/anik81) | [Portfolio](https://anik81.github.io)

---

## 📝 Disclaimer

This documentation is for portfolio purposes. All screenshots use demo data. Metrics are approximate. This project was developed at Interactive Cares and shared with approval.

---

## 📄 License

Proprietary - Interactive Cares

---

**Built with** ❤️ **using Next.js, TypeScript, FastAPI, and AI**
