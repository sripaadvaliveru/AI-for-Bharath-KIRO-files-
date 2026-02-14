# Vyasa AI - Design Document

## 1. System Architecture

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Client Layer: Mobile App | Kiosk | WhatsApp Bot | IVR      │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  API Gateway: Load Balancer + Auth + Rate Limiting          │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Microservices: Voice | Scheme | User | Analytics           │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Data Layer: PostgreSQL | Redis | Cloud Storage | Vector DB │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  External: Bhashini API | Google Cloud | WhatsApp | Twilio  │
└─────────────────────────────────────────────────────────────┘
```

**Key Principles**:
- Hybrid architecture: Local AI models + cloud services
- Offline-first design for core features
- Microservices for scalability
- Multi-channel support from single backend

## 2. Component Design

### 2.1 Mobile Application (Flutter)

**Architecture**: Clean Architecture (Presentation → Domain → Data)

**State Management**: Riverpod

**Local Storage (SQLite)**:
- schemes: Complete scheme database
- user_profile: Optional user data
- query_history: Cached queries for sync
- favorites: Saved schemes
- voice_cache: Offline voice data

**Offline Voice**: Vosk models (~50MB per language) with lazy loading

**Key Screens**: Home | Scheme List | Scheme Detail | Profile | Settings | Kiosk Mode

### 2.2 Backend Services

#### Voice Service (FastAPI)
- Speech-to-text (Bhashini primary, Google Cloud fallback, Vosk offline)
- Text-to-speech (Google Cloud TTS)
- Language detection
- Audio preprocessing with FFmpeg

**Endpoints**: `/voice/speech-to-text`, `/voice/text-to-speech`, `/voice/detect-language`

#### Scheme Service (FastAPI)
- Recommendation engine based on user profile
- Eligibility checking
- Search with full-text and semantic search (Vector DB)
- Content management

**Recommendation Logic**:
1. Filter by location (state, district)
2. Filter by demographics (age, gender, occupation)
3. Score by relevance
4. Return top 10 schemes

**Endpoints**: `/schemes/recommend`, `/schemes/check-eligibility`, `/schemes/search`, `/schemes/{id}`

#### User Service (FastAPI)
- OTP-based authentication (5-min validity, 3 requests/hour limit)
- Profile management
- Consent tracking
- JWT tokens (30-day expiry)

**Endpoints**: `/users/send-otp`, `/users/verify-otp`, `/users/profile`, `/users/consent`

#### Analytics Service (FastAPI)
- Usage tracking (queries, users, schemes)
- Geographic distribution
- Report generation
- Grafana dashboards

**Endpoints**: `/analytics/track-query`, `/analytics/dashboard-data`, `/analytics/reports/*`

### 2.3 Database Design

**PostgreSQL Schema**:
```sql
schemes: id, name_{lang}, description_{lang}, category, state, district, 
         eligibility (JSONB), documents (JSONB), deadline, is_active

users: id, phone_hash, age, occupation, state, district, village, 
       gender, language, created_at

user_consents: id, user_id, consent_type, granted, granted_at

query_logs: id, user_id, query_text, query_language, response_text, 
            schemes_recommended (JSONB), timestamp, channel

favorites: id, user_id, scheme_id, added_at
```

**Redis Cache**:
- `session:{id}`: User session data
- `otp:{phone_hash}`: OTP codes with expiry
- `scheme:{id}`: Cached scheme data
- `rate_limit:otp:{phone}`: Rate limiting counters

### 2.4 Voice Processing Pipeline

**Speech-to-Text**:
Audio → Validation → Preprocessing → Language Detection → STT (Bhashini/Google/Vosk) → Post-processing → Text

**Text-to-Speech**:
Text → Validation → Chunking (2-min segments) → TTS (Google Cloud) → Caching → Audio

**Code-Mixing**: Detect language spans, use bilingual models or process separately

## 3. Integration Design

### 3.1 WhatsApp Bot
- WhatsApp Business API via Twilio/360Dialog
- Webhook-based message handling
- Conversation states: Welcome → Language → Profile → Query → Scheme Browsing → Feedback

### 3.2 IVR System (Twilio)
- Call flow: Greeting → Language (DTMF) → Voice Input → STT → Recommendation → TTS → Options
- DTMF menu: 1=Telugu, 2=Hindi, 3=Tamil, 4=Kannada

### 3.3 Admin Dashboard (React + Material-UI)
- Scheme management (CRUD, multilingual editor, bulk import)
- Analytics dashboard (real-time metrics, heatmaps)
- User management (anonymized data, deletion requests)
- Content moderation

## 4. Security & Privacy

**Authentication**:
- Mobile: Optional OTP + JWT
- Admin: Email/password + 2FA + RBAC

**Encryption**:
- In transit: TLS 1.3, certificate pinning
- At rest: AES-256 for user data, encrypted backups

**Privacy**:
- Anonymous usage by default
- Data retention: Query logs 90 days → anonymized, Voice recordings 30 days → deleted
- DPDPA compliance, clear consent mechanisms

## 5. Deployment

**Infrastructure**: Google Cloud Platform (GCP)
- GKE for Kubernetes orchestration
- Cloud SQL (PostgreSQL), Memorystore (Redis)
- Cloud Storage for voice recordings
- Cloud CDN for static assets

**Kubernetes**:
- Namespaces: production, staging, development
- Deployments: voice-service (3 replicas), scheme-service (3 replicas), user-service (2), analytics-service (2)
- Auto-scaling: Min 3, Max 20 replicas based on 70% CPU utilization

**CI/CD**: GitHub Actions
Pipeline: Checkout → Tests → Build → Push → Deploy Staging → Smoke Tests → Approval → Deploy Production

## 6. Monitoring

**Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)

**Metrics**: Prometheus + Grafana
- Request rate, response time (p50, p95, p99), error rate, CPU/memory, cache hit rate

**Alerts**:
- Error rate > 5% for 5 min
- Response time p95 > 3 sec
- Service down > 1 min

## 7. Offline Functionality

**Sync Strategy**:
- Full sync on first install
- Incremental daily sync (when connected)
- Manual sync option

**Sync Process**: Check timestamp → Fetch updates → Download models → Upload queued queries → Update local DB

**Conflict Resolution**: Last-write-wins for profiles, merge favorites, append query history

## 8. Localization

**Content Storage**: Multilingual JSON with language codes (en, te, hi, ta, kn)

**UI**: Flutter intl package with .arb translation files

**Voice**: Native TTS voices, gender-neutral, clear pronunciation

## 9. Performance Optimization

**Mobile**: Lazy loading, image caching, pagination, background sync, <50MB app size

**Backend**: Query optimization, Redis caching, connection pooling, async processing, CDN

**Voice**: Audio compression, streaming STT, TTS caching, parallel batch processing

## 10. Testing Strategy

**Unit Tests**: 80% coverage (flutter_test, pytest)

**Integration Tests**: End-to-end flows, API testing (Postman/Newman)

**Voice Tests**: Sample queries in all languages, WER < 15%, satisfaction > 85%

**Performance Tests**: Load (10K concurrent users), stress, endurance (24h), spike testing (JMeter, Locust)

## 11. Disaster Recovery

**Backups**:
- Database: Daily, 30-day retention, separate region
- Voice recordings: Weekly, 90-day retention, encrypted

**Recovery**: RTO 4 hours, RPO 24 hours
Steps: Identify failure → Switch to backup region → Restore DB → Verify → Resume → Notify

## 12. Development Timeline

**Phase 1 (Weeks 1-4)**: Setup, database design, API structure, app skeleton

**Phase 2 (Weeks 5-8)**: Voice service, scheme service, mobile UI, offline functionality

**Phase 3 (Weeks 9-12)**: WhatsApp/IVR integration, admin dashboard, testing

**Phase 4 (Weeks 13-16)**: Optimization, security audit, UAT, deployment, documentation

## 13. Success Metrics

**Technical**: Voice accuracy >85%, Response time <3s, Uptime >99.5%, Crash rate <1%

**User**: 1,000+ active users (month 1), >95% satisfaction, >5 min avg session, >60% return rate

**Business**: 5,000+ interactions, 5+ districts coverage, balanced language distribution, <₹10/user/month

---

**Version**: 1.0 | **Date**: February 14, 2026

