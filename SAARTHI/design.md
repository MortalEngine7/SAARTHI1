# Design Document: AI Content Engine for Bharat

## 1. System Overview

### 1.1 Architecture Philosophy
The AI Content Engine follows a microservices architecture with cloud-native design principles, optimized for scalability, reliability, and cost-effectiveness. The system prioritizes mobile-first design and low-bandwidth optimization for Indian market conditions.

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Web App      │  │ Mobile App   │  │ Progressive  │      │
│  │ (React)      │  │ (React Native│  │ Web App      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway Layer                        │
│              (Kong / AWS API Gateway)                        │
│  - Authentication  - Rate Limiting  - Load Balancing        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Application Services                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │ Content  │ │Personal- │ │Content   │ │Distribu- │      │
│  │Generator │ │ization   │ │Manager   │ │tion      │      │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│  │Moderation│ │Analytics │ │User      │                   │
│  │Service   │ │Service   │ │Service   │                   │
│  └──────────┘ └──────────┘ └──────────┘                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      AI/ML Layer                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │Text Gen  │ │Image Gen │ │Recommend │ │Moderation│      │
│  │Models    │ │Models    │ │Engine    │ │Models    │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │PostgreSQL│ │MongoDB   │ │Redis     │ │S3/Object │      │
│  │(Metadata)│ │(Content) │ │(Cache)   │ │Storage   │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────┘
```

## 2. Component Design

### 2.1 Content Generator Service

**Purpose:** Handle all AI-powered content generation requests

**Key Components:**

- Text Generation Module
- Image Generation Module
- Translation Module
- Content Enhancement Module

**Technology Stack:**
- Runtime: Python 3.11+ with FastAPI
- AI Models:
  - Text: GPT-J-6B, BLOOM-7B (multilingual), IndicBERT
  - Translation: IndicTrans2, mBART-50
  - Enhancement: T5-base for paraphrasing
- Model Serving: vLLM or TorchServe
- Queue: RabbitMQ for async processing

**API Endpoints:**
```
POST /api/v1/generate/text
POST /api/v1/generate/image
POST /api/v1/translate
POST /api/v1/enhance
GET  /api/v1/generate/status/{job_id}
```

**Data Flow:**
1. Client sends generation request with prompt and parameters
2. Request validated and queued
3. Job ID returned immediately to client
4. Worker picks up job from queue
5. AI model processes request
6. Result stored in object storage
7. Metadata saved to database
8. Client polls or receives webhook notification

### 2.2 Personalization Service

**Purpose:** Provide personalized content recommendations and user profiling

**Key Components:**
- User Profile Manager
- Recommendation Engine
- Behavior Tracker
- Feedback Processor

**Technology Stack:**
- Runtime: Python 3.11+ with FastAPI
- ML Framework: PyTorch, Scikit-learn
- Recommendation: Surprise library, custom collaborative filtering
- Feature Store: Redis for real-time features
- Batch Processing: Apache Spark (optional for scale)

**Algorithms:**
- Collaborative Filtering (User-User, Item-Item)
- Content-Based Filtering using embeddings
- Hybrid approach combining both
- Cold start: Popularity-based + demographic filtering

**API Endpoints:**
```
GET  /api/v1/recommendations
POST /api/v1/profile/preferences
GET  /api/v1/profile
POST /api/v1/feedback
```

### 2.3 Content Manager Service

**Purpose:** Organize, search, and manage user content

**Key Components:**
- Content Repository
- Search Engine
- Tagging System
- Version Control

**Technology Stack:**
- Runtime: Node.js with Express or Python FastAPI
- Database: MongoDB for flexible content schema
- Search: Elasticsearch with multilingual analyzers
- Tagging: spaCy for NLP-based auto-tagging
- Storage: AWS S3 / Google Cloud Storage / MinIO

**API Endpoints:**
```
POST   /api/v1/content
GET    /api/v1/content/{id}
PUT    /api/v1/content/{id}
DELETE /api/v1/content/{id}
GET    /api/v1/content/search
POST   /api/v1/content/tag
GET    /api/v1/content/versions/{id}
```

**Search Features:**
- Full-text search with language-specific analyzers
- Fuzzy matching for typo tolerance
- Filters: date, language, tags, format
- Voice search using Web Speech API

### 2.4 Distribution Service

**Purpose:** Optimize and automate content distribution across platforms

**Key Components:**
- Platform Integrations
- Scheduling Engine
- Content Adapter
- Analytics Collector

**Technology Stack:**
- Runtime: Node.js with Express
- Scheduler: Bull queue with Redis
- Platform SDKs: Official APIs for social platforms
- Analytics: Custom aggregation + platform APIs

**Supported Platforms:**
- Facebook Graph API
- Instagram Graph API
- Twitter API v2
- LinkedIn API
- YouTube Data API
- WhatsApp Business API

**API Endpoints:**
```
POST /api/v1/publish
POST /api/v1/schedule
GET  /api/v1/schedule/optimal-time
GET  /api/v1/analytics/{content_id}
DELETE /api/v1/schedule/{id}
```

**Optimal Time Algorithm:**
1. Collect historical engagement data per platform
2. Analyze user's audience activity patterns
3. Use time-series analysis to identify peaks
4. Consider platform-specific best practices
5. Return top 3 recommended time slots

### 2.5 Moderation Service

**Purpose:** Ensure content safety and cultural appropriateness

**Key Components:**
- Content Scanner
- Toxicity Detector
- Cultural Sensitivity Checker
- Copyright Detector

**Technology Stack:**
- Runtime: Python 3.11+ with FastAPI
- Models:
  - Toxicity: Detoxify, IndicBERT fine-tuned
  - NSFW: CLIP-based image classifier
  - Hate Speech: Custom multilingual classifier
- Rules Engine: Python-based rule system

**API Endpoints:**
```
POST /api/v1/moderate/text
POST /api/v1/moderate/image
GET  /api/v1/moderate/report/{id}
```

**Moderation Pipeline:**
1. Content submitted for moderation
2. Parallel checks: toxicity, hate speech, NSFW, cultural sensitivity
3. Aggregate scores and flags
4. Apply threshold rules
5. Return moderation result with confidence scores
6. Log flagged content for review

### 2.6 User Service

**Purpose:** Handle authentication, authorization, and user management

**Technology Stack:**
- Runtime: Node.js with Express or Go
- Database: PostgreSQL for user data
- Auth: JWT tokens, OAuth 2.0
- Password: bcrypt hashing

**API Endpoints:**
```
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/logout
GET  /api/v1/user/profile
PUT  /api/v1/user/profile
```

### 2.7 Analytics Service

**Purpose:** Track and analyze platform usage and content performance

**Technology Stack:**
- Runtime: Python with FastAPI
- Time-series DB: InfluxDB or TimescaleDB
- Visualization: Grafana
- Processing: Apache Kafka for event streaming

**Metrics Tracked:**
- User engagement (sessions, duration, actions)
- Content performance (views, shares, engagement rate)
- System performance (latency, errors, throughput)
- Business metrics (conversions, revenue)

## 3. Data Models

### 3.1 User Schema (PostgreSQL)
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    phone VARCHAR(20),
    language_preference VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    subscription_tier VARCHAR(50) DEFAULT 'free'
);

CREATE TABLE user_preferences (
    user_id UUID REFERENCES users(id),
    interests TEXT[],
    content_types TEXT[],
    platforms TEXT[],
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### 3.2 Content Schema (MongoDB)
```javascript
{
  _id: ObjectId,
  user_id: UUID,
  title: String,
  content_type: String, // 'text', 'image', 'mixed'
  language: String,
  body: {
    text: String,
    images: [String], // URLs
    metadata: Object
  },
  tags: [String],
  auto_tags: [String],
  status: String, // 'draft', 'published', 'archived'
  platforms: [{
    name: String,
    published_at: Date,
    post_id: String,
    analytics: Object
  }],
  versions: [{
    version: Number,
    body: Object,
    created_at: Date
  }],
  created_at: Date,
  updated_at: Date
}
```

### 3.3 Generation Job Schema (MongoDB)
```javascript
{
  _id: ObjectId,
  job_id: String,
  user_id: UUID,
  type: String, // 'text', 'image', 'translation'
  status: String, // 'pending', 'processing', 'completed', 'failed'
  input: {
    prompt: String,
    parameters: Object
  },
  output: {
    result: String, // URL or text
    metadata: Object
  },
  error: String,
  created_at: Date,
  completed_at: Date,
  processing_time_ms: Number
}
```

### 3.4 User Behavior Schema (InfluxDB)
```
measurement: user_events
tags: user_id, event_type, content_id, platform
fields: duration, engagement_score, timestamp
```

## 4. AI Models Selection

### 4.1 Text Generation Models

**Primary: BLOOM-7B1**
- Multilingual support (46 languages including Indian languages)
- Open-source (BigScience RAIL License)
- Reasonable size for deployment
- Good quality for general content

**Alternative: GPT-J-6B**
- English content generation
- Apache 2.0 license
- Faster inference

**Indian Language Specific: IndicBERT**
- 12 Indian languages
- Fine-tuned for Indian context
- Use for language-specific tasks

### 4.2 Translation Models

**Primary: IndicTrans2**
- Covers 22 Indian languages
- State-of-art for Indian language translation
- Open-source (MIT License)
- Optimized for Indian language pairs

**Fallback: mBART-50**
- 50 languages including major Indian languages
- Good for less common language pairs

### 4.3 Image Generation Models

**Primary: Stable Diffusion v2.1**
- Open-source (CreativeML Open RAIL++-M License)
- High-quality image generation
- Can be fine-tuned for Indian contexts
- Reasonable compute requirements

**Optimization: Stable Diffusion XL Turbo**
- Faster inference (1-4 steps)
- Lower latency for better UX

### 4.4 Recommendation Models

**Collaborative Filtering:**
- Matrix Factorization (SVD)
- Neural Collaborative Filtering

**Content-Based:**
- Sentence-BERT for content embeddings
- Cosine similarity for matching

### 4.5 Moderation Models

**Toxicity Detection:**
- Detoxify (multilingual)
- Custom fine-tuned IndicBERT for Indian languages

**NSFW Detection:**
- CLIP-based classifier
- Custom dataset with Indian context

**Hate Speech:**
- HateXplain model
- Fine-tuned for Indian languages and cultural context

## 5. Technology Stack Summary

### 5.1 Frontend
- **Web App:** React 18+ with TypeScript
- **Mobile App:** React Native with TypeScript
- **State Management:** Redux Toolkit or Zustand
- **UI Framework:** Material-UI or Chakra UI
- **PWA:** Workbox for offline support

### 5.2 Backend
- **API Services:** Python FastAPI, Node.js Express
- **API Gateway:** Kong or AWS API Gateway
- **Authentication:** JWT, OAuth 2.0 (Passport.js)
- **Message Queue:** RabbitMQ or AWS SQS
- **Caching:** Redis

### 5.3 AI/ML Infrastructure
- **Model Serving:** vLLM, TorchServe, or TensorFlow Serving
- **ML Framework:** PyTorch, Transformers (Hugging Face)
- **Model Optimization:** ONNX Runtime, TensorRT
- **GPU:** NVIDIA T4 or A10G instances

### 5.4 Data Storage
- **Relational:** PostgreSQL 15+
- **Document:** MongoDB 6+
- **Cache:** Redis 7+
- **Object Storage:** AWS S3 / Google Cloud Storage / MinIO
- **Search:** Elasticsearch 8+
- **Time-series:** InfluxDB or TimescaleDB

### 5.5 DevOps & Infrastructure
- **Cloud Provider:** AWS / Google Cloud / Azure
- **Container:** Docker
- **Orchestration:** Kubernetes (EKS/GKE/AKS) or Docker Compose
- **CI/CD:** GitHub Actions or GitLab CI
- **Monitoring:** Prometheus + Grafana
- **Logging:** ELK Stack (Elasticsearch, Logstash, Kibana)
- **Error Tracking:** Sentry

### 5.6 CDN & Performance
- **CDN:** Cloudflare or AWS CloudFront
- **Image Optimization:** Sharp.js, ImageMagick
- **Compression:** Brotli, Gzip

## 6. API Structure

### 6.1 API Design Principles
- RESTful architecture
- Versioned APIs (/api/v1/)
- JSON request/response format
- Consistent error handling
- Rate limiting per user tier
- Pagination for list endpoints

### 6.2 Authentication Flow
```
1. User registers/logs in
2. Server returns JWT access token (15 min) + refresh token (7 days)
3. Client includes access token in Authorization header
4. Server validates token on each request
5. Client refreshes token when expired
```

### 6.3 Rate Limiting
```
Free Tier:
- 100 requests/hour
- 10 content generations/day
- 5 image generations/day

Paid Tier:
- 1000 requests/hour
- 100 content generations/day
- 50 image generations/day
```

### 6.4 Error Response Format
```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Prompt cannot be empty",
    "details": {
      "field": "prompt",
      "constraint": "required"
    },
    "request_id": "req_123456"
  }
}
```

### 6.5 Pagination Format
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8,
    "has_next": true,
    "has_prev": false
  }
}
```

## 7. Deployment Architecture

### 7.1 Environment Strategy
- **Development:** Local Docker Compose
- **Staging:** Kubernetes cluster (smaller instances)
- **Production:** Kubernetes cluster (auto-scaling)

### 7.2 Kubernetes Deployment

**Namespaces:**
- `api-services`: Application microservices
- `ai-ml`: AI model serving
- `data`: Databases and caches
- `monitoring`: Observability stack

**Resource Allocation (Production):**
```yaml
Content Generator Service:
  replicas: 3-10 (auto-scale)
  cpu: 2 cores
  memory: 4Gi
  gpu: 1x NVIDIA T4 (for AI inference)

Personalization Service:
  replicas: 2-5
  cpu: 1 core
  memory: 2Gi

Content Manager Service:
  replicas: 2-5
  cpu: 1 core
  memory: 2Gi

Distribution Service:
  replicas: 2-4
  cpu: 1 core
  memory: 2Gi

Moderation Service:
  replicas: 2-4
  cpu: 2 cores
  memory: 4Gi
  gpu: 1x NVIDIA T4

User Service:
  replicas: 2-4
  cpu: 1 core
  memory: 1Gi
```

### 7.3 Database Deployment
- **PostgreSQL:** Managed service (AWS RDS, Google Cloud SQL) or self-hosted with replication
- **MongoDB:** MongoDB Atlas or self-hosted replica set
- **Redis:** AWS ElastiCache or self-hosted cluster
- **Elasticsearch:** Elastic Cloud or self-hosted cluster

### 7.4 Auto-Scaling Strategy
```yaml
Horizontal Pod Autoscaler:
  - CPU threshold: 70%
  - Memory threshold: 80%
  - Custom metrics: Queue depth, request latency
  
Cluster Autoscaler:
  - Node scaling based on pending pods
  - GPU nodes on-demand for AI workloads
```

### 7.5 Cost Optimization
- Use spot instances for non-critical workloads
- Scale down during off-peak hours (2 AM - 6 AM IST)
- Cache frequently accessed content
- Compress and optimize images
- Use CDN for static assets
- Implement request batching for AI models

## 8. Security Architecture

### 8.1 Network Security
- All services behind API Gateway
- Private subnets for databases
- VPC peering for service communication
- WAF rules for common attacks
- DDoS protection via Cloudflare

### 8.2 Data Security
- Encryption at rest (AES-256)
- Encryption in transit (TLS 1.3)
- Database encryption enabled
- Secrets management (AWS Secrets Manager / HashiCorp Vault)
- Regular security audits

### 8.3 Application Security
- Input validation and sanitization
- SQL injection prevention (parameterized queries)
- XSS prevention (content escaping)
- CSRF protection
- Rate limiting and throttling
- API key rotation

### 8.4 Compliance
- Data residency in India (Mumbai region)
- GDPR-compliant data handling
- User data export capability
- Right to deletion implementation
- Audit logging for compliance

## 9. Monitoring & Observability

### 9.1 Metrics Collection
- **Application Metrics:** Request rate, latency, error rate
- **Business Metrics:** Content generations, user signups, conversions
- **Infrastructure Metrics:** CPU, memory, disk, network
- **AI Model Metrics:** Inference time, model accuracy, GPU utilization

### 9.2 Logging Strategy
- Structured logging (JSON format)
- Log levels: DEBUG, INFO, WARN, ERROR
- Centralized logging with ELK stack
- Log retention: 30 days
- PII masking in logs

### 9.3 Alerting
- Critical: P0 (immediate response) - Service down, database failure
- High: P1 (15 min response) - High error rate, slow response time
- Medium: P2 (1 hour response) - Elevated resource usage
- Low: P3 (next day) - Non-critical warnings

### 9.4 Dashboards
- System health dashboard
- Business metrics dashboard
- AI model performance dashboard
- User analytics dashboard

## 10. Bandwidth Optimization Strategies

### 10.1 Frontend Optimization
- Lazy loading for images and components
- Code splitting and tree shaking
- Minification and compression (Brotli)
- Service worker for offline caching
- Progressive image loading (blur-up)

### 10.2 API Optimization
- Response compression (Gzip/Brotli)
- Field selection (GraphQL-style)
- Pagination with cursor-based approach
- ETags for cache validation
- WebSocket for real-time updates (reduces polling)

### 10.3 Media Optimization
- Image formats: WebP with JPEG fallback
- Responsive images (srcset)
- Video: Adaptive bitrate streaming
- Thumbnail generation
- CDN for media delivery

### 10.4 Lite Mode
- Disable auto-play videos
- Lower resolution images
- Simplified UI
- Reduced animations
- Text-only mode option

## 11. Disaster Recovery

### 11.1 Backup Strategy
- **Databases:** Daily full backup + continuous incremental
- **Object Storage:** Cross-region replication
- **Configuration:** Version controlled in Git
- **Retention:** 30 days for all backups

### 11.2 Recovery Procedures
- **RTO (Recovery Time Objective):** 4 hours
- **RPO (Recovery Point Objective):** 1 hour
- Automated failover for databases
- Multi-region deployment for critical services
- Regular disaster recovery drills

## 12. Development Workflow

### 12.1 Git Workflow
- Main branch: Production-ready code
- Develop branch: Integration branch
- Feature branches: feature/feature-name
- Hotfix branches: hotfix/issue-description
- Pull request reviews required

### 12.2 CI/CD Pipeline
```
1. Code commit → GitHub
2. Automated tests run (unit, integration)
3. Code quality checks (linting, security scan)
4. Build Docker images
5. Push to container registry
6. Deploy to staging (auto)
7. Run E2E tests
8. Manual approval for production
9. Deploy to production (blue-green)
10. Health checks and monitoring
```

### 12.3 Testing Strategy
- **Unit Tests:** 80% code coverage target
- **Integration Tests:** API endpoint testing
- **E2E Tests:** Critical user flows
- **Load Tests:** Performance benchmarking
- **Security Tests:** OWASP Top 10 checks

## 13. Phased Rollout Plan

### Phase 1: MVP (Months 1-3)
- User authentication and profile management
- Basic text generation (English + Hindi)
- Simple content management
- Single platform publishing (Facebook)
- Basic moderation

### Phase 2: Core Features (Months 4-6)
- Multi-language support (10 languages)
- Image generation
- Personalization engine
- Multi-platform distribution
- Advanced moderation
- Analytics dashboard

### Phase 3: Scale & Optimize (Months 7-9)
- Performance optimization
- Bandwidth optimization
- Advanced scheduling
- Collaboration features
- Mobile app launch
- Payment integration

### Phase 4: Advanced Features (Months 10-12)
- Voice search
- Advanced analytics
- API access
- Template marketplace
- Enterprise features

## 14. Success Criteria

### 14.1 Technical Success
- 99.5% uptime achieved
- < 3 second page load time on 3G
- < 5 second content generation time
- Support 100K concurrent users

### 14.2 Business Success
- 50K registered users in 6 months
- 20% MAU rate
- 10% free-to-paid conversion
- 4+ star user rating

### 14.3 User Success
- 80% user satisfaction with content quality
- 60% of generated content published
- 40% user retention after 3 months
- NPS > 40

---

## Appendix A: Cost Estimation (Monthly)

### Infrastructure Costs (AWS Mumbai Region)
- **Compute (EKS):** $800-1500
- **GPU Instances (AI):** $1200-2000
- **Databases (RDS, MongoDB Atlas):** $400-800
- **Storage (S3):** $100-300
- **CDN (CloudFront):** $200-500
- **Monitoring & Logging:** $100-200
- **Total:** $2800-5300/month

### Scaling Projections
- 10K users: ~$3000/month
- 50K users: ~$8000/month
- 100K users: ~$15000/month

## Appendix B: Open Source Alternatives

To minimize costs, consider:
- **Cloud:** DigitalOcean, Linode (cheaper than AWS)
- **Database:** Self-hosted PostgreSQL, MongoDB
- **Object Storage:** MinIO (self-hosted S3 alternative)
- **CDN:** Cloudflare (free tier generous)
- **Monitoring:** Self-hosted Prometheus + Grafana

---

**Document Version:** 1.0  
**Last Updated:** February 15, 2026  
**Status:** Draft for Review
