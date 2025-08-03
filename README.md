# 📱 QR Code Authentication System

A high-concurrency QR Code authentication system designed to support millions of users for secure and fast check-in/out.


### 📚 Table of Contents

1. Introduction & Background

2. Feature Scope & Assumptions

3. Traffic & Capacity Estimation

4. High-Level Architecture

5. QR Code Format & Validation Logic

6. Scalability & Low Latency Strategies

7. Storage Design

8. Security Considerations

9. Observability & Deployment

10. Error Handling & User Experience

11. Testing & Validation

12. Trade-Off Analysis

13. Glossary


## 🧩 1. Introduction & Background

This system enables secure, fast, one-time QR code–based user authentication for clock-in/out and event check-ins. It aims to handle millions of daily scans with high concurrency and low latency, with global deployment capability.

#### Target Users: 
Mobile app users (staff, event attendees)

#### Stakeholders: 
Product, engineering, security, infra teams

#### Assumptions:

- Users scan QR codes via mobile app
- Each QR is valid for one scan and short TTL (~5 mins)
- System should prevent replay attacks, bot abuse, and ensure high availability

## 📦 2. Feature Scope & Assumptions

### In Scope:
- One-time QR authentication (stateless or JWT-based)
- Event ID–scoped validation
- Optional location/device checks
- Observability and fallback handling

### Out of Scope:
- Payment or identity verification
- Long-lived sessions or OAuth flows

### Design Constraints:
- Sub-100ms response time under high QPS
-High write-to-read ratio

## 📊 3. Traffic & Capacity Estimation

|Metric|Value|
|------|-----|
|Expected peak QPS|10,000 scans/sec|
|Daily active users|~1 million|
|Redis memory footprint|~500MB for UUIDs|
|Typical latency goal|< 50ms per request|
|Write:Read ratio|~10:1 (mostly write-heavy)|


## 🧱 4. High-Level Architecture
```
        [Mobile App]
             |
          [API Gateway]  ← Rate limiting, bot detection
             |
    +---------------------+
    |      QR Service     |  ← Core logic: validate, deduplicate, record
    +---------------------+
         |         |
     [Cache]   [Message Queue]
         |         |
     [Database]  [Log/Notification Service]
```
Optional:

Auth Service for user session/token validation

Global Load Balancer for multi-region deployment

## ⚙️ 5. QR Code Format & Validation Logic

QR Payload Format (Base64 or JWT):
```
{
  "uuid": "abc123",         // unique ID
  "exp": 1728000000,        // expiration timestamp (e.g. 5 min TTL)
  "event_id": "ev001",     // associated event
  "nonce": "randomString"   // replay protection
}
```
### One-Time Use Logic
- QR uuid stored in Redis with TTL (e.g. 5 mins)
- Use SETNX or Lua script to ensure atomic scan logic:
```
SETNX qr:uuid:abc123 "scanned"
```
- 1: first scan (valid)
- 0: already scanned (invalid)

### Prevents:

- Replays
- Double rewards
- Abuse of scan endpoints

## 🚀 6. Scalability & Low Latency Strategies
### Caching & Filtering
- Edge Cache/CDN for expired QR image or error page
- Redis with Bloom Filter to quickly reject invalid/expired codes
  - ✅ Fast "not-exist" judgment
  - ❌ May falsely assume existence
### Async Processing
- Kafka or Pub/Sub to decouple DB writes from QR validation
- Prevents latency spikes in peak traffic
### Rate Limiting (at Gateway)
- Redis + Lua script (atomic INCR + EXPIRE)
- Bucket by user IP, device ID, or session

## 🧮 7. Storage Design
### Redis (Volatile QR Records)
```
qr:uuid:{uuid} = {
  user_id,
  event_id,
  scan_time
} (TTL: 5 mins)
```
### Relational DB: scan_records

|Field|Type|Notes|
|-----|----|-----|
|id|bigint|PK|
|user_id|uuid||
|event_id|string|indexed/partitioned|
|scan_time|datetime||
|device_info|jsonb|optional|
|location|point|optional|

- Unique constraint: (user_id, event_id)
- Optional: Use DynamoDB + TTL for multi-region support

## 🔐 8. Security Considerations

### QR Code Protection

- Signed JWT (RS256 preferred for pub/private key separation)
- Short TTL (2–5 mins)
- Optional: Encrypt payload (AES) to obfuscate content

### Replay & Abuse Protection
- IP/device fingerprinting
- Location/geofence validation
- One-time use enforced by Redis atomic ops
  
### Transport Layer
- HTTPS-only
- Strict transport security headers

## 🔭 9. Observability & Deployment

Metrics & Tracing

OpenTelemetry: from app → gateway → service → Redis/DB

Prometheus + Grafana: scan volume, error rates, latency

Logging

Result (success/fail), error type, event_id

Can be shipped to ELK or Cloud Logging

Alerts

Scan failure spike

Redis/DB latency alerts

Bot spike alerts (from rate limiter logs)

Deployment

GCP/AWS ECS, GKE, or Lambda

Horizontal auto-scaling via CPU/RPS

Canary release via GitHub Actions

## ⛑ 10. Error Handling & User Experience

|Scenario|HTTP Status|Message|
|--------|-----------|-------|
|Already scanned|409|已完成打卡|
|QR expired|410|此 QR 已失效，請刷新畫面重試|
|Invalid format|400|無效的 QR Code|
|JWT verification fail|400|QR Code 無效|
|Redis/DB failure|500|系統忙碌，請稍後再試|

- Fallback UI with retry / support contact button|
- A/B test: UX tolerance vs. security friction

## 🧪 11. Testing & Validation

- Tools: wrk, locust
- Test TTL expiration & cleanup
- Simulate race condition with duplicate scans
- Monitor Redis memory & Bloom Filter false-positive rates
- Fault injection to test observability pipeline and alerting

## ⚖️ 12. Trade-Off Analysis

|Decision|Alternative|Reason|
|--------|-----------|------|
|Redis for QR dedup|DynamoDB|Lower latency, in-memory TTL support|
|Bloom Filter|DB-only check|Saves DB reads; minor false-positive risk|
|JWT w/ RS256|HS256 or UUID|Public/private key security|
|One-time UUID TTL|Long-lived token|Prevent replay attacks|
|Async via MQ|Synchronous write|Faster UX, better scalability|


