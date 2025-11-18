# Technology Stack Recommendations

## Overview

This document provides detailed analysis and justification for each technology choice in the Virtual Receptionist system, with special consideration for the Bulgarian market, development complexity, cost, and scalability.

## Technology Selection Criteria

Each technology was evaluated based on:
1. **Bulgarian Language Support**: Quality of Bulgarian language processing
2. **Cost Efficiency**: Especially important for SMB hotel market
3. **Development Speed**: Time to market considerations
4. **Scalability**: Ability to grow with customer base
5. **Reliability**: Uptime and stability requirements
6. **Local Availability**: Services available in Bulgaria/EU
7. **GDPR Compliance**: EU data residency and privacy
8. **Developer Experience**: Ease of development and debugging

---

## Backend Stack

### Runtime: Node.js 20 LTS

**Choice**: Node.js with TypeScript

**Justification**:
- ✅ **Async I/O**: Perfect for handling concurrent calls and API requests
- ✅ **Rich Ecosystem**: Extensive libraries for telephony, AI, and web services
- ✅ **TypeScript**: Type safety reduces bugs, improves maintainability
- ✅ **Developer Availability**: Large talent pool in Bulgaria
- ✅ **Performance**: V8 engine provides excellent performance for I/O-bound tasks
- ✅ **Unified Stack**: Share types and code between frontend and backend

**Alternatives Considered**:
- **Python/FastAPI**: Excellent for AI/ML, but Node.js better for real-time telephony
- **Go**: Great performance, but smaller ecosystem for AI integrations
- **.NET/C#**: Good choice, but higher hosting costs and less flexible

### Framework: NestJS

**Choice**: NestJS 10+

**Justification**:
- ✅ **Enterprise Architecture**: Built-in support for microservices, modules, DI
- ✅ **TypeScript-First**: Native TypeScript support with decorators
- ✅ **Scalability**: Easy to break into microservices as system grows
- ✅ **Testing**: Excellent testing utilities and patterns
- ✅ **Documentation**: Auto-generated API docs with Swagger
- ✅ **Community**: Large ecosystem and active development

**Example Structure**:
```typescript
// src/modules/call/call.controller.ts
@Controller('calls')
export class CallController {
  constructor(
    private readonly callService: CallService,
    private readonly aiService: AiService,
  ) {}

  @Post('webhook/incoming')
  async handleIncomingCall(
    @Body() callData: TwilioWebhookDto
  ): Promise<TwiMLResponse> {
    const session = await this.callService.initializeSession(callData);
    const greeting = await this.aiService.generateGreeting(session);
    return this.buildTwiMLResponse(greeting);
  }
}
```

**Alternative**:
- **Express.js**: Lighter but requires more manual setup
- **Fastify**: Faster but less mature ecosystem

### API Architecture: RESTful + WebSocket

**Choice**: REST for CRUD operations, WebSocket for real-time updates

**Justification**:
- ✅ **REST**: Simple, well-understood, easy to document and consume
- ✅ **WebSocket**: Real-time call status updates without polling
- ✅ **Hybrid Approach**: Use the right tool for each use case

**Endpoints Pattern**:
```
GET    /api/v1/calls                    # List calls
GET    /api/v1/calls/:id                # Get call details
POST   /api/v1/bookings                 # Create booking
GET    /api/v1/calendar/availability    # Check availability
WS     /ws/calls                        # Real-time call updates
```

**Alternative**:
- **GraphQL**: More flexible but adds complexity for this use case
- **gRPC**: Better performance but harder to debug and less frontend-friendly

---

## Database & Storage

### Primary Database: PostgreSQL 16

**Choice**: PostgreSQL (managed service)

**Justification**:
- ✅ **ACID Compliance**: Critical for booking transactions
- ✅ **JSON Support**: Flexible schema for call metadata and settings
- ✅ **Full-Text Search**: Search through transcripts
- ✅ **Mature**: Battle-tested, excellent reliability
- ✅ **Performance**: Excellent query optimizer and indexing
- ✅ **Extensions**: PostGIS (if location features needed), pg_cron for scheduling

**Provider Options**:

| Provider | Location | Cost (2 CPU, 4GB RAM) | Pros | Cons |
|----------|----------|----------------------|------|------|
| **DigitalOcean Managed DB** | Frankfurt | $60/month | Simple, affordable, EU location | Limited scaling options |
| **AWS RDS** | Frankfurt | $85/month | Advanced features, auto-scaling | More complex, higher cost |
| **Azure Database** | Netherlands | $90/month | Good EU presence, GDPR tools | Microsoft ecosystem lock-in |
| **Supabase** | Frankfurt | $25/month (starter) | Modern DX, integrated auth, real-time | Newer, less proven at scale |

**Recommendation**: Start with DigitalOcean for simplicity and cost, migrate to AWS if scaling needs increase.

**Schema Example**:
```sql
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    property_id UUID NOT NULL REFERENCES properties(id),
    room_id UUID NOT NULL REFERENCES rooms(id),
    guest_id UUID NOT NULL REFERENCES guests(id),
    call_id UUID REFERENCES calls(id),
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    guests INTEGER NOT NULL,
    status booking_status NOT NULL DEFAULT 'pending',
    total_price DECIMAL(10,2),
    special_requests JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    CONSTRAINT valid_dates CHECK (check_out > check_in),
    CONSTRAINT valid_guests CHECK (guests > 0)
);

CREATE INDEX idx_bookings_dates ON bookings(check_in, check_out);
CREATE INDEX idx_bookings_property ON bookings(property_id, status);
CREATE INDEX idx_bookings_special_requests ON bookings USING GIN(special_requests);
```

**Alternatives**:
- **MySQL**: Good but PostgreSQL has better JSON and advanced features
- **MongoDB**: Flexible schema but no ACID transactions (critical for bookings)

### Cache: Redis 7

**Choice**: Redis (managed service or cluster)

**Justification**:
- ✅ **Speed**: Sub-millisecond response times
- ✅ **Versatility**: Cache, session store, pub/sub, rate limiting
- ✅ **Persistence**: Optional persistence for critical data
- ✅ **Data Structures**: Rich data types (sets, sorted sets, hashes)

**Use Cases**:
```typescript
// Availability caching
const cacheKey = `availability:${propertyId}:${dateRange}`;
const cached = await redis.get(cacheKey);
if (cached) return JSON.parse(cached);

// Cache for 5 minutes
await redis.setex(cacheKey, 300, JSON.stringify(availability));

// Session management
await redis.setex(`session:${sessionId}`, 3600, JSON.stringify(sessionData));

// Rate limiting
const key = `ratelimit:${ip}`;
const requests = await redis.incr(key);
if (requests === 1) await redis.expire(key, 60);
if (requests > 100) throw new RateLimitError();
```

**Provider Options**:
- **DigitalOcean Managed Redis**: $15/month (1GB)
- **AWS ElastiCache**: $25/month (similar specs)
- **Upstash**: Serverless, pay-per-request (good for low traffic)

### Object Storage: S3-Compatible

**Choice**: DigitalOcean Spaces or AWS S3

**Justification**:
- ✅ **Scalability**: Unlimited storage growth
- ✅ **Durability**: 99.999999999% durability
- ✅ **Cost-Effective**: ~$5/TB/month
- ✅ **CDN Integration**: Fast global access to recordings
- ✅ **Lifecycle Policies**: Automatic archival/deletion

**Storage Breakdown**:
```
call-recordings/
  ├── 2025/
  │   ├── 11/
  │   │   ├── 18/
  │   │   │   └── {call-id}.mp3
  │
exports/
  ├── analytics/
  │   └── report-2025-11.csv
  │
backups/
  └── db-backup-2025-11-18.sql.gz
```

**Pricing Example** (100 calls/day):
- Storage: ~3GB/day × 90 days = 270GB
- Cost: 270GB × $0.02/GB = $5.40/month
- Bandwidth: ~50GB/month = $1.00/month
- **Total**: ~$6.50/month

---

## Frontend Stack

### Framework: Next.js 14 (App Router)

**Choice**: Next.js with React 18

**Justification**:
- ✅ **Server-Side Rendering**: Better SEO and initial load performance
- ✅ **API Routes**: Backend-for-frontend patterns
- ✅ **TypeScript**: Shared types with backend
- ✅ **File-Based Routing**: Intuitive organization
- ✅ **Image Optimization**: Automatic optimization for assets
- ✅ **Developer Experience**: Hot reload, excellent debugging

**Project Structure**:
```
src/app/
  ├── (auth)/
  │   ├── login/
  │   └── layout.tsx
  ├── (dashboard)/
  │   ├── calendar/
  │   ├── calls/
  │   ├── bookings/
  │   ├── analytics/
  │   └── layout.tsx
  └── api/
      └── [...catch-all]/
          └── route.ts
```

**Alternatives**:
- **Vite + React**: Faster dev server but no SSR out of the box
- **SvelteKit**: Excellent DX but smaller ecosystem
- **Remix**: Good SSR but less mature than Next.js

### UI Component Library: shadcn/ui

**Choice**: shadcn/ui + Tailwind CSS

**Justification**:
- ✅ **Customizable**: Components you own, not a dependency
- ✅ **Modern**: Built on Radix UI primitives
- ✅ **TypeScript**: Full type safety
- ✅ **Accessible**: ARIA compliance out of the box
- ✅ **Tailwind Integration**: Utility-first styling
- ✅ **No Bundle Size**: Only include what you use

**Example Component**:
```tsx
import { Button } from "@/components/ui/button"
import { Calendar } from "@/components/ui/calendar"

export function AvailabilityCalendar() {
  const [date, setDate] = useState<Date | undefined>(new Date())

  return (
    <div className="space-y-4">
      <Calendar
        mode="single"
        selected={date}
        onSelect={setDate}
        className="rounded-md border"
      />
      <Button onClick={() => checkAvailability(date)}>
        Check Availability
      </Button>
    </div>
  )
}
```

**Alternatives**:
- **Material-UI**: More complete but heavier, opinionated design
- **Ant Design**: Enterprise-focused but large bundle size
- **Chakra UI**: Good but shadcn/ui more modern

### State Management: Zustand

**Choice**: Zustand for global state

**Justification**:
- ✅ **Simple**: Minimal boilerplate compared to Redux
- ✅ **TypeScript**: Excellent type inference
- ✅ **Performance**: Only re-renders when needed
- ✅ **DevTools**: Redux DevTools integration
- ✅ **Small**: <1KB gzipped

**Example Store**:
```typescript
import { create } from 'zustand'

interface CallStore {
  activeCalls: Call[]
  addCall: (call: Call) => void
  updateCall: (id: string, updates: Partial<Call>) => void
  removeCall: (id: string) => void
}

export const useCallStore = create<CallStore>((set) => ({
  activeCalls: [],
  addCall: (call) => set((state) => ({
    activeCalls: [...state.activeCalls, call]
  })),
  updateCall: (id, updates) => set((state) => ({
    activeCalls: state.activeCalls.map(call =>
      call.id === id ? { ...call, ...updates } : call
    )
  })),
  removeCall: (id) => set((state) => ({
    activeCalls: state.activeCalls.filter(call => call.id !== id)
  })),
}))
```

**Alternatives**:
- **Redux Toolkit**: More powerful but more complex
- **Jotai**: Atomic state, excellent for large apps
- **React Context**: Built-in but can cause performance issues

### Data Fetching: TanStack Query (React Query)

**Choice**: React Query v5

**Justification**:
- ✅ **Caching**: Automatic caching and invalidation
- ✅ **Background Updates**: Keep data fresh automatically
- ✅ **Optimistic Updates**: Better UX for mutations
- ✅ **Error Handling**: Built-in retry and error states
- ✅ **DevTools**: Excellent debugging tools

**Example Usage**:
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// Fetch calls
export function useCalls(propertyId: string) {
  return useQuery({
    queryKey: ['calls', propertyId],
    queryFn: () => fetchCalls(propertyId),
    refetchInterval: 5000, // Refresh every 5 seconds
  })
}

// Create booking
export function useCreateBooking() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createBooking,
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['bookings'] })
      queryClient.invalidateQueries({ queryKey: ['availability'] })
    },
  })
}
```

### Charts & Visualization: Recharts

**Choice**: Recharts for analytics dashboards

**Justification**:
- ✅ **React-Native**: Built for React, not a wrapper
- ✅ **Composable**: Build complex charts from simple components
- ✅ **Responsive**: Works on all screen sizes
- ✅ **TypeScript**: Full type support

**Example**:
```tsx
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip } from 'recharts'

export function CallVolumeChart({ data }: { data: CallMetrics[] }) {
  return (
    <LineChart width={600} height={300} data={data}>
      <CartesianGrid strokeDasharray="3 3" />
      <XAxis dataKey="date" />
      <YAxis />
      <Tooltip />
      <Line type="monotone" dataKey="calls" stroke="#8884d8" />
      <Line type="monotone" dataKey="bookings" stroke="#82ca9d" />
    </LineChart>
  )
}
```

---

## Telephony & Voice

### Telephony Provider: Twilio

**Choice**: Twilio Voice

**Justification**:
- ✅ **Reliability**: 99.95% uptime SLA
- ✅ **Global Reach**: Bulgarian numbers available
- ✅ **Rich API**: Comprehensive SDKs and documentation
- ✅ **Features**: Recording, transcription, conferencing, SIP
- ✅ **WebHooks**: Real-time call events
- ✅ **Developer Experience**: Excellent documentation and support

**Bulgarian Number Availability**:
- **Local Numbers**: Available in Sofia and major cities
- **Mobile Numbers**: Available
- **Toll-Free**: Not currently available for Bulgaria
- **Recommendation**: Use local Sofia number (+359 2 XXX XXXX)

**Pricing (Bulgaria)**:
```
Phone Number Rental: $1/month
Incoming Calls: $0.0085/minute
Outgoing Calls: $0.0220/minute
Recording Storage: $0.0005/minute

Example Monthly Cost (1000 calls, avg 3 min):
- Number: $1
- Incoming: 3000 min × $0.0085 = $25.50
- Recording: 3000 min × $0.0005 = $1.50
Total: ~$28/month
```

**Integration Example**:
```typescript
import twilio from 'twilio'

const VoiceResponse = twilio.twiml.VoiceResponse

@Post('webhook/incoming')
async handleIncoming(@Body() body: TwilioWebhookDto) {
  const twiml = new VoiceResponse()

  // Start recording
  twiml.record({
    maxLength: 300, // 5 minutes
    transcribe: false, // We'll use our own STT
    recordingStatusCallback: '/webhook/recording-complete',
  })

  // Gather speech input
  const gather = twiml.gather({
    input: ['speech'],
    action: '/webhook/speech',
    language: 'bg-BG',
    speechTimeout: 'auto',
  })

  gather.say(
    { language: 'bg-BG' },
    'Добре дошли в Хотел Слънчев Бряг. Как мога да ви помогна?'
  )

  return twiml.toString()
}
```

**Alternatives for Bulgaria**:

| Provider | Bulgarian Numbers | Pricing | Pros | Cons |
|----------|------------------|---------|------|------|
| **Vonage** | Yes | Similar to Twilio | Good API, reliable | Less documentation |
| **Vivacom Business** | Yes | Lower (~30% less) | Local support, cheaper | Limited API, less features |
| **A1 Bulgaria** | Yes | Competitive | Local provider | Limited international features |
| **Bandwidth** | Limited | Similar to Twilio | Good quality | Limited BG presence |

**Recommendation**: **Twilio** for MVP due to developer experience and features. Consider local providers for cost optimization in Phase 2.

---

## AI Services

### Speech-to-Text: Google Cloud Speech-to-Text

**Choice**: Google Cloud Speech-to-Text v2

**Justification**:
- ✅ **Bulgarian Quality**: Best Bulgarian language support
- ✅ **Streaming**: Real-time transcription
- ✅ **Accuracy**: >95% for clear speech
- ✅ **Custom Models**: Can train on hospitality vocabulary
- ✅ **GDPR Compliant**: EU regions available

**Pricing**:
```
Standard Model: $0.006 per 15 seconds
Enhanced Model: $0.009 per 15 seconds

Example (1000 calls, 3 min avg):
3 min × 60 sec = 180 sec
180 / 15 = 12 units per call
1000 calls × 12 × $0.006 = $72/month
```

**Integration Example**:
```typescript
import speech from '@google-cloud/speech'

const client = new speech.SpeechClient()

async function transcribeAudio(audioStream: Stream) {
  const request = {
    config: {
      encoding: 'MULAW',
      sampleRateHertz: 8000,
      languageCode: 'bg-BG',
      model: 'latest_long',
      useEnhanced: true,
      enableAutomaticPunctuation: true,
    },
    interimResults: true,
  }

  const recognizeStream = client
    .streamingRecognize(request)
    .on('data', (data) => {
      const transcription = data.results[0]?.alternatives[0]?.transcript
      if (transcription) {
        // Send to LLM for processing
        processTranscription(transcription)
      }
    })

  audioStream.pipe(recognizeStream)
}
```

**Alternatives**:

| Provider | Bulgarian Support | Accuracy | Pricing | Recommendation |
|----------|------------------|----------|---------|----------------|
| **Azure Speech** | Good | ~90-95% | $1/hour | Good alternative, slightly more expensive |
| **OpenAI Whisper API** | Excellent | ~95% | $0.006/min | Great accuracy but may need optimization |
| **AssemblyAI** | Limited | ~85% | $0.00025/sec | Good price but lower Bulgarian quality |

**Recommendation**: **Google Cloud Speech** for production. Test **Whisper** for comparison.

### Text-to-Speech: Google Cloud TTS (WaveNet)

**Choice**: Google Cloud Text-to-Speech with WaveNet voices

**Justification**:
- ✅ **Natural Sound**: WaveNet produces most natural Bulgarian voices
- ✅ **Multiple Voices**: 2 Bulgarian voices (male/female)
- ✅ **SSML Support**: Control speed, pitch, emphasis
- ✅ **Streaming**: Low latency for real-time responses
- ✅ **Cost-Effective**: Reasonable pricing for quality

**Available Bulgarian Voices**:
- `bg-BG-Standard-A` (Female) - Standard quality
- `bg-BG-Wavenet-A` (Female) - Natural, recommended

**Pricing**:
```
WaveNet Voices: $16 per 1M characters
Standard Voices: $4 per 1M characters

Example (1000 calls, avg 500 characters response × 10 turns):
1000 × 500 × 10 = 5M characters
5M characters × $16 / 1M = $80/month
```

**Integration Example**:
```typescript
import textToSpeech from '@google-cloud/text-to-speech'

const client = new textToSpeech.TextToSpeechClient()

async function synthesizeSpeech(text: string): Promise<Buffer> {
  const request = {
    input: { text },
    voice: {
      languageCode: 'bg-BG',
      name: 'bg-BG-Wavenet-A',
      ssmlGender: 'FEMALE',
    },
    audioConfig: {
      audioEncoding: 'MP3',
      speakingRate: 1.0,
      pitch: 0.0,
      volumeGainDb: 0.0,
    },
  }

  const [response] = await client.synthesizeSpeech(request)
  return response.audioContent as Buffer
}
```

**Alternatives**:

| Provider | Bulgarian Quality | Voices Available | Pricing | Notes |
|----------|------------------|------------------|---------|-------|
| **ElevenLabs** | Excellent (best) | Custom cloning | $0.30/1K chars | Premium option, very natural |
| **Azure Neural TTS** | Very Good | 2 voices | $16/1M chars | Similar to Google |
| **Amazon Polly** | Good | 1 voice | $4/1M chars | Lower quality |
| **PlayHT** | Excellent | Custom voices | $0.24/1K chars | Good alternative to ElevenLabs |

**Recommendation**: **Google WaveNet** for MVP. Consider **ElevenLabs** for premium tier if customers willing to pay more for better quality.

### Large Language Model: OpenAI GPT-4 Turbo

**Choice**: GPT-4 Turbo with function calling

**Justification**:
- ✅ **Reasoning**: Best-in-class reasoning for complex conversations
- ✅ **Bulgarian**: Excellent Bulgarian language understanding
- ✅ **Function Calling**: Native support for booking actions
- ✅ **Context Window**: 128K tokens for long conversations
- ✅ **JSON Mode**: Structured outputs for reliability
- ✅ **Speed**: Turbo variant provides good latency

**Pricing**:
```
Input: $0.01 per 1K tokens
Output: $0.03 per 1K tokens

Example Conversation:
System Prompt: ~500 tokens
User Turn: ~100 tokens
Context: ~200 tokens
Response: ~150 tokens

Cost per turn: (800 × $0.01 + 150 × $0.03) / 1000 = $0.0125
10 turns per call: $0.125
1000 calls: $125/month
```

**Integration Example**:
```typescript
import OpenAI from 'openai'

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

async function generateResponse(conversation: Message[]) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4-turbo-preview',
    messages: [
      {
        role: 'system',
        content: `Вие сте виртуален рецепционист за хотел ${hotelName}.
        Вашата задача е да помагате на клиентите с резервации.
        Бъдете учтиви, професионални и полезни.`,
      },
      ...conversation,
    ],
    functions: [
      {
        name: 'check_availability',
        description: 'Check room availability for given dates',
        parameters: {
          type: 'object',
          properties: {
            check_in: { type: 'string', format: 'date' },
            check_out: { type: 'string', format: 'date' },
            guests: { type: 'number' },
            room_type: { type: 'string', enum: ['standard', 'deluxe', 'suite'] },
          },
          required: ['check_in', 'check_out', 'guests'],
        },
      },
      {
        name: 'create_booking',
        description: 'Create a new booking reservation',
        parameters: {
          type: 'object',
          properties: {
            guest_name: { type: 'string' },
            phone: { type: 'string' },
            email: { type: 'string', format: 'email' },
            check_in: { type: 'string', format: 'date' },
            check_out: { type: 'string', format: 'date' },
            guests: { type: 'number' },
            room_type: { type: 'string' },
            special_requests: { type: 'string' },
          },
          required: ['guest_name', 'phone', 'check_in', 'check_out', 'guests'],
        },
      },
    ],
    temperature: 0.7,
  })

  return response.choices[0]
}
```

**Alternatives**:

| Model | Bulgarian Quality | Reasoning | Function Calling | Cost (est.) | Notes |
|-------|------------------|-----------|------------------|-------------|-------|
| **Claude 3.5 Sonnet** | Excellent | Excellent | Yes | $3/$15 per 1M | Great alternative, better instruction following |
| **GPT-3.5 Turbo** | Good | Good | Yes | $0.50/$1.50 per 1M | 10x cheaper, good for MVP |
| **Gemini Pro** | Very Good | Very Good | Limited | Free (quota) | Good fallback, free tier |
| **Llama 3 70B** | Good | Good | Manual | Self-hosted | Open source, need infrastructure |

**Recommendation**:
- **MVP**: Start with **GPT-3.5 Turbo** to validate and reduce costs
- **Production**: Upgrade to **GPT-4 Turbo** or **Claude 3.5 Sonnet** for better quality
- **Hybrid**: Use GPT-3.5 for simple queries, GPT-4 for complex bookings

---

## Infrastructure & DevOps

### Cloud Provider: DigitalOcean

**Choice**: DigitalOcean for initial deployment

**Justification**:
- ✅ **Simplicity**: Easy to use, great for SMBs
- ✅ **Cost**: 30-40% cheaper than AWS for similar specs
- ✅ **EU Presence**: Frankfurt data center (GDPR compliant)
- ✅ **Managed Services**: Database, Redis, Kubernetes, Spaces
- ✅ **Predictable Pricing**: No surprise bills
- ✅ **Great DX**: Excellent documentation and UI

**Monthly Cost Estimate** (Medium Hotel):
```
- Kubernetes Cluster (3 nodes, 2 vCPU, 4GB): $72
- Managed PostgreSQL (2 vCPU, 4GB, 25GB): $60
- Managed Redis (1GB): $15
- Spaces (Object Storage, 250GB): $5
- Bandwidth (1TB): Included
- Load Balancer: $12
Total: ~$164/month
```

**Migration Path**: Start DigitalOcean → Scale to AWS/Azure if needed (multi-region, advanced features)

**Alternatives**:

| Provider | EU Presence | Managed Services | Cost (vs DO) | Best For |
|----------|-------------|------------------|--------------|----------|
| **AWS** | Frankfurt, Ireland | Extensive | +40-50% | Enterprise, scaling to 100+ hotels |
| **Azure** | Netherlands, Ireland | Extensive | +35-45% | Microsoft ecosystem integration |
| **Hetzner** | Germany, Finland | Basic | -30% | Cost-sensitive, simpler needs |
| **Google Cloud** | Belgium, Netherlands | Extensive | +45-55% | Heavy AI/ML usage |

### Container Orchestration: Kubernetes (K8s)

**Choice**: Managed Kubernetes (DigitalOcean Kubernetes)

**Justification**:
- ✅ **Scalability**: Auto-scaling pods based on load
- ✅ **Self-Healing**: Automatic restart of failed containers
- ✅ **Rolling Updates**: Zero-downtime deployments
- ✅ **Industry Standard**: Portable across cloud providers
- ✅ **Ecosystem**: Vast ecosystem of tools and integrations

**Simple Alternative for MVP**: Docker Compose on VPS
- Much simpler to start
- Good for single-hotel pilot
- Easy migration to K8s later
- Cost: ~$40/month for powerful VPS

**Recommendation**:
- **Phase 1 (Pilot)**: Docker Compose on VPS
- **Phase 2 (Production)**: Migrate to managed Kubernetes

### CI/CD: GitHub Actions

**Choice**: GitHub Actions for CI/CD

**Justification**:
- ✅ **Integration**: Native GitHub integration
- ✅ **Free Tier**: 2000 minutes/month for private repos
- ✅ **Easy Setup**: YAML-based, simple configuration
- ✅ **Marketplace**: Extensive action marketplace

**Pipeline Example**:
```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        uses: digitalocean/action-doctl@v2
        with:
          token: ${{ secrets.DIGITALOCEAN_TOKEN }}
      - run: kubectl apply -f k8s/
      - run: kubectl rollout status deployment/api
```

### Monitoring: Prometheus + Grafana

**Choice**: Prometheus for metrics, Grafana for visualization

**Justification**:
- ✅ **Open Source**: No licensing costs
- ✅ **K8s Native**: Built for Kubernetes
- ✅ **Powerful Queries**: PromQL for complex analysis
- ✅ **Alerting**: Alert manager integration
- ✅ **Ecosystem**: Extensive exporters

**Alternative**: **Datadog** (paid, excellent UX, $15/host/month)

---

## Development Tools

### Version Control: Git + GitHub

**Choice**: GitHub for code hosting

**Features Used**:
- Pull requests and code review
- GitHub Actions for CI/CD
- Project boards for planning
- GitHub Packages for Docker registry

### Code Quality

**Linting**: ESLint with TypeScript
**Formatting**: Prettier
**Testing**: Jest + Supertest (backend), Vitest + Testing Library (frontend)
**Type Checking**: TypeScript strict mode

**Pre-commit Hook** (Husky):
```json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

---

## Cost Summary (Monthly Operating Costs)

### Small Hotel (10-20 rooms, 50 calls/day)

```
Infrastructure:
- VPS (4 vCPU, 8GB): $40
- Database (managed, 1 vCPU): $15
- Object Storage: $5

Services:
- Telephony (Twilio): $13
- STT (Google): $11
- TTS (Google): $12
- LLM (GPT-3.5): $19

Total: ~$115/month
```

### Medium Hotel (20-50 rooms, 150 calls/day)

```
Infrastructure:
- Kubernetes (3 nodes): $72
- Database (2 vCPU, 4GB): $60
- Redis: $15
- Object Storage: $5

Services:
- Telephony: $40
- STT: $32
- TTS: $36
- LLM (GPT-4): $56

Total: ~$316/month
```

### Large Hotel (50+ rooms, 300 calls/day)

```
Infrastructure:
- Kubernetes (5 nodes): $120
- Database (4 vCPU, 8GB): $120
- Redis (cluster): $45
- Object Storage: $10

Services:
- Telephony: $85
- STT: $65
- TTS: $72
- LLM (GPT-4): $112

Total: ~$629/month
```

---

## Technology Selection Matrix

| Requirement | Technology | Alternative | Rationale |
|-------------|-----------|-------------|-----------|
| **Backend Runtime** | Node.js 20 | Python, Go | Async I/O, rich ecosystem |
| **Backend Framework** | NestJS | Express, Fastify | Enterprise architecture |
| **Database** | PostgreSQL 16 | MySQL, MongoDB | ACID, JSON support |
| **Cache** | Redis 7 | Memcached | Versatility, pub/sub |
| **Frontend** | Next.js 14 | Vite+React | SSR, great DX |
| **UI Library** | shadcn/ui | Material-UI | Customizable, modern |
| **State Management** | Zustand | Redux, Jotai | Simple, performant |
| **Telephony** | Twilio | Vonage, Local | Reliability, features |
| **STT** | Google Speech | Azure, Whisper | Bulgarian quality |
| **TTS** | Google WaveNet | ElevenLabs, Azure | Natural, cost-effective |
| **LLM** | GPT-4 Turbo | Claude, GPT-3.5 | Best reasoning |
| **Cloud** | DigitalOcean | AWS, Azure | Cost, simplicity |
| **Container** | Docker | - | Industry standard |
| **Orchestration** | Kubernetes | Docker Compose | Scalability (Phase 2) |
| **CI/CD** | GitHub Actions | GitLab CI | Integration, free tier |
| **Monitoring** | Prometheus | Datadog | Open source, powerful |

---

## Next Steps

1. Review [API Specifications](./04-api-specifications.md) for detailed endpoint definitions
2. See [Database Schema](./05-database-schema.md) for complete data models
3. Check [Implementation Roadmap](./10-implementation-roadmap.md) for development phases
4. Review [Cost Analysis](./11-cost-analysis.md) for detailed budget breakdown
