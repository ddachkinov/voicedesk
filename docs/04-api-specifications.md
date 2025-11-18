# API Specifications

## Overview

The Virtual Receptionist system exposes RESTful APIs for:
- **Telephony Webhooks**: Twilio integration for call handling
- **Booking Management**: CRUD operations for reservations
- **Calendar Operations**: Availability queries and updates
- **Analytics**: Reporting and metrics
- **User Management**: Authentication and authorization
- **Microinvest Integration**: PMS synchronization

**Base URL**: `https://api.virtualreceptionist.bg/v1`

**Authentication**: Bearer JWT tokens

**Rate Limiting**: 1000 requests/hour per API key

---

## Authentication

### POST /auth/login

Authenticate user and receive JWT token.

**Request**:
```json
{
  "email": "manager@hotel.bg",
  "password": "securepassword123"
}
```

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "manager@hotel.bg",
    "full_name": "Иван Петров",
    "role": "property_manager",
    "property_id": "660e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Errors**:
- `401 Unauthorized`: Invalid credentials
- `403 Forbidden`: Account disabled

---

### POST /auth/refresh

Refresh access token using refresh token.

**Request**:
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response** (200 OK):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600
}
```

---

## Telephony Webhooks

### POST /webhooks/twilio/incoming

Twilio webhook for incoming calls.

**Authentication**: Twilio signature validation

**Request** (from Twilio):
```json
{
  "CallSid": "CA123456789",
  "From": "+359888123456",
  "To": "+35929876543",
  "CallStatus": "ringing",
  "Direction": "inbound"
}
```

**Response** (200 OK, TwiML):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Say language="bg-BG" voice="Google.bg-BG-Wavenet-A">
        Добре дошли в Хотел Слънчев Бряг. Моля изчакайте, свързвам ви.
    </Say>
    <Gather input="speech" language="bg-BG" action="/webhooks/twilio/speech" timeout="5">
        <Say language="bg-BG">Как мога да ви помогна?</Say>
    </Gather>
</Response>
```

---

### POST /webhooks/twilio/speech

Process speech input from caller.

**Request** (from Twilio):
```json
{
  "CallSid": "CA123456789",
  "SpeechResult": "Искам да резервирам стая за 15-ти декември",
  "Confidence": 0.92
}
```

**Response** (200 OK, TwiML):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Say language="bg-BG">
        Разбирам, искате резервация за 15-ти декември. За колко нощувки?
    </Say>
    <Gather input="speech" language="bg-BG" action="/webhooks/twilio/speech">
    </Gather>
</Response>
```

---

### POST /webhooks/twilio/status

Call status updates from Twilio.

**Request**:
```json
{
  "CallSid": "CA123456789",
  "CallStatus": "completed",
  "CallDuration": "185"
}
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

## Calendar & Availability

### GET /properties/:propertyId/availability

Check room availability for date range.

**Authentication**: Required

**Query Parameters**:
- `check_in` (required): ISO 8601 date (e.g., "2025-12-15")
- `check_out` (required): ISO 8601 date
- `guests` (optional): Number of guests (default: 2)
- `room_type` (optional): Filter by room type

**Example Request**:
```
GET /properties/660e8400-e29b-41d4-a716-446655440000/availability
    ?check_in=2025-12-15
    &check_out=2025-12-20
    &guests=2
    &room_type=deluxe
```

**Response** (200 OK):
```json
{
  "check_in": "2025-12-15",
  "check_out": "2025-12-20",
  "nights": 5,
  "available_rooms": [
    {
      "id": "770e8400-e29b-41d4-a716-446655440000",
      "room_number": "201",
      "room_type": "deluxe",
      "capacity": 2,
      "price_per_night": 120.00,
      "total_price": 600.00,
      "currency": "BGN",
      "amenities": [
        "wifi",
        "air_conditioning",
        "balcony",
        "sea_view"
      ]
    },
    {
      "id": "880e8400-e29b-41d4-a716-446655440000",
      "room_number": "305",
      "room_type": "deluxe",
      "capacity": 3,
      "price_per_night": 140.00,
      "total_price": 700.00,
      "currency": "BGN",
      "amenities": [
        "wifi",
        "air_conditioning",
        "balcony",
        "sea_view",
        "bathtub"
      ]
    }
  ],
  "alternative_dates": [
    {
      "check_in": "2025-12-16",
      "check_out": "2025-12-21",
      "available_count": 5
    },
    {
      "check_in": "2025-12-14",
      "check_out": "2025-12-19",
      "available_count": 3
    }
  ]
}
```

**Errors**:
- `400 Bad Request`: Invalid date format or check_out before check_in
- `404 Not Found`: Property not found

---

### GET /properties/:propertyId/calendar

Get calendar view for property.

**Authentication**: Required

**Query Parameters**:
- `start_date` (required): Start of calendar range
- `end_date` (required): End of calendar range
- `room_id` (optional): Filter specific room

**Example Request**:
```
GET /properties/660e8400-e29b-41d4-a716-446655440000/calendar
    ?start_date=2025-12-01
    &end_date=2025-12-31
```

**Response** (200 OK):
```json
{
  "property_id": "660e8400-e29b-41d4-a716-446655440000",
  "start_date": "2025-12-01",
  "end_date": "2025-12-31",
  "rooms": [
    {
      "id": "770e8400-e29b-41d4-a716-446655440000",
      "room_number": "201",
      "room_type": "deluxe",
      "calendar": [
        {
          "date": "2025-12-15",
          "status": "available",
          "price": 120.00
        },
        {
          "date": "2025-12-16",
          "status": "occupied",
          "booking_id": "990e8400-e29b-41d4-a716-446655440000",
          "guest_name": "Мария Иванова"
        },
        {
          "date": "2025-12-17",
          "status": "blocked",
          "notes": "Ремонт"
        }
      ]
    }
  ],
  "summary": {
    "total_rooms": 25,
    "occupancy_rate": 68.5,
    "available_nights": 236,
    "occupied_nights": 540,
    "blocked_nights": 24
  }
}
```

---

### PATCH /rooms/:roomId/availability/:date

Update availability for specific room and date.

**Authentication**: Required (role: property_manager or higher)

**Request**:
```json
{
  "status": "blocked",
  "notes": "Планиран ремонт",
  "price_override": null
}
```

**Response** (200 OK):
```json
{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "room_id": "770e8400-e29b-41d4-a716-446655440000",
  "date": "2025-12-25",
  "status": "blocked",
  "notes": "Планиран ремонт",
  "price_override": null,
  "updated_at": "2025-11-18T10:30:00Z"
}
```

---

## Bookings

### POST /bookings

Create a new booking.

**Authentication**: Required (or API key for AI system)

**Request**:
```json
{
  "property_id": "660e8400-e29b-41d4-a716-446655440000",
  "room_id": "770e8400-e29b-41d4-a716-446655440000",
  "guest": {
    "full_name": "Петър Георгиев",
    "phone": "+359888123456",
    "email": "peter@example.com",
    "country": "BG"
  },
  "check_in": "2025-12-15",
  "check_out": "2025-12-20",
  "guests": 2,
  "special_requests": [
    "Стая на висок етаж",
    "Допълнителна възглавница"
  ],
  "call_id": "bb0e8400-e29b-41d4-a716-446655440000"
}
```

**Response** (201 Created):
```json
{
  "id": "cc0e8400-e29b-41d4-a716-446655440000",
  "confirmation_code": "A7B9C2D1",
  "property_id": "660e8400-e29b-41d4-a716-446655440000",
  "room_id": "770e8400-e29b-41d4-a716-446655440000",
  "guest_id": "dd0e8400-e29b-41d4-a716-446655440000",
  "guest": {
    "full_name": "Петър Георгиев",
    "phone": "+359888123456",
    "email": "peter@example.com"
  },
  "check_in": "2025-12-15",
  "check_out": "2025-12-20",
  "nights": 5,
  "guests": 2,
  "status": "confirmed",
  "total_price": 600.00,
  "currency": "BGN",
  "special_requests": [
    "Стая на висок етаж",
    "Допълнителна възглавница"
  ],
  "created_at": "2025-11-18T10:45:00Z",
  "confirmed_at": "2025-11-18T10:45:00Z"
}
```

**Errors**:
- `400 Bad Request`: Invalid data or room not available
- `409 Conflict`: Room already booked for those dates

---

### GET /bookings/:id

Get booking details.

**Authentication**: Required

**Response** (200 OK):
```json
{
  "id": "cc0e8400-e29b-41d4-a716-446655440000",
  "confirmation_code": "A7B9C2D1",
  "property": {
    "id": "660e8400-e29b-41d4-a716-446655440000",
    "name": "Хотел Слънчев Бряг"
  },
  "room": {
    "id": "770e8400-e29b-41d4-a716-446655440000",
    "room_number": "201",
    "room_type": "deluxe"
  },
  "guest": {
    "id": "dd0e8400-e29b-41d4-a716-446655440000",
    "full_name": "Петър Георгиев",
    "phone": "+359888123456",
    "email": "peter@example.com"
  },
  "check_in": "2025-12-15",
  "check_out": "2025-12-20",
  "nights": 5,
  "guests": 2,
  "status": "confirmed",
  "total_price": 600.00,
  "currency": "BGN",
  "special_requests": [
    "Стая на висок етаж",
    "Допълнителна възглавница"
  ],
  "call_id": "bb0e8400-e29b-41d4-a716-446655440000",
  "created_at": "2025-11-18T10:45:00Z",
  "confirmed_at": "2025-11-18T10:45:00Z",
  "history": [
    {
      "action": "created",
      "timestamp": "2025-11-18T10:45:00Z",
      "user": "AI System"
    },
    {
      "action": "confirmed",
      "timestamp": "2025-11-18T10:45:00Z",
      "user": "AI System"
    }
  ]
}
```

---

### GET /properties/:propertyId/bookings

List all bookings for a property.

**Authentication**: Required

**Query Parameters**:
- `status` (optional): Filter by status (pending, confirmed, etc.)
- `start_date` (optional): Filter bookings with check_in >= start_date
- `end_date` (optional): Filter bookings with check_out <= end_date
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20, max: 100)

**Example Request**:
```
GET /properties/660e8400-e29b-41d4-a716-446655440000/bookings
    ?status=confirmed
    &start_date=2025-12-01
    &page=1
    &limit=20
```

**Response** (200 OK):
```json
{
  "data": [
    {
      "id": "cc0e8400-e29b-41d4-a716-446655440000",
      "confirmation_code": "A7B9C2D1",
      "guest_name": "Петър Георгиев",
      "room_number": "201",
      "check_in": "2025-12-15",
      "check_out": "2025-12-20",
      "nights": 5,
      "guests": 2,
      "status": "confirmed",
      "total_price": 600.00,
      "created_at": "2025-11-18T10:45:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "total_pages": 3
  }
}
```

---

### PATCH /bookings/:id

Update booking details.

**Authentication**: Required

**Request**:
```json
{
  "status": "cancelled",
  "cancellation_reason": "Промяна в плановете на клиента"
}
```

**Response** (200 OK):
```json
{
  "id": "cc0e8400-e29b-41d4-a716-446655440000",
  "status": "cancelled",
  "cancellation_reason": "Промяна в плановете на клиента",
  "cancelled_at": "2025-11-18T11:00:00Z",
  "updated_at": "2025-11-18T11:00:00Z"
}
```

---

### DELETE /bookings/:id

Cancel a booking (soft delete).

**Authentication**: Required

**Response** (204 No Content)

---

## Calls & Conversations

### GET /calls/:id

Get call details including transcript.

**Authentication**: Required

**Response** (200 OK):
```json
{
  "id": "bb0e8400-e29b-41d4-a716-446655440000",
  "property_id": "660e8400-e29b-41d4-a716-446655440000",
  "twilio_call_sid": "CA123456789",
  "caller_number": "+359888123456",
  "started_at": "2025-11-18T10:30:00Z",
  "ended_at": "2025-11-18T10:35:30Z",
  "duration_seconds": 330,
  "status": "completed",
  "outcome": "booking_created",
  "sentiment_score": 0.85,
  "guest": {
    "id": "dd0e8400-e29b-41d4-a716-446655440000",
    "full_name": "Петър Георгиев",
    "phone": "+359888123456"
  },
  "booking_id": "cc0e8400-e29b-41d4-a716-446655440000",
  "transcript": [
    {
      "turn": 1,
      "speaker": "ai",
      "content": "Добре дошли в Хотел Слънчев Бряг. Как мога да ви помогна?",
      "timestamp": "2025-11-18T10:30:05Z"
    },
    {
      "turn": 2,
      "speaker": "caller",
      "content": "Здравейте, искам да резервирам стая за 15-ти декември.",
      "confidence": 0.92,
      "timestamp": "2025-11-18T10:30:12Z"
    },
    {
      "turn": 3,
      "speaker": "ai",
      "content": "Разбирам, искате резервация за 15-ти декември. За колко нощувки?",
      "timestamp": "2025-11-18T10:30:18Z"
    },
    {
      "turn": 4,
      "speaker": "caller",
      "content": "За пет нощувки, до 20-ти декември.",
      "confidence": 0.95,
      "timestamp": "2025-11-18T10:30:25Z"
    }
  ],
  "recording": {
    "id": "ee0e8400-e29b-41d4-a716-446655440000",
    "url": "https://recordings.virtualreceptionist.bg/calls/bb0e8400-e29b-41d4-a716-446655440000.mp3",
    "duration_seconds": 330,
    "format": "mp3"
  }
}
```

---

### GET /properties/:propertyId/calls

List calls for a property.

**Authentication**: Required

**Query Parameters**:
- `outcome` (optional): Filter by outcome
- `start_date` (optional): Filter calls after this date
- `end_date` (optional): Filter calls before this date
- `page` (optional): Page number
- `limit` (optional): Items per page

**Response** (200 OK):
```json
{
  "data": [
    {
      "id": "bb0e8400-e29b-41d4-a716-446655440000",
      "caller_number": "+359888123456",
      "started_at": "2025-11-18T10:30:00Z",
      "duration_seconds": 330,
      "outcome": "booking_created",
      "sentiment_score": 0.85,
      "booking_id": "cc0e8400-e29b-41d4-a716-446655440000"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 152,
    "total_pages": 8
  }
}
```

---

## Analytics

### GET /properties/:propertyId/analytics/summary

Get summary analytics for a property.

**Authentication**: Required

**Query Parameters**:
- `start_date` (required): Start of period
- `end_date` (required): End of period

**Response** (200 OK):
```json
{
  "period": {
    "start_date": "2025-11-01",
    "end_date": "2025-11-30"
  },
  "calls": {
    "total": 324,
    "successful": 298,
    "failed": 26,
    "avg_duration_seconds": 245,
    "total_duration_minutes": 1323
  },
  "outcomes": {
    "booking_created": 87,
    "inquiry_only": 145,
    "no_availability": 42,
    "transferred_to_staff": 18,
    "abandoned": 23,
    "error": 9
  },
  "conversion_rate": 26.85,
  "bookings": {
    "total": 87,
    "total_revenue": 15240.00,
    "currency": "BGN",
    "avg_booking_value": 175.17,
    "avg_nights": 3.2
  },
  "sentiment": {
    "avg_score": 0.72,
    "positive_calls": 245,
    "neutral_calls": 52,
    "negative_calls": 27
  },
  "peak_hours": [
    {"hour": 10, "calls": 45},
    {"hour": 14, "calls": 38},
    {"hour": 16, "calls": 35}
  ],
  "popular_dates": [
    {"date": "2025-12-24", "inquiries": 28},
    {"date": "2025-12-31", "inquiries": 25},
    {"date": "2025-12-25", "inquiries": 22}
  ]
}
```

---

### GET /properties/:propertyId/analytics/calls-by-day

Get daily call volume.

**Authentication**: Required

**Query Parameters**:
- `start_date` (required)
- `end_date` (required)

**Response** (200 OK):
```json
{
  "data": [
    {
      "date": "2025-11-01",
      "total_calls": 12,
      "bookings_created": 3,
      "conversion_rate": 25.0,
      "avg_duration_seconds": 215,
      "avg_sentiment": 0.75
    },
    {
      "date": "2025-11-02",
      "total_calls": 15,
      "bookings_created": 5,
      "conversion_rate": 33.33,
      "avg_duration_seconds": 268,
      "avg_sentiment": 0.82
    }
  ]
}
```

---

## Microinvest Integration

### POST /properties/:propertyId/microinvest/sync

Trigger synchronization with Microinvest.

**Authentication**: Required (role: property_manager or higher)

**Request**:
```json
{
  "sync_type": "incremental_sync",
  "start_date": "2025-11-01",
  "end_date": "2025-12-31"
}
```

**Response** (202 Accepted):
```json
{
  "sync_id": "ff0e8400-e29b-41d4-a716-446655440000",
  "status": "in_progress",
  "started_at": "2025-11-18T11:00:00Z",
  "estimated_completion": "2025-11-18T11:05:00Z"
}
```

---

### GET /properties/:propertyId/microinvest/sync/:syncId

Get sync job status.

**Authentication**: Required

**Response** (200 OK):
```json
{
  "id": "ff0e8400-e29b-41d4-a716-446655440000",
  "sync_type": "incremental_sync",
  "status": "completed",
  "started_at": "2025-11-18T11:00:00Z",
  "completed_at": "2025-11-18T11:03:42Z",
  "records_processed": 47,
  "errors": [],
  "summary": {
    "rooms_synced": 25,
    "bookings_synced": 18,
    "availability_updated": 750
  }
}
```

---

## WebSocket API

### WS /ws/calls

Real-time call updates.

**Authentication**: JWT token via query param `?token=...`

**Connection**:
```javascript
const ws = new WebSocket('wss://api.virtualreceptionist.bg/ws/calls?token=JWT_TOKEN')
```

**Subscribe to property calls**:
```json
{
  "type": "subscribe",
  "property_id": "660e8400-e29b-41d4-a716-446655440000"
}
```

**Events received**:

**Call Started**:
```json
{
  "type": "call_started",
  "data": {
    "call_id": "bb0e8400-e29b-41d4-a716-446655440000",
    "caller_number": "+359888123456",
    "started_at": "2025-11-18T10:30:00Z"
  }
}
```

**Call Updated**:
```json
{
  "type": "call_updated",
  "data": {
    "call_id": "bb0e8400-e29b-41d4-a716-446655440000",
    "status": "in_progress",
    "transcript_turn": {
      "turn": 5,
      "speaker": "caller",
      "content": "Искам стая с изглед към морето."
    }
  }
}
```

**Call Ended**:
```json
{
  "type": "call_ended",
  "data": {
    "call_id": "bb0e8400-e29b-41d4-a716-446655440000",
    "ended_at": "2025-11-18T10:35:30Z",
    "duration_seconds": 330,
    "outcome": "booking_created",
    "booking_id": "cc0e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

## Error Responses

All error responses follow this format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {
        "field": "check_in",
        "message": "check_in must be a valid ISO 8601 date"
      }
    ]
  },
  "request_id": "req_abc123xyz"
}
```

**Common Error Codes**:
- `VALIDATION_ERROR` (400): Invalid request data
- `UNAUTHORIZED` (401): Missing or invalid authentication
- `FORBIDDEN` (403): Insufficient permissions
- `NOT_FOUND` (404): Resource not found
- `CONFLICT` (409): Resource conflict (e.g., double booking)
- `RATE_LIMIT_EXCEEDED` (429): Too many requests
- `INTERNAL_ERROR` (500): Server error

---

## Rate Limiting

**Headers**:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 845
X-RateLimit-Reset: 1700308800
```

**When limit exceeded** (429 Too Many Requests):
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 15 minutes.",
    "retry_after": 900
  }
}
```

---

## Pagination

List endpoints support cursor-based pagination:

**Request**:
```
GET /properties/660e8400-e29b-41d4-a716-446655440000/bookings?limit=20&cursor=eyJpZCI6IjEyMyJ9
```

**Response**:
```json
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "next_cursor": "eyJpZCI6IjE0MyJ9",
    "has_more": true
  }
}
```

---

## Webhooks (Outbound)

Properties can register webhooks to receive events.

### Webhook Events

**booking.created**:
```json
{
  "event": "booking.created",
  "timestamp": "2025-11-18T10:45:00Z",
  "data": {
    "booking_id": "cc0e8400-e29b-41d4-a716-446655440000",
    "confirmation_code": "A7B9C2D1",
    "guest_name": "Петър Георгиев",
    "check_in": "2025-12-15",
    "check_out": "2025-12-20"
  }
}
```

**call.completed**:
```json
{
  "event": "call.completed",
  "timestamp": "2025-11-18T10:35:30Z",
  "data": {
    "call_id": "bb0e8400-e29b-41d4-a716-446655440000",
    "duration_seconds": 330,
    "outcome": "booking_created",
    "booking_id": "cc0e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

## SDK Examples

### JavaScript/TypeScript

```typescript
import { VirtualReceptionistClient } from '@virtualreceptionist/sdk'

const client = new VirtualReceptionistClient({
  apiKey: process.env.VR_API_KEY,
  baseURL: 'https://api.virtualreceptionist.bg/v1'
})

// Check availability
const availability = await client.availability.check({
  propertyId: 'property-uuid',
  checkIn: '2025-12-15',
  checkOut: '2025-12-20',
  guests: 2
})

// Create booking
const booking = await client.bookings.create({
  propertyId: 'property-uuid',
  roomId: 'room-uuid',
  guest: {
    fullName: 'Петър Георгиев',
    phone: '+359888123456',
    email: 'peter@example.com'
  },
  checkIn: '2025-12-15',
  checkOut: '2025-12-20',
  guests: 2
})

// Listen to real-time calls
client.calls.subscribe('property-uuid', (event) => {
  console.log('Call event:', event)
})
```

---

## API Versioning

The API uses URL versioning (`/v1`, `/v2`, etc.).

**Current Version**: `v1`

**Deprecation Policy**:
- Minimum 6 months notice before deprecating endpoints
- 12 months support for deprecated versions

---

## Next Steps

1. Review [Call Flow Diagrams](./07-call-flows.md) to understand conversation logic
2. See [Database Schema](./05-database-schema.md) for data models
3. Check [Implementation Roadmap](./10-implementation-roadmap.md) for development phases
