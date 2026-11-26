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

- 2026-11-25: Cognito + JWT auth — done, see the section below.
- Next: CloudWatch dashboards for the async pipeline — queue
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

## 🧾 2026-11-25 — Cognito + JWT auth: who can register, and what a token proves

Until now the API is open: anyone can register a user, create a
board or post a message. That is fine for a demo and wrong for
anything real. Today it gets identity: Amazon Cognito User Pool
for sign-up and sign-in, and JWT access tokens checked at the API
boundary.

### User Pool vs Identity Pool — pick the right one

- **User Pool** is a user directory: sign-up, sign-in, password
  reset, groups, and JWTs (id token, access token, refresh token).
  This is what the API needs.
- **Identity Pool** exchanges a token for temporary AWS
  credentials, so a client can call AWS services directly (upload
  to S3, for example). The API does not need it — clients talk to
  API Gateway, not to AWS directly. Adding it now would hand out
  AWS credentials for no reason.

### The flow

1. Client signs up (via the Cognito SDK or hosted UI), confirms
   email, signs in.
2. Cognito returns an **id token** (who the user is), an **access
   token** (what the client may do, 1 hour), and a **refresh token**
   (30 days, used to get new access tokens).
3. The client sends `Authorization: Bearer <access_token>` on
   every API call.
4. API Gateway's **JWT authorizer** verifies the token before the
   Lambda ever runs: signature against the pool's JWKS, `iss`
   matches the pool, `aud`/`client_id` matches the app client,
   `exp` not passed, `token_use` is `access`.
5. The Lambda receives the claims — `sub` (stable user id),
   `email`, `cognito:groups` — and uses them for authorisation.

### What a token proves — and what it does not

A valid access token proves the bearer authenticated as that user
at that time. It does **not** prove they may do this action.
Authorisation is still the API's job:

- Only the board creator may edit the board: check `sub` against
  the board's `createdBy`, in the Lambda, not in the frontend.
- Admin actions check `cognito:groups` contains `admin`. Groups
  are a claim in the token, so the check is one string comparison
  — but it must happen server-side, where the client cannot fake it.

### Verify properly, or do not verify at all

- Fetch the JWKS from
  `https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json`,
  cache the keys, and verify the RS256 signature. A token accepted
  without signature verification is just a JSON blob the client
  wrote itself.
- Check `iss`, `aud`, `exp` and `token_use` every time. The JWT
  authorizer does all of this — prefer it over hand-rolled
  verification in the Lambda.
- Short-lived access tokens (1 hour) plus refresh-token rotation:
  a stolen access token is useful for an hour at most, and a
  revoked refresh token stops working. Use revocation for logout
  and for banning a user.
- Do not store tokens in `localStorage` for a browser client —
  any script on the page can read them. Use an `httpOnly` cookie
  or keep them in memory.

### What changes in this API

| Endpoint | Before | After |
|---|---|---|
| `POST /api/v1/register` | Open | Stays open — registering *is* the point |
| `POST /api/v1/boards` | Open | Requires access token |
| `POST /api/v1/boards/{id}/messages` | Open | Requires access token |
| `GET /api/v1/boards`, `GET /.../messages` | Open | Stay open (public boards); per-board privacy comes later |
| `GET /api/v1/user/{email}` | Open | Requires token, and `sub` must own the record |

The 401/403 rows from the error table (2026-11-23) now have a
real meaning: 401 is "no valid token", 403 is "valid token, not
your board". The correlation id from that section still travels
with the request, so a rejected call is traceable end to end.

### Checklist before the next deploy

- [ ] User Pool + app client created, JWT authorizer on the
      protected routes
- [ ] `POST /boards` and `POST /messages` reject requests with no
      token (401) and with a token for a different user (403)
- [ ] `sub` checked against resource owner in the Lambda, not the
      client
- [ ] Refresh-token rotation on; revocation tested (sign out kills
      the session)

## 🧾 2026-11-26 — CloudWatch dashboards for the async pipeline: see the queue, not just the code

The 21 Nov section added idempotency keys and a DLQ; the 25 Nov
section added Cognito auth. All of that is invisible in production
without observability. An async pipeline fails quietly by design —
a message sits in a queue, a Lambda errors and retries, a user
never gets their welcome email, and nobody notices until support
gets the ticket. Today the pipeline gets eyes: one dashboard, four
alarms, and an end-to-end latency number.

### The four questions the dashboard must answer

1. **Is work flowing?** Messages published to SNS vs messages
   processed by the Lambda. If published climbs and processed
   stays flat, the pipeline is stuck.
2. **Is work stuck?** `ApproximateNumberOfMessagesVisible` on the
   board-creation SQS queue (queue depth) and
   `ApproximateAgeOfOldestMessage` (oldest message age). Depth
   tells you how much is waiting; age tells you how long the
   unluckiest message has waited. Age is the better alarm — depth
   can be healthy during a burst, age cannot.
3. **Is work failing?** Lambda `Errors` and `Throttles` per
   function, plus DLQ depth (`ApproximateNumberOfMessagesVisible`
   on the DLQ). Any message in the DLQ is an alarm, not a metric
   to watch — see the 21 Nov checklist.
4. **How slow is it, end to end?** From the API returning 202 to
   the user record becoming `active`. The correlation id from the
   23 Nov section is the join key: log it with a timestamp at
   publish time and at process time, and the difference is the
   latency. Without it you have two unrelated graphs and a guess.

### The dashboard (one `aws cloudwatch put-dashboard` call)

```json
{
  "widgets": [
    {
      "type": "metric", "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Queue depth + oldest message age",
        "metrics": [
          ["AWS/SQS", "ApproximateNumberOfMessagesVisible",
           "QueueName", "board-creation-queue"],
          [".", "ApproximateAgeOfOldestMessage",
           ".", "board-creation-queue", { "yAxis": "right" }]
        ],
        "view": "timeSeries", "stacked": false, "period": 300
      }
    },
    {
      "type": "metric", "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Lambda errors + throttles",
        "metrics": [
          ["AWS/Lambda", "Errors", "FunctionName", "process-registration"],
          [".", "Throttles", ".", "."],
          [".", "Errors", ".", "process-board"]
        ],
        "view": "timeSeries", "stacked": false, "period": 300
      }
    },
    {
      "type": "metric", "x": 0, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "DLQ depth (any message here = alarm)",
        "metrics": [
          ["AWS/SQS", "ApproximateNumberOfMessagesVisible",
           "QueueName", "board-creation-dlq"]
        ],
        "view": "timeSeries", "stacked": false, "period": 60
      }
    },
    {
      "type": "metric", "x": 12, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "Lambda duration p95 (ms)",
        "metrics": [
          ["AWS/Lambda", "Duration", "FunctionName", "process-registration",
           { "stat": "p95" }]
        ],
        "view": "timeSeries", "stacked": false, "period": 300
      }
    }
  ]
}
```

Why these four widgets and not more: a dashboard you never open
is decoration. Four widgets fit on one screen; each answers one
of the four questions above. Add widgets only when a real incident
needed a graph you did not have.

### Alarms — the dashboard is for humans, alarms are for the pager

| Alarm | Metric | Threshold | Why this value |
|---|---|---|---|
| DLQ not empty | DLQ `ApproximateNumberOfMessagesVisible` | > 0 for 1 min | Any DLQ message is a failed registration; see 21 Nov |
| Oldest message too old | Queue `ApproximateAgeOfOldestMessage` | > 300 s for 5 min | A registration should process in seconds, not minutes |
| Lambda errors | `Errors` summed per function | > 0 for 5 min | Errors in an async worker are invisible without this |
| Lambda throttles | `Throttles` summed per function | > 0 for 5 min | Throttling means the concurrency limit is hit — a capacity question, not a bug |

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name board-creation-dlq-not-empty \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=board-creation-dlq \
  --statistic Maximum --period 60 --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 --alarm-actions <sns-topic-arn>
```

Send alarms to an SNS topic (email for now; a PagerDuty/Opsgenie
integration later). An alarm with no notification is the DLQ
problem again — a bin nobody looks at.

### Cost and limits, honestly

- CloudWatch dashboards cost ~$3/month per dashboard; alarms
  ~$0.10/month each; metrics are mostly free at this scale (Lambda
  and SQS emit them by default). Observability for this pipeline is
  under $5/month — cheaper than one lost registration.
- `ApproximateNumberOfMessagesVisible` is approximate and sampled
  per minute; do not build exact-count logic on it. The alarm
  threshold (> 0) is safe because the DLQ should be empty, not
  because the number is precise.
- Log-based latency (correlation id timestamps) needs a log
  Insights query or a custom metric with an embedded metric format;
  that is the next step, not this one — the dashboard above works
  with default metrics only.

### Checklist before the next deploy

- [ ] Dashboard created from the JSON above (one screen, four widgets)
- [ ] All four alarms created, wired to the SNS topic, and tested
      (send one bad message to the DLQ on purpose, watch the alarm fire)
- [ ] Correlation id logged at publish and at process time
- [ ] Alarm test recorded: date, which alarm, time to notify


## 🧾 2026-11-27 — End-to-end latency as a real number: log-based metric + p95 alarm

The 26 Nov dashboard shows queue depth, Lambda errors and Lambda
duration — but not the number a user actually feels: from the API
returning 202 to the user record becoming `active`. Duration tells
you the worker was slow; it does not tell you the registration was
slow. The queue wait in between is invisible. Today that latency
becomes a real metric, built from the correlation id that has been
travelling with every request since 23 Nov.

### The raw material: two timestamps, one correlation id

The API Lambda already logs the correlation id at publish time; the
processing Lambda logs it at process time. Add the timestamp to
both, in the same shape:

```text
[9f2c4a1e-…] PUBLISHED service=register latency_stage=publish
[9f2c4a1e-…] PROCESSED service=register latency_stage=process
```

CloudWatch Logs Insights joins them:

```sql
fields @timestamp, correlationId, latency_stage
| filter service = "register"
| stats min(@timestamp) as first, max(@timestamp) as last
    by correlationId
| stats avg(last - first) as avg_ms, pct(last - first, 95) as p95_ms
```

That query is the answer, but it is a query — it only runs when
someone runs it. For an alarm, the latency has to be a metric. That
is what Embedded Metric Format (EMF) is for: log a JSON line and
CloudWatch turns it into a metric, no `PutMetricData` call, no
extra SDK, no extra cost beyond the log line itself.

### The processing Lambda emits the metric

At the end of a successful process — where the code already knows
the publish timestamp, because it arrived as an SQS message
attribute — log one EMF line:

```json
{
  "_aws": {
    "Timestamp": 1793412000000,
    "CloudWatchMetrics": [
      {
        "Namespace": "AsyncUserRegisterApi",
        "Dimensions": [["Service"]],
        "Metrics": [{ "Name": "EndToEndLatencyMs", "Unit": "Milliseconds" }]
      }
    ]
  },
  "Service": "register",
  "EndToEndLatencyMs": 2340
}
```

Rules for the EMF line, learned the hard way:

- **One metric per line, one line per registration.** EMF is a log
  format, not a batch API — a line with ten values is ten data
  points CloudWatch cannot attribute to one registration.
- **The timestamp is the process time, not the publish time.**
  The latency *value* is `processed_at - published_at`; the EMF
  `Timestamp` is when the metric was emitted. Mixing them up shifts
  every spike by the latency itself.
- **Dimensions stay small.** `Service=register` only. A dimension
  per user or per correlation id creates a metric per user — a
  cardinality explosion that costs real money ($0.30 per metric
  per month, times thousands of users).
- **Log the publish timestamp as a message attribute, not in the
  payload.** The payload is the business data; the attribute is
  the pipeline metadata. Same rule as the correlation id (23 Nov).

### Reading it: p50, p95, max — and what each one means

| Statistic | What it tells you |
|---|---|
| p50 (~2 s here) | The typical registration. If this climbs, the whole pipeline slowed down. |
| p95 (~8 s here) | The unlucky 5%. This is the number to alarm on — it is what a real user feels on a bad day. |
| max | One bad message. Useful for the post-incident read, useless as an alarm — one slow registration pages nobody. |

Expected shape for this pipeline: p50 of a couple of seconds
(SNS → Lambda → DynamoDB is fast when nothing is wrong), p95
under a minute even with a retry. If p95 is minutes, the queue is
the bottleneck, not the code — check the oldest-message-age alarm
from 26 Nov before touching the Lambda.

### The p95 alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name register-end-to-end-latency-p95-high \
  --namespace AsyncUserRegisterApi \
  --metric-name EndToEndLatencyMs \
  --dimensions Name=Service,Value=register \
  --statistic p95 --period 300 --threshold 60000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 --treat-missing-data notBreaching \
  --alarm-actions <sns-topic-arn>
```

- **Threshold 60 s for 1 period (5 min).** A registration should
  complete in seconds; a minute at p95 means the pipeline is
  backed up, not that one message was slow.
- **`--treat-missing-data notBreaching`.** No registrations means
  no metric, and no metric is not an incident — it is a quiet
  hour. The DLQ alarm from 26 Nov is the one that fires when work
  exists and is not moving; this one fires when work exists and is
  moving too slowly.

### Cost, honestly

One custom metric ($0.30/month) + one alarm ($0.10/month). The EMF
log lines cost the same as any other log line — nothing extra. The
whole observability stack for this pipeline is still under
$5/month, and it now answers all four questions from 26 Nov plus
the one that matters most: how slow was it, end to end.

### Checklist before the next deploy

- [ ] Publish timestamp sent as an SQS/SNS message attribute;
      correlation id logged at publish and at process time
- [ ] Processing Lambda logs one EMF line per successful
      registration (`EndToEndLatencyMs`, `Service=register`)
- [ ] p95 alarm created (60 s, 5 min), wired to the SNS topic,
      tested with a slow message on purpose
- [ ] Dashboard's fourth widget (Lambda duration p95) kept — it
      answers "was the worker slow", this metric answers "was the
      registration slow"; both are needed to tell them apart

### What I will add next in this file

- API Gateway throttling + usage plans, so one abusive client
  cannot fill the queue faster than the workers drain it.
