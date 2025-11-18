# Testing Strategy

## Overview

Comprehensive testing strategy for the Virtual Receptionist system to ensure quality, reliability, and customer satisfaction. Testing covers voice AI, booking workflows, Bulgarian language quality, and system integration.

---

## Testing Pyramid

```
                    /\
                   /  \
                  / E2E \
                 /  Tests \
                /──────────\
               /Integration \
              /    Tests     \
             /────────────────\
            /   Unit Tests     \
           /                    \
          /______________________\

          70% Unit | 20% Integration | 10% E2E
```

**Target Coverage**:
- Unit Tests: 80%+ code coverage
- Integration Tests: All API endpoints
- E2E Tests: All critical user flows
- Manual Tests: Bulgarian language quality

---

## Unit Testing

### Backend (NestJS/TypeScript)

**Framework**: Jest

**Coverage Requirements**:
- Controllers: 80%
- Services: 90%
- Utilities: 95%
- Models: 80%

**Example: Booking Service**

```typescript
// src/modules/booking/booking.service.spec.ts
import { Test } from '@nestjs/testing'
import { BookingService } from './booking.service'
import { getRepositoryToken } from '@nestjs/typeorm'
import { Booking } from './booking.entity'

describe('BookingService', () => {
  let service: BookingService
  let repository: any

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        BookingService,
        {
          provide: getRepositoryToken(Booking),
          useValue: {
            find: jest.fn(),
            findOne: jest.fn(),
            save: jest.fn(),
            delete: jest.fn()
          }
        }
      ]
    }).compile()

    service = module.get<BookingService>(BookingService)
    repository = module.get(getRepositoryToken(Booking))
  })

  describe('createBooking', () => {
    it('should create a booking successfully', async () => {
      const bookingData = {
        roomId: 'room-123',
        guestId: 'guest-456',
        checkIn: new Date('2025-12-15'),
        checkOut: new Date('2025-12-20'),
        guests: 2
      }

      repository.save.mockResolvedValue({ id: 'booking-789', ...bookingData })

      const result = await service.createBooking(bookingData)

      expect(result).toBeDefined()
      expect(result.id).toBe('booking-789')
      expect(repository.save).toHaveBeenCalledWith(
        expect.objectContaining(bookingData)
      )
    })

    it('should throw error if room not available', async () => {
      const bookingData = {
        roomId: 'room-123',
        checkIn: new Date('2025-12-15'),
        checkOut: new Date('2025-12-20')
      }

      jest.spyOn(service, 'checkAvailability').mockResolvedValue(false)

      await expect(service.createBooking(bookingData))
        .rejects
        .toThrow('Room not available for selected dates')
    })

    it('should validate check-out after check-in', async () => {
      const bookingData = {
        checkIn: new Date('2025-12-20'),
        checkOut: new Date('2025-12-15') // Invalid: before check-in
      }

      await expect(service.createBooking(bookingData))
        .rejects
        .toThrow('Check-out must be after check-in')
    })
  })

  describe('checkAvailability', () => {
    it('should return true when room is available', async () => {
      repository.find.mockResolvedValue([]) // No conflicting bookings

      const available = await service.checkAvailability(
        'room-123',
        new Date('2025-12-15'),
        new Date('2025-12-20')
      )

      expect(available).toBe(true)
    })

    it('should return false when room is occupied', async () => {
      repository.find.mockResolvedValue([
        { id: 'booking-existing' } // Conflicting booking exists
      ])

      const available = await service.checkAvailability(
        'room-123',
        new Date('2025-12-15'),
        new Date('2025-12-20')
      )

      expect(available).toBe(false)
    })
  })
})
```

### Frontend (Next.js/React)

**Framework**: Vitest + Testing Library

**Example: Calendar Component**

```typescript
// src/components/calendar/calendar.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Calendar } from './calendar'

describe('Calendar', () => {
  it('should render current month', () => {
    render(<Calendar />)

    expect(screen.getByText(/Декември 2025/i)).toBeInTheDocument()
  })

  it('should show room status correctly', () => {
    const rooms = [
      { id: '1', number: '201', status: 'available' },
      { id: '2', number: '305', status: 'occupied' }
    ]

    render(<Calendar rooms={rooms} />)

    const room201 = screen.getByTestId('room-201-15')
    const room305 = screen.getByTestId('room-305-15')

    expect(room201).toHaveClass('status-available')
    expect(room305).toHaveClass('status-occupied')
  })

  it('should navigate to next month', () => {
    render(<Calendar />)

    const nextButton = screen.getByLabelText('Next month')
    fireEvent.click(nextButton)

    expect(screen.getByText(/Януари 2026/i)).toBeInTheDocument()
  })
})
```

---

## Integration Testing

### API Integration Tests

**Framework**: Supertest + Jest

**Example: Booking API**

```typescript
// src/modules/booking/booking.controller.spec.ts
import request from 'supertest'
import { Test } from '@nestjs/testing'
import { AppModule } from '@/app.module'
import { INestApplication } from '@nestjs/common'

describe('Booking API (e2e)', () => {
  let app: INestApplication
  let authToken: string

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule]
    }).compile()

    app = module.createNestApplication()
    await app.init()

    // Get auth token
    const loginResponse = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'password' })

    authToken = loginResponse.body.access_token
  })

  afterAll(async () => {
    await app.close()
  })

  describe('POST /bookings', () => {
    it('should create a booking', async () => {
      const response = await request(app.getHttpServer())
        .post('/bookings')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          propertyId: 'property-123',
          roomId: 'room-456',
          guest: {
            fullName: 'Петър Георгиев',
            phone: '+359888123456',
            email: 'peter@example.com'
          },
          checkIn: '2025-12-15',
          checkOut: '2025-12-20',
          guests: 2
        })
        .expect(201)

      expect(response.body).toMatchObject({
        id: expect.any(String),
        confirmationCode: expect.stringMatching(/^[A-Z0-9]{8}$/),
        status: 'confirmed',
        checkIn: '2025-12-15',
        checkOut: '2025-12-20'
      })
    })

    it('should return 400 for invalid dates', async () => {
      await request(app.getHttpServer())
        .post('/bookings')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          roomId: 'room-456',
          checkIn: '2025-12-20',
          checkOut: '2025-12-15' // Invalid: before check-in
        })
        .expect(400)
        .expect(res => {
          expect(res.body.error.message).toContain('Check-out must be after check-in')
        })
    })

    it('should return 409 for double booking', async () => {
      // Create first booking
      await request(app.getHttpServer())
        .post('/bookings')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          roomId: 'room-456',
          checkIn: '2025-12-15',
          checkOut: '2025-12-20'
        })

      // Try to create overlapping booking
      await request(app.getHttpServer())
        .post('/bookings')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          roomId: 'room-456',
          checkIn: '2025-12-17', // Overlaps
          checkOut: '2025-12-22'
        })
        .expect(409)
    })

    it('should require authentication', async () => {
      await request(app.getHttpServer())
        .post('/bookings')
        .send({ roomId: 'room-456' })
        .expect(401)
    })
  })

  describe('GET /bookings/:id', () => {
    it('should return booking details', async () => {
      const createResponse = await request(app.getHttpServer())
        .post('/bookings')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          roomId: 'room-456',
          checkIn: '2025-12-15',
          checkOut: '2025-12-20'
        })

      const bookingId = createResponse.body.id

      const response = await request(app.getHttpServer())
        .get(`/bookings/${bookingId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)

      expect(response.body).toMatchObject({
        id: bookingId,
        checkIn: '2025-12-15',
        checkOut: '2025-12-20'
      })
    })

    it('should return 404 for non-existent booking', async () => {
      await request(app.getHttpServer())
        .get('/bookings/non-existent-id')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(404)
    })
  })
})
```

---

## End-to-End (E2E) Testing

### Conversation Flow Tests

**Framework**: Custom test harness

**Test Scenario: Happy Path Booking**

```typescript
// tests/e2e/conversation/booking-happy-path.spec.ts
import { ConversationTester } from '@/tests/utils/conversation-tester'

describe('E2E: Successful Booking Conversation', () => {
  let tester: ConversationTester

  beforeEach(() => {
    tester = new ConversationTester({
      propertyId: 'test-property',
      language: 'bg-BG'
    })
  })

  it('should complete a booking from start to finish', async () => {
    // Initialize call
    const call = await tester.startCall('+359888123456')

    // AI greeting
    const greeting = await call.getResponse()
    expect(greeting).toContain('Добре дошли')

    // User wants to book
    await call.say('Искам да резервирам стая за 15-ти декември')

    const response1 = await call.getResponse()
    expect(response1).toContain('За колко нощувки')

    // User provides duration
    await call.say('За пет нощувки')

    const response2 = await call.getResponse()
    expect(response2).toContain('За колко човека')

    // User provides guest count
    await call.say('За двама')

    const response3 = await call.getResponse()
    expect(response3).toMatch(/стандартн|делукс|апартамент/i)

    // User selects room
    await call.say('Делукс стая с изглед')

    const response4 = await call.getResponse()
    expect(response4).toContain('Отличен избор')
    expect(response4).toContain('вашите три имена')

    // Provide guest info
    await call.say('Петър Георгиев Иванов')
    await call.waitForResponse()

    await call.say('Нула осемстотин и осемдесет и осем, един две три, четири пет шест')
    await call.waitForResponse()

    await call.say('petar@example.com')

    const summary = await call.getResponse()
    expect(summary).toContain('Петър Георгиев Иванов')
    expect(summary).toContain('0888 123 456')
    expect(summary).toContain('24 декември')

    // Confirm
    await call.say('Да, всичко е наред')

    const confirmation = await call.getResponse()
    expect(confirmation).toMatch(/потвърдена/i)
    expect(confirmation).toMatch(/[A-Z0-9]{8}/) // Confirmation code

    // Verify booking created in database
    const booking = await tester.getLatestBooking()
    expect(booking).toMatchObject({
      guest: {
        fullName: 'Петър Георгиев Иванов',
        phone: '0888123456',
        email: 'petar@example.com'
      },
      checkIn: '2025-12-15',
      checkOut: '2025-12-20',
      status: 'confirmed'
    })

    // End call
    await call.end()

    // Verify call record
    const callRecord = await tester.getCallRecord(call.id)
    expect(callRecord.outcome).toBe('booking_created')
    expect(callRecord.duration).toBeGreaterThan(0)
  })
})
```

### UI E2E Tests

**Framework**: Playwright

```typescript
// tests/e2e/ui/dashboard.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Dashboard', () => {
  test('should show active calls', async ({ page }) => {
    await page.goto('https://app.virtualreceptionist.bg')

    // Login
    await page.fill('input[name="email"]', 'test@example.com')
    await page.fill('input[name="password"]', 'password')
    await page.click('button[type="submit"]')

    // Wait for dashboard
    await page.waitForSelector('h1:has-text("Табло")')

    // Check metrics
    const callsToday = await page.textContent('[data-testid="calls-today"]')
    expect(callsToday).toMatch(/\d+/)

    // Check active calls section
    const activeCalls = page.locator('[data-testid="active-calls"]')
    await expect(activeCalls).toBeVisible()
  })

  test('should create manual booking', async ({ page }) => {
    await page.goto('https://app.virtualreceptionist.bg/bookings')

    // Click new booking
    await page.click('button:has-text("Нова резервация")')

    // Fill form
    await page.fill('input[name="guestName"]', 'Мария Иванова')
    await page.fill('input[name="phone"]', '0889765432')
    await page.fill('input[name="email"]', 'maria@example.com')

    await page.fill('input[name="checkIn"]', '2025-12-18')
    await page.fill('input[name="checkOut"]', '2025-12-22')

    await page.selectOption('select[name="roomId"]', 'room-305')

    // Submit
    await page.click('button[type="submit"]')

    // Verify success message
    await expect(page.locator('.toast-success')).toContainText('Резервацията е създадена')

    // Verify in list
    await expect(page.locator('table')).toContainText('Мария Иванова')
  })
})
```

---

## Bulgarian Language Testing

### Manual Testing Scenarios

**Test Cases**:

| Scenario | Input (Bulgarian) | Expected Intent | Expected Entities |
|----------|-------------------|----------------|-------------------|
| Basic booking | "Искам да резервирам стая" | booking | - |
| Specific dates | "За 15-ти декември до 20-ти" | booking | dates: [2025-12-15, 2025-12-20] |
| Guest count | "За двама възрастни и едно дете" | booking | guests: {adults: 2, children: 1} |
| Room preference | "Искам делукс стая" | booking | room_type: deluxe |
| Amenities inquiry | "Имате ли басейн?" | inquiry | topic: amenities |
| Location inquiry | "Къде се намира хотелът?" | inquiry | topic: location |
| Transfer request | "Искам да говоря с човек" | transfer | - |

### Speech Recognition Quality Test

```typescript
// tests/quality/stt-accuracy.spec.ts
describe('STT Accuracy (Bulgarian)', () => {
  const testPhrases = [
    { audio: 'audio/booking-request.wav', expected: 'искам да резервирам стая' },
    { audio: 'audio/dates.wav', expected: 'от петнадесети до двадесети декември' },
    { audio: 'audio/guests.wav', expected: 'за двама възрастни и едно дете' },
    // ... more test cases
  ]

  testPhrases.forEach(({ audio, expected }) => {
    it(`should correctly transcribe: "${expected}"`, async () => {
      const transcription = await sttService.transcribe(audio)

      const similarity = calculateLevenshteinSimilarity(transcription, expected)

      expect(similarity).toBeGreaterThan(0.85) // 85% accuracy threshold
    })
  })
})
```

### Voice Quality Evaluation

**Metrics**:
- Naturalness (1-5 scale)
- Pronunciation accuracy
- Intonation appropriateness
- Speaking pace

**Testing Method**:
- Generate 50 sample responses
- Native Bulgarian speakers rate each
- Target average score: >4.0/5

---

## Performance Testing

### Load Testing

**Framework**: k6

```javascript
// tests/performance/load-test.js
import http from 'k6/http'
import { check, sleep } from 'k6'

export let options = {
  stages: [
    { duration: '2m', target: 10 },  // Ramp up to 10 concurrent calls
    { duration: '5m', target: 10 },  // Stay at 10
    { duration: '2m', target: 20 },  // Ramp to 20
    { duration: '5m', target: 20 },  // Stay at 20
    { duration: '2m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<3000'], // 95% of requests under 3s
    http_req_failed: ['rate<0.05'],    // <5% failure rate
  }
}

export default function() {
  // Simulate incoming call
  const callPayload = JSON.stringify({
    From: '+359888123456',
    CallSid: `CA${Math.random().toString(36).substr(2, 32)}`
  })

  const res = http.post('https://api.virtualreceptionist.bg/webhooks/twilio/incoming', callPayload, {
    headers: { 'Content-Type': 'application/json' }
  })

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 3s': (r) => r.timings.duration < 3000,
    'contains TwiML': (r) => r.body.includes('<Response>')
  })

  sleep(30) // Average call duration: 30 seconds
}
```

### Stress Testing

**Objectives**:
- Find breaking point
- Measure recovery time
- Identify bottlenecks

**Scenario**:
- Gradually increase load from 10 to 100 concurrent calls
- Monitor response times, error rates, CPU, memory
- Identify at what point system degrades

**Acceptance Criteria**:
- Handle 50 concurrent calls without degradation
- Graceful degradation beyond capacity
- Auto-recovery within 2 minutes

---

## Security Testing

### Penetration Testing

**Scope**:
- Authentication bypass attempts
- SQL injection
- XSS attacks
- CSRF attacks
- API abuse
- Data exposure

**Tools**:
- OWASP ZAP
- Burp Suite
- SQLMap
- Manual testing

**Schedule**: Quarterly

### Vulnerability Scanning

```bash
# Dependency vulnerability scan
npm audit

# SAST (Static Application Security Testing)
npm run lint:security

# Container scanning
trivy image virtualreceptionist:latest

# Infrastructure scanning
checkov -d ./infrastructure
```

---

## Regression Testing

### Test Suite

Automated regression suite runs on every PR:

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Unit Tests
        run: npm test

      - name: Integration Tests
        run: npm run test:integration

      - name: E2E Tests
        run: npm run test:e2e

      - name: Coverage Report
        run: npm run test:coverage

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
```

**Criteria for Passing**:
- All unit tests pass
- All integration tests pass
- Critical E2E tests pass
- Code coverage >80%
- No critical security vulnerabilities

---

## Acceptance Testing

### User Acceptance Testing (UAT)

**Participants**: Pilot hotel staff

**Duration**: 2 weeks

**Test Scenarios**:
1. Make 10 test calls with different scenarios
2. Review transcripts for accuracy
3. Verify bookings created correctly
4. Test dashboard functionality
5. Test conflict resolution
6. Test CSV import

**Success Criteria**:
- >90% call success rate
- >85% language accuracy rating
- >80% user satisfaction
- All critical bugs fixed

### Beta Testing

**Participants**: 5-10 hotels

**Duration**: 1 month

**Metrics**:
- Call volume and success rate
- Booking conversion rate
- User feedback (surveys)
- Bug reports
- Feature requests

---

## Test Data Management

### Test Data Sets

**Properties**:
```json
{
  "small_hotel": { "rooms": 15, "types": ["standard", "deluxe"] },
  "medium_hotel": { "rooms": 35, "types": ["standard", "deluxe", "suite"] },
  "large_hotel": { "rooms": 75, "types": ["standard", "deluxe", "suite", "penthouse"] }
}
```

**Bookings**:
- Past bookings (completed)
- Current bookings (active)
- Future bookings (upcoming)
- Cancelled bookings
- Edge cases (same-day, long-stay, etc.)

**Calls**:
- Successful bookings
- Inquiries only
- Transfers
- Abandoned calls
- Error scenarios

---

## Continuous Testing

### CI/CD Integration

```
Code Commit → Unit Tests → Integration Tests → Build → Deploy to Staging → E2E Tests → Deploy to Production
```

**Automated Testing Gates**:
- Pre-commit: Linting, type checking
- Pre-push: Unit tests
- PR: Full test suite
- Pre-deployment: Integration + E2E tests
- Post-deployment: Smoke tests

---

## Monitoring & Alerting

### Production Monitoring

**Synthetic Monitoring**:
- Simulate calls every 15 minutes
- Monitor availability endpoints
- Track response times
- Alert on failures

**Real User Monitoring**:
- Track actual call success rates
- Monitor conversation quality
- Measure booking conversion
- Sentiment analysis

**Alerts**:
- Call success rate <85%
- API response time >3s
- Error rate >5%
- Database connection issues
- High CPU/memory usage

---

## Test Reporting

### Test Metrics

- **Test Coverage**: 80%+
- **Pass Rate**: 95%+
- **Execution Time**: <10 minutes (unit + integration)
- **Flaky Test Rate**: <2%
- **Bug Detection Rate**: Track bugs found in testing vs production

### Weekly Test Report

```
Week of 2025-11-18

Test Execution:
- Unit Tests: 1,234 passed, 2 failed (99.8%)
- Integration Tests: 156 passed, 0 failed (100%)
- E2E Tests: 45 passed, 1 failed (97.8%)
- Coverage: 82.5% (+1.2% from last week)

Quality Metrics:
- Call Success Rate: 89.3%
- Bulgarian STT Accuracy: 91.2%
- Average Response Time: 1.8s

Bugs:
- 3 new bugs reported
- 5 bugs fixed
- 2 bugs in progress
- 0 critical bugs open

Recommendations:
- Investigate E2E test failure in booking flow
- Improve STT accuracy for noisy environments
- Optimize response time for availability queries
```

---

## Next Steps

1. Review [Implementation Roadmap](./10-implementation-roadmap.md) for testing timeline
2. See [Security & Compliance](./09-security-compliance.md) for security testing requirements
3. Check [Deployment & Operations](./13-deployment-operations.md) for production monitoring
