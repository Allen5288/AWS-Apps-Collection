# Simple Async User Register API

A simple serverless API that demonstrates async processing with AWS services.

## 📝 Further implementation things:
For production and furthermore development, add multiple middleware, add authentication, add more user vertification process like Cognito, adding JWT token, validation, pagination, filtering, rate limit, eror handling, api document, security header, input sanitization, monitoring, cloudwatch, add unit test, integration test, CICD pipeline.

And remove all console log~~！ But currently, for the test version

## 🎯 What This API Does

- **Register users** → Process asynchronously via SNS → Store in DynamoDB
- **Create message boards** → Process asynchronously via SQS → Store in DynamoDB
- **Post messages** → Process asynchronously via SNS → Store in DynamoDB + WebSocket notifications
- **Real-time messages** → WebSocket subscriptions for live updates

## 🚀 API Endpoints (v1)

All endpoints are prefixed with `/api/v1`

- `POST /api/v1/register` - Register user (returns 202, processes async)
- `GET /api/v1/user/{email}` - Get user details
- `GET /api/v1/boards` - List all message boards
- `POST /api/v1/boards` - Create board (returns 202, processes async)
- `POST /api/v1/boards/{id}/messages` - Post message (returns 202, processes async)
- `GET /api/v1/boards/{id}/messages` - Get messages from board

## 🛠️ Setup & Deployment

### 1. Prerequisites
- AWS Account
- Node.js 18+
- AWS CLI installed

### 2. AWS Setup
```bash
# Install AWS CLI
# Download from: https://aws.amazon.com/cli/

# Configure AWS credentials
aws configure
# Enter your:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (us-east-1)
# - Output format (json)
```

### 3. Install Dependencies
```bash
npm install
npm install -g serverless
```

### 4. Deploy
```bash
# Deploy to AWS
npm run deploy

# After deployment, update .env file with the API URLs shown in output
```

### 5. Test
```bash
# Run automated tests, all api test already write in test-api.sh
npm test
```

## 📋 Quick Test Example

After deployment, update the API_BASE_URL in test-api.sh with your actual URL, then:

```bash
# 1. Register a user
curl -X POST "YOUR_API_URL/api/v1/register" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Doe", "email": "john@example.com"}'

# 2. Wait 5 seconds, then get user
curl "YOUR_API_URL/api/v1/user/john@example.com"

# 3. Create a board (use userId from step 2)
curl -X POST "YOUR_API_URL/api/v1/boards" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Board", "description": "Testing", "createdBy": "USER_ID"}'

# 4. Wait 5 seconds, then list boards
curl "YOUR_API_URL/api/v1/boards"
```

## ⚡ Key Features

- **Async Processing**: All create operations return immediately (202 status)
- **SNS**: User registration & message posting
- **SQS**: Board creation
- **DynamoDB**: Data storage with proper indexing
- **WebSocket**: Real-time message notifications
- **TypeScript**: Full type safety

## 🔧 Environment Variables

Create `.env` file with your AWS credentials:
```bash
AWS_DEFAULT_REGION=us-east-1
```

## 🧪 WebSocket Testing

1. Open `websocket-test.html` in browser
2. Enter WebSocket URL from deployment output
3. Subscribe to boards for real-time messages

## 🗂️ Architecture

```
API Gateway → Lambda → SNS/SQS → Processing Lambda → DynamoDB
                                                   ↓
WebSocket ← Notification Lambda ←─────────────────┘
```

## 🧹 Cleanup

```bash
npm run remove
```

This removes all AWS resources to avoid charges.

## 🧾 2026-11-21 — Making async registration safe: idempotency keys and the dead-letter queue

`POST /api/v1/register` returns 202 and processes the registration
asynchronously through SNS → Lambda → DynamoDB. That shape is fast,
but SNS and SQS deliver *at least once*: a retry, a timeout, or a
double-click can deliver the same registration twice. Without a guard,
the same user is written twice — or two welcome emails go out. Today
the API gets the two guards that make at-least-once delivery safe.

### 1. Idempotency key: one key, one write

- The client sends an `Idempotency-Key` header (a UUID it generates
  once per registration attempt) and retries with the *same* key.
- The processing Lambda writes to DynamoDB with a condition
  expression: `attribute_not_exists(idempotency_key)`. The first
  delivery wins; a duplicate delivery sees the key already exists
  and returns the stored result instead of writing again.
- Store the key with the user record (or in a small idempotency
  table with a TTL of 24 hours) so a retry hours later is still
  recognised. Without a TTL the table grows forever; 24 hours covers
  any realistic retry window for a registration.
- The API layer also returns the same 202 and the same
  `registration_id` for a repeated key, so the client cannot tell —
  and does not need to tell — whether this was the first attempt.

### 2. Dead-letter queue: let failures fail loudly, once

- The SQS queue behind board creation gets a DLQ with
  `maxReceiveCount` of 3: a message that fails processing three
  times moves to the DLQ instead of being retried forever.
- Set the queue's visibility timeout to at least 6× the Lambda
  timeout. If the Lambda times out at 30 s, a visibility timeout
  shorter than that makes the message visible again while the first
  attempt is still running — the same message is then processed
  twice at once.
- Alarm on the DLQ (any message in it) and redrive manually after
  fixing the cause. A DLQ you never look at is just a bin; a DLQ
  with an alarm is the difference between "one user failed to
  register" and "registrations have been silently failing all week".
- SNS fan-out for registration has no DLQ of its own — subscribe an
  SQS queue to the topic first, then let the Lambda read from that
  queue, so the same redrive pattern applies.

### Checklist before the next deploy

- [ ] `Idempotency-Key` accepted on `POST /api/v1/register` and
      stored with a 24-hour TTL
- [ ] Conditional write (`attribute_not_exists`) in the processing
      Lambda; duplicate delivery returns the stored result
- [ ] DLQ on the board-creation SQS queue, `maxReceiveCount` 3
- [ ] Visibility timeout ≥ 6× Lambda timeout
- [ ] CloudWatch alarm on DLQ depth > 0

### What I will add next in this file

- 2026-11-24: Cognito + JWT auth — who can register, and what a
  token proves at the API boundary.
- Later: CloudWatch dashboards for the async pipeline — queue
  depth, DLQ alarms, and end-to-end latency.

## 🧾 2026-11-23 — Request validation and error responses: one error shape for the whole API

Yesterday's idempotency and DLQ guards make the async pipeline
safe. Today the API boundary gets the same treatment: validate
every request before it touches SNS or SQS, and answer every
failure in one predictable shape, so clients never have to guess
what went wrong.

### One error shape, everywhere

```json
{
  "code": "VALIDATION_ERROR",
  "message": "email is not a valid email address",
  "correlationId": "9f2c4a1e-…",
  "details": [
    { "field": "email", "issue": "must be a valid email address" }
  ]
}
```

- `code` is a stable machine-readable string — clients switch on
  it, never on the message text.
- `message` is one human-readable sentence.
- `correlationId` is generated per request and logged with every
  downstream step, so a 400 at the edge can be traced through the
  SNS topic, the processing Lambda, and DynamoDB.
- `details` is present only for validation errors, one entry per
  bad field. Never include a stack trace, an SQL fragment, or an
  AWS error — internals stay in the server log (see the NodeJS
  error-handling notes, 2026-11-21).

### HTTP codes, by failure

| Situation | Code | `code` value |
|---|---|---|
| A field fails validation | 400 | `VALIDATION_ERROR` |
| Missing or invalid auth (once Cognito lands) | 401 | `UNAUTHORISED` |
| Valid request, not allowed | 403 | `FORBIDDEN` |
| Unknown user / board / message id | 404 | `NOT_FOUND` |
| Same idempotency key, different body | 409 | `IDEMPOTENCY_CONFLICT` |
| Anything unexpected server-side | 500 | `INTERNAL_ERROR`, generic message only |

### Validate at the boundary, with a schema

Validate in the API Lambda before publishing to SNS/SQS — a bad
request rejected in 10 ms costs nothing; a bad message that
travels the whole pipeline and dies in the processing Lambda
costs a DLQ entry and a redrive. In TypeScript, a Zod-style schema
keeps the rule next to the type:

```typescript
import { z } from "zod";

const RegisterUserSchema = z.object({
  name: z.string().trim().min(1).max(100),
  email: z.string().email().max(254),
});

const result = RegisterUserSchema.safeParse(body);
if (!result.success) {
  return errorResponse(400, "VALIDATION_ERROR", result.error.issues);
}
```

Rules that follow from this: trim strings before validating;
reject unknown fields rather than silently dropping them (a client
sending `emai` instead of `email` should get a 400, not a user
with no email); cap string lengths — DynamoDB and SNS both have
size limits, and an unbounded string is a cheap denial of service.

### Correlation ids through the async pipeline

The API generates the correlation id and passes it as an SNS/SQS
message attribute, not inside the payload. The processing Lambda
logs it with every step, and the 202 response returns it, so a
client that reports "my registration never arrived" gives support
one id to search in CloudWatch instead of a timestamp and a guess.
