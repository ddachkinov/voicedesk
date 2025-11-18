# System Architecture

## Overview

The Virtual Receptionist system follows a microservices-based, cloud-native architecture designed for scalability, reliability, and maintainability. The system is composed of several key components that work together to provide seamless voice-based booking services.

## High-Level Architecture

```mermaid
graph TB
    subgraph "External Users"
        Caller[📞 Caller/Tourist]
        Staff[👤 Hotel Staff]
        Microinvest[🏨 Microinvest Хотел Pro]
    end

    subgraph "Telephony Layer"
        Phone[☎️ Phone Number]
        Twilio[📡 Twilio/VoIP Provider]
    end

    subgraph "AI Services Layer"
        STT[🎤 Speech-to-Text<br/>Bulgarian]
        TTS[🔊 Text-to-Speech<br/>Bulgarian]
        LLM[🤖 LLM<br/>GPT-4/Claude]
    end

    subgraph "Application Layer"
        API[⚙️ REST API Server<br/>Node.js/NestJS]
        CallEngine[📞 Call Engine<br/>Conversation Manager]
        BookingEngine[📅 Booking Engine<br/>Availability & Reservations]
        Analytics[📊 Analytics Engine<br/>Reporting & Insights]
    end

    subgraph "Data Layer"
        DB[(🗄️ PostgreSQL<br/>Main Database)]
        Cache[(⚡ Redis<br/>Cache & Sessions)]
        Storage[💾 S3/Object Storage<br/>Call Recordings]
    end

    subgraph "Frontend Layer"
        Web[🖥️ Web Dashboard<br/>Next.js/React]
    end

    Caller -->|Call| Phone
    Phone --> Twilio
    Twilio -->|Webhook| CallEngine
    CallEngine <--> STT
    CallEngine <--> TTS
    CallEngine <--> LLM
    CallEngine --> API
    BookingEngine --> API
    Analytics --> API

    API <--> DB
    API <--> Cache
    CallEngine --> Storage

    Staff --> Web
    Web --> API

    API <-->|API/Sync| Microinvest

    style Caller fill:#e1f5ff
    style Staff fill:#e1f5ff
    style Microinvest fill:#ffe1e1
    style LLM fill:#ffe1f5
    style DB fill:#f5ffe1
    style Cache fill:#f5ffe1
    style Storage fill:#f5ffe1
```

## Component Architecture

### 1. Telephony Layer

```mermaid
graph LR
    subgraph "Inbound Call Flow"
        A[Caller Dials] --> B[Bulgarian Phone Number]
        B --> C[VoIP Provider]
        C --> D{Call Type}
        D -->|New Call| E[Call Engine]
        D -->|Transfer| F[Human Staff]
    end

    subgraph "Outbound"
        G[System] --> H[Confirmation Calls]
        H --> C
    end

    style A fill:#e1f5ff
    style F fill:#ffe1e1
```

**Responsibilities:**
- Route incoming calls to the system
- Provide Bulgarian local/toll-free numbers
- Handle call recording
- Manage call quality and reliability
- Support call transfer to human staff

**Key Technologies:**
- **Primary**: Twilio (international, proven reliability)
- **Alternative**: Vonage, or Bulgarian providers (Vivacom Business, A1 Bulgaria)
- **Protocols**: SIP, WebRTC for voice transmission

### 2. Call Engine (Conversation Manager)

```mermaid
graph TB
    A[Incoming Call] --> B[Initialize Session]
    B --> C[Generate Greeting]
    C --> D[TTS: Play Greeting]
    D --> E[Listen for Response]
    E --> F[STT: Transcribe]
    F --> G[Extract Intent & Entities]
    G --> H{Intent Type?}

    H -->|Availability| I[Query Calendar]
    H -->|Booking| J[Create Reservation]
    H -->|Info| K[Provide Information]
    H -->|Unclear| L[Ask Clarification]
    H -->|Transfer| M[Connect to Staff]

    I --> N[LLM: Generate Response]
    J --> N
    K --> N
    L --> N

    N --> O[TTS: Speak Response]
    O --> P{Call Complete?}
    P -->|No| E
    P -->|Yes| Q[End Call & Save]

    style A fill:#e1f5ff
    style M fill:#ffe1e1
    style Q fill:#e1ffe1
```

**Responsibilities:**
- Manage conversation state and context
- Coordinate between STT, TTS, and LLM services
- Handle interruptions and context switching
- Implement conversation timeout logic
- Log all interactions for analytics

**Core Features:**
- **Session Management**: Maintain conversation context across turns
- **Intent Recognition**: Classify user requests (availability, booking, info, etc.)
- **Entity Extraction**: Extract dates, room types, guest counts, names, contact info
- **Context Awareness**: Remember previous turns in conversation
- **Fallback Handling**: Transfer to human when confidence is low

### 3. AI Services Integration

```mermaid
graph LR
    subgraph "Speech Processing"
        Audio[Audio Stream] --> STT[Speech-to-Text]
        STT --> Text[Transcribed Text]

        Response[Response Text] --> TTS[Text-to-Speech]
        TTS --> AudioOut[Audio Stream]
    end

    subgraph "Language Understanding"
        Text --> LLM[Large Language Model]
        LLM -->|+ Context<br/>+ Calendar Data<br/>+ Booking Rules| Response
    end

    style Audio fill:#e1f5ff
    style AudioOut fill:#e1ffe1
```

**Speech-to-Text (STT) Options:**
1. **Google Cloud Speech-to-Text**
   - Excellent Bulgarian support
   - Real-time streaming
   - Cost: ~$0.006/15 seconds

2. **Azure Speech Services**
   - Good Bulgarian quality
   - Custom model training available
   - Cost: ~$1/hour

3. **OpenAI Whisper API**
   - Excellent accuracy
   - May require optimization for Bulgarian
   - Cost: ~$0.006/minute

**Text-to-Speech (TTS) Options:**
1. **Google Cloud TTS**
   - Natural Bulgarian voices (WaveNet)
   - Multiple voice options
   - Cost: ~$4/1M characters (WaveNet)

2. **Azure Neural TTS**
   - High-quality Bulgarian voices
   - SSML support for control
   - Cost: ~$16/1M characters

3. **ElevenLabs**
   - Most natural voices available
   - Custom voice cloning possible
   - Cost: ~$0.18/1000 characters (higher quality)

**LLM Options:**
1. **OpenAI GPT-4 Turbo**
   - Excellent reasoning and Bulgarian
   - Function calling for booking actions
   - Cost: ~$0.01/1K input tokens, ~$0.03/1K output tokens

2. **Anthropic Claude 3.5 Sonnet**
   - Strong reasoning, excellent instruction following
   - Good Bulgarian support
   - Cost: ~$0.003/1K input tokens, ~$0.015/1K output tokens

3. **Azure OpenAI**
   - Same models with EU data residency
   - GDPR compliant
   - Similar pricing to OpenAI

### 4. Backend Services Architecture

```mermaid
graph TB
    subgraph "API Gateway"
        Gateway[API Gateway<br/>Rate Limiting & Auth]
    end

    subgraph "Microservices"
        CallSvc[Call Service<br/>Conversation Logic]
        BookingSvc[Booking Service<br/>Reservations]
        CalendarSvc[Calendar Service<br/>Availability]
        AnalyticsSvc[Analytics Service<br/>Reports]
        NotifySvc[Notification Service<br/>SMS/Email]
        IntegrationSvc[Integration Service<br/>Microinvest Sync]
    end

    subgraph "Shared Services"
        AuthSvc[Auth Service<br/>JWT/Sessions]
        EventBus[Event Bus<br/>RabbitMQ/Redis]
    end

    Gateway --> CallSvc
    Gateway --> BookingSvc
    Gateway --> CalendarSvc
    Gateway --> AnalyticsSvc

    CallSvc --> EventBus
    BookingSvc --> EventBus
    NotifySvc --> EventBus
    IntegrationSvc --> EventBus

    CallSvc -.->|Auth| AuthSvc
    BookingSvc -.->|Auth| AuthSvc

    style Gateway fill:#ffe1e1
    style EventBus fill:#f5e1ff
```

**Service Breakdown:**

**Call Service**
- Handles conversation orchestration
- Manages AI service integration
- Stores call transcripts and metadata
- Implements business logic for call flow

**Booking Service**
- Creates and manages reservations
- Validates booking constraints
- Calculates pricing (if applicable)
- Sends confirmations

**Calendar Service**
- Manages room availability
- Handles date range queries
- Syncs with Microinvest
- Optimizes availability searches

**Analytics Service**
- Aggregates call data
- Generates reports and metrics
- Provides dashboard data
- Exports data in various formats

**Notification Service**
- Sends booking confirmations (SMS/Email)
- Alerts staff of special requests
- Reminder notifications

**Integration Service**
- Syncs with Microinvest Хотел Pro
- Handles data transformations
- Manages sync schedules
- Error handling and retry logic

### 5. Data Architecture

```mermaid
erDiagram
    PROPERTY ||--o{ ROOM : has
    ROOM ||--o{ BOOKING : receives
    BOOKING ||--o{ GUEST : for
    CALL ||--o{ BOOKING : creates
    CALL ||--o{ TRANSCRIPT : contains
    CALL ||--o{ RECORDING : has
    PROPERTY ||--o{ USER : employs

    PROPERTY {
        uuid id PK
        string name
        string address
        json settings
        timestamp created_at
    }

    ROOM {
        uuid id PK
        uuid property_id FK
        string room_number
        string room_type
        int capacity
        decimal price_per_night
        json amenities
    }

    BOOKING {
        uuid id PK
        uuid room_id FK
        uuid guest_id FK
        uuid call_id FK
        date check_in
        date check_out
        int guests
        enum status
        decimal total_price
        json special_requests
        timestamp created_at
    }

    GUEST {
        uuid id PK
        string full_name
        string phone
        string email
        json preferences
        timestamp created_at
    }

    CALL {
        uuid id PK
        uuid property_id FK
        string caller_number
        timestamp started_at
        timestamp ended_at
        int duration_seconds
        enum outcome
        float sentiment_score
        json metadata
    }

    TRANSCRIPT {
        uuid id PK
        uuid call_id FK
        int turn_number
        enum speaker
        text content
        float confidence
        timestamp timestamp
    }

    RECORDING {
        uuid id PK
        uuid call_id FK
        string storage_url
        int duration_seconds
        string format
    }

    USER {
        uuid id PK
        uuid property_id FK
        string email
        string password_hash
        enum role
        timestamp last_login
    }
```

**Database Technology:** PostgreSQL
- **Why**: ACID compliance, JSON support, mature ecosystem
- **Scaling**: Read replicas for analytics queries
- **Backup**: Daily automated backups with point-in-time recovery

**Caching Layer:** Redis
- Session storage for active calls
- Availability cache (TTL: 5 minutes)
- Rate limiting counters
- Real-time analytics aggregations

**Object Storage:** AWS S3 / DigitalOcean Spaces
- Call recordings (audio files)
- Exported reports
- Backup archives
- Retention: 90 days for calls, 7 years for bookings

### 6. Frontend Architecture

```mermaid
graph TB
    subgraph "Web Application"
        App[Next.js App]

        subgraph "Pages"
            Dashboard[Dashboard<br/>Overview & Metrics]
            Calendar[Calendar<br/>Room View]
            Calls[Call History<br/>Transcripts]
            Bookings[Bookings<br/>Management]
            Analytics[Analytics<br/>Reports]
            Settings[Settings<br/>Configuration]
        end

        subgraph "State Management"
            Store[Redux/Zustand Store]
            API_Client[API Client<br/>Axios/Fetch]
        end

        subgraph "Real-time"
            WS[WebSocket<br/>Live Updates]
        end
    end

    App --> Dashboard
    App --> Calendar
    App --> Calls
    App --> Bookings
    App --> Analytics
    App --> Settings

    Dashboard --> Store
    Calendar --> Store
    Calls --> Store

    Store <--> API_Client
    WS --> Store

    API_Client <--> Backend[Backend API]
    WS <--> Backend

    style App fill:#e1f5ff
    style Backend fill:#ffe1e1
```

**Technology Stack:**
- **Framework**: Next.js 14 (React)
- **UI Components**: shadcn/ui or Material-UI
- **State Management**: Zustand or Redux Toolkit
- **API Client**: Axios with React Query for caching
- **Real-time**: Socket.io or WebSocket for live call updates
- **Charts**: Recharts or Chart.js for analytics
- **Calendar**: FullCalendar or react-big-calendar

## Network Architecture

```mermaid
graph TB
    subgraph "Public Internet"
        Users[Users/Callers]
        Staff[Hotel Staff]
    end

    subgraph "CDN Layer"
        CloudFlare[CloudFlare CDN<br/>Static Assets & DDoS Protection]
    end

    subgraph "Application Layer - DMZ"
        LB[Load Balancer<br/>SSL Termination]
        Web[Web Servers<br/>Next.js]
        API[API Servers<br/>NestJS]
    end

    subgraph "Service Layer - Private Network"
        Services[Microservices<br/>Internal APIs]
        Workers[Background Workers<br/>Job Processing]
    end

    subgraph "Data Layer - Private Network"
        DB[(Primary Database<br/>PostgreSQL)]
        Replica[(Read Replica<br/>PostgreSQL)]
        Cache[(Redis Cluster)]
    end

    subgraph "External Services"
        Twilio[Twilio API]
        OpenAI[OpenAI/Anthropic]
        Google[Google Cloud<br/>STT/TTS]
    end

    Users -->|HTTPS| CloudFlare
    Staff -->|HTTPS| CloudFlare
    CloudFlare --> LB

    LB --> Web
    LB --> API

    Web --> API
    API --> Services
    Services --> Workers

    API --> DB
    API --> Cache
    Services --> DB
    Services --> Replica
    Services --> Cache

    Services <-->|HTTPS| Twilio
    Services <-->|HTTPS| OpenAI
    Services <-->|HTTPS| Google

    style Users fill:#e1f5ff
    style Staff fill:#e1f5ff
    style CloudFlare fill:#ffe1e1
    style DB fill:#f5ffe1
    style Cache fill:#f5ffe1
```

## Deployment Architecture

### Development Environment
```
- Local development with Docker Compose
- Mock AI services for cost efficiency
- SQLite or PostgreSQL in container
- Hot reload for rapid development
```

### Staging Environment
```
- Kubernetes cluster (3 nodes)
- Actual AI services with quota limits
- PostgreSQL managed database
- Subset of production data
- Same infrastructure as production
```

### Production Environment
```
- Kubernetes cluster (5+ nodes, auto-scaling)
- Multi-region for high availability
- Managed PostgreSQL with replication
- Redis cluster (3+ nodes)
- Full monitoring and alerting
- Automatic backups
```

## Security Architecture

```mermaid
graph TB
    subgraph "Security Layers"
        Edge[Edge Security<br/>CloudFlare WAF<br/>DDoS Protection]

        API_Auth[API Authentication<br/>JWT/API Keys]

        Network[Network Security<br/>Private Subnets<br/>Security Groups]

        Data[Data Security<br/>Encryption at Rest<br/>Encryption in Transit]

        App[Application Security<br/>Input Validation<br/>OWASP Compliance]

        Audit[Audit Logging<br/>Access Logs<br/>Change Tracking]
    end

    Edge --> API_Auth
    API_Auth --> Network
    Network --> App
    App --> Data
    Data --> Audit

    style Edge fill:#ffe1e1
    style Data fill:#f5ffe1
```

**Security Measures:**
1. **Authentication**: JWT tokens with refresh mechanism
2. **Authorization**: Role-based access control (RBAC)
3. **Encryption**: TLS 1.3 for transit, AES-256 for rest
4. **Data Privacy**: GDPR compliance, data minimization
5. **Audit Trail**: Complete logging of all data access
6. **Secrets Management**: HashiCorp Vault or AWS Secrets Manager
7. **Network Isolation**: Private subnets for databases and services

## Scalability Considerations

### Horizontal Scaling
- **API Servers**: Auto-scale based on CPU/memory (2-20 instances)
- **Worker Nodes**: Scale based on queue depth
- **Database**: Read replicas for analytics queries

### Vertical Scaling
- **Database**: Upgrade instance size as data grows
- **Cache**: Increase Redis memory for larger datasets

### Performance Optimizations
1. **Caching Strategy**:
   - Room availability: 5-minute TTL
   - Static data: 1-hour TTL
   - User sessions: Redis

2. **Database Indexing**:
   - Composite indexes on (property_id, date) for availability queries
   - Full-text search indexes on transcripts

3. **CDN Usage**:
   - Static assets served from edge locations
   - API responses cached when possible

4. **Connection Pooling**:
   - Database connection pools (min: 10, max: 100)
   - HTTP keep-alive for external APIs

## Monitoring & Observability

```mermaid
graph LR
    subgraph "Application"
        App[Application<br/>Services]
    end

    subgraph "Monitoring Stack"
        Metrics[Prometheus<br/>Metrics Collection]
        Logs[Loki/ELK<br/>Log Aggregation]
        Traces[Jaeger<br/>Distributed Tracing]
    end

    subgraph "Visualization"
        Grafana[Grafana<br/>Dashboards]
        Alerts[AlertManager<br/>Notifications]
    end

    App --> Metrics
    App --> Logs
    App --> Traces

    Metrics --> Grafana
    Logs --> Grafana
    Traces --> Grafana

    Metrics --> Alerts
    Logs --> Alerts

    style App fill:#e1f5ff
    style Grafana fill:#e1ffe1
```

**Key Metrics:**
- Call volume and success rate
- Response time (P50, P95, P99)
- AI service latency
- Database query performance
- Error rates by endpoint
- Cost tracking per call

**Alerting:**
- System downtime (>1 minute)
- High error rate (>5%)
- Slow response times (>3 seconds)
- Database connection pool exhaustion
- High AI service costs

## Disaster Recovery

**Backup Strategy:**
- **Database**: Continuous backup with 35-day retention
- **Point-in-time Recovery**: Up to 7 days
- **Call Recordings**: Replicated across regions
- **Configuration**: Version controlled in Git

**Recovery Objectives:**
- **RTO (Recovery Time Objective)**: 4 hours
- **RPO (Recovery Point Objective)**: 1 hour

**Failover Plan:**
1. Detect failure via monitoring
2. Promote read replica to primary
3. Update DNS/load balancer
4. Verify system health
5. Investigate root cause

## Technology Decisions Summary

| Component | Technology | Rationale |
|-----------|------------|-----------|
| **Backend** | Node.js/NestJS | TypeScript, async I/O, rich ecosystem |
| **Database** | PostgreSQL | ACID compliance, JSON support, mature |
| **Cache** | Redis | Fast, versatile, pub/sub support |
| **Frontend** | Next.js | SSR, excellent DX, React ecosystem |
| **Telephony** | Twilio | Reliable, well-documented, global reach |
| **STT** | Google Cloud Speech | Best Bulgarian support |
| **TTS** | Google Cloud TTS | Natural Bulgarian voices |
| **LLM** | OpenAI GPT-4 | Best reasoning, function calling |
| **Container** | Docker | Standard, portable |
| **Orchestration** | Kubernetes | Scalable, self-healing, industry standard |
| **Monitoring** | Prometheus + Grafana | Open source, powerful, flexible |

## Next Steps

1. Review [Technology Stack](./03-technology-stack.md) for detailed technology analysis
2. See [API Specifications](./04-api-specifications.md) for endpoint definitions
3. Check [Database Schema](./05-database-schema.md) for complete data models
4. Review [Security & Compliance](./09-security-compliance.md) for security details
