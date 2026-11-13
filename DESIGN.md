# URL Shortener - System Design Document

## 1. System Architecture

### Overview
A scalable, production-grade URL shortener service built with Spring Boot, designed to handle high throughput with low latency.

### Architecture Components

```
┌─────────┐
│ Client  │ (Web/Mobile/API consumers)
└────┬────┘
     │ HTTP/REST
     ▼
┌─────────────────────────────────────┐
│         API Layer                   │
│  (Spring Boot REST Controllers)     │
│  - Request validation               │
│  - Rate limiting                    │
│  - Authentication/Authorization     │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│      Service Layer                  │
│  - URL Shortening Service           │
│  - URL Resolution Service           │
│  - Analytics Service                │
│  - Validation Service               │
└────┬────────────────────────────────┘
     │
     ├──────────────────┬──────────────────┐
     ▼                  ▼                  ▼
┌──────────┐      ┌──────────┐      ┌──────────┐
│  Redis   │      │Database  │      │ Analytics│
│  Cache   │      │(PostgreSQL│      │  Queue   │
│          │      │ /MySQL)   │      │(Kafka/   │
│  - Hot   │      │           │      │ RabbitMQ)│
│  URLs    │      │ - URL     │      │          │
│  - Rate  │      │   mappings│      │ - Click  │
│  limiting│      │ - Analytics│     │   events │
│          │      │   (cold)  │      │          │
└──────────┘      └──────────┘      └──────────┘
```

### Component Details

**Client**
- Web browsers, mobile apps, API consumers
- Makes requests to shorten URLs or resolve short codes

**API Layer**
- Spring Boot REST controllers
- Request validation and sanitization
- Rate limiting (per IP/user)
- Authentication/Authorization (JWT/OAuth2)
- Request/Response logging
- Error handling and standardized error responses

**Service Layer**
- Business logic encapsulation
- URL normalization and validation
- Short code generation
- Cache management
- Analytics event generation

**Cache (Redis)**
- **Hot URL storage**: Frequently accessed short codes → long URLs
- **Rate limiting**: Per-user/IP request counters
- **TTL management**: Cache expiration for stale data
- **Write-through cache**: Cache-aside pattern

**Database (PostgreSQL/MySQL)**
- **Primary storage**: All URL mappings (short code → long URL)
- **Analytics storage**: Historical click data (partitioned by date)
- **ACID compliance**: Data consistency guarantees
- **Replication**: Read replicas for scaling reads

**Analytics Flow**
- **Asynchronous processing**: Click events sent to message queue
- **Batch processing**: Consumer processes events in batches
- **Data warehouse**: Aggregated analytics stored separately
- **Real-time metrics**: Redis counters for live stats

---

## 2. REST API Specifications

### 2.1 Create Short URL

**HTTP Method:** `POST`

**Endpoint:** `/api/v1/urls/shorten`

**Request Body:**
```json
{
  "longUrl": "https://example.com/very/long/url/path",
  "customAlias": "optional-custom-alias",  // Optional
  "expirationDays": 365,                   // Optional, default: never expires
  "tags": ["tag1", "tag2"]                 // Optional
}
```

**Response (201 Created):**
```json
{
  "shortCode": "aB3xY9",
  "shortUrl": "https://short.ly/aB3xY9",
  "longUrl": "https://example.com/very/long/url/path",
  "createdAt": "2024-01-15T10:30:00Z",
  "expiresAt": "2025-01-15T10:30:00Z",
  "qrCode": "https://short.ly/qr/aB3xY9"  // Optional
}
```

**Error Responses:**
- `400 Bad Request`: Invalid URL format, custom alias already exists
- `401 Unauthorized`: Missing/invalid authentication
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error

**Purpose:** Convert a long URL into a short, shareable URL.

---

### 2.2 Resolve Short URL

**HTTP Method:** `GET`

**Endpoint:** `/api/v1/urls/{shortCode}`

**Path Parameters:**
- `shortCode`: The short code to resolve (e.g., "aB3xY9")

**Response (302 Found):**
```
Location: https://example.com/very/long/url/path
```

**Response (200 OK) - If `resolve=false` query param:**
```json
{
  "shortCode": "aB3xY9",
  "longUrl": "https://example.com/very/long/url/path",
  "createdAt": "2024-01-15T10:30:00Z",
  "expiresAt": "2025-01-15T10:30:00Z",
  "clickCount": 1234,
  "isActive": true
}
```

**Error Responses:**
- `404 Not Found`: Short code doesn't exist or expired
- `410 Gone`: URL was deleted/deactivated
- `500 Internal Server Error`: Server error

**Purpose:** Resolve short code to original URL and redirect, or return metadata.

---

### 2.3 Get URL Analytics

**HTTP Method:** `GET`

**Endpoint:** `/api/v1/urls/{shortCode}/analytics`

**Query Parameters:**
- `startDate` (optional): ISO 8601 date
- `endDate` (optional): ISO 8601 date
- `granularity` (optional): `hour`, `day`, `week`, `month` (default: `day`)

**Response (200 OK):**
```json
{
  "shortCode": "aB3xY9",
  "totalClicks": 1234,
  "uniqueClicks": 890,
  "timeSeries": [
    {
      "timestamp": "2024-01-15T00:00:00Z",
      "clicks": 45,
      "uniqueClicks": 32
    }
  ],
  "topReferrers": [
    {"referrer": "https://twitter.com", "clicks": 234},
    {"referrer": "direct", "clicks": 189}
  ],
  "topCountries": [
    {"country": "US", "clicks": 456},
    {"country": "UK", "clicks": 234}
  ],
  "topBrowsers": [
    {"browser": "Chrome", "clicks": 567},
    {"browser": "Safari", "clicks": 234}
  ]
}
```

**Error Responses:**
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized to view analytics
- `404 Not Found`: Short code doesn't exist

**Purpose:** Retrieve analytics data for a short URL.

---

### 2.4 Update Short URL

**HTTP Method:** `PUT`

**Endpoint:** `/api/v1/urls/{shortCode}`

**Request Body:**
```json
{
  "longUrl": "https://new-url.com",  // Optional
  "isActive": true,                   // Optional
  "expirationDays": 180               // Optional
}
```

**Response (200 OK):**
```json
{
  "shortCode": "aB3xY9",
  "longUrl": "https://new-url.com",
  "updatedAt": "2024-01-16T10:30:00Z"
}
```

**Error Responses:**
- `400 Bad Request`: Invalid data
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized
- `404 Not Found`: Short code doesn't exist

**Purpose:** Update URL metadata (destination, expiration, active status).

---

### 2.5 Delete Short URL

**HTTP Method:** `DELETE`

**Endpoint:** `/api/v1/urls/{shortCode}`

**Response (204 No Content)**

**Error Responses:**
- `401 Unauthorized`: Not authenticated
- `403 Forbidden`: Not authorized
- `404 Not Found`: Short code doesn't exist

**Purpose:** Soft delete or deactivate a short URL.

---

### 2.6 List User URLs

**HTTP Method:** `GET`

**Endpoint:** `/api/v1/urls`

**Query Parameters:**
- `page` (optional): Page number (default: 0)
- `size` (optional): Page size (default: 20, max: 100)
- `sort` (optional): Sort field (default: `createdAt`)
- `order` (optional): `asc` or `desc` (default: `desc`)
- `tags` (optional): Filter by tags (comma-separated)

**Response (200 OK):**
```json
{
  "content": [
    {
      "shortCode": "aB3xY9",
      "shortUrl": "https://short.ly/aB3xY9",
      "longUrl": "https://example.com/very/long/url/path",
      "createdAt": "2024-01-15T10:30:00Z",
      "clickCount": 1234,
      "isActive": true
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 150,
  "totalPages": 8
}
```

**Purpose:** Retrieve paginated list of URLs created by authenticated user.

---

### 2.7 Bulk Shorten URLs

**HTTP Method:** `POST`

**Endpoint:** `/api/v1/urls/bulk`

**Request Body:**
```json
{
  "urls": [
    {"longUrl": "https://example.com/1"},
    {"longUrl": "https://example.com/2", "customAlias": "custom2"}
  ]
}
```

**Response (200 OK):**
```json
{
  "results": [
    {
      "shortCode": "aB3xY9",
      "shortUrl": "https://short.ly/aB3xY9",
      "longUrl": "https://example.com/1",
      "status": "success"
    },
    {
      "longUrl": "https://example.com/2",
      "status": "error",
      "error": "Custom alias already exists"
    }
  ]
}
```

**Purpose:** Create multiple short URLs in a single request.

---

## 3. Database Schema

### 3.1 URL Mapping Table (`url_mappings`)

```sql
CREATE TABLE url_mappings (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL UNIQUE,
    long_url TEXT NOT NULL,
    user_id BIGINT,  -- NULL for anonymous users
    custom_alias VARCHAR(50) UNIQUE,  -- NULL if auto-generated
    is_active BOOLEAN DEFAULT TRUE,
    expires_at TIMESTAMP,  -- NULL for never-expiring URLs
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    click_count BIGINT DEFAULT 0,
    tags TEXT[],  -- Array of tags
    
    -- Indexes
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_custom_alias (custom_alias),
    INDEX idx_created_at (created_at),
    INDEX idx_expires_at (expires_at),
    INDEX idx_tags (tags) USING GIN,  -- PostgreSQL GIN index for array search
    
    -- Constraints
    CONSTRAINT chk_short_code_length CHECK (LENGTH(short_code) >= 4 AND LENGTH(short_code) <= 10),
    CONSTRAINT chk_url_format CHECK (long_url ~* '^https?://.+')
);

-- Unique constraint on short_code (already in UNIQUE, but explicit)
CREATE UNIQUE INDEX idx_short_code_unique ON url_mappings(short_code);
```

**Indexing Strategy:**
- **Primary Key**: `id` (B-tree, auto-indexed)
- **Unique Index**: `short_code` (B-tree) - Critical for lookups
- **Index**: `user_id` (B-tree) - For user URL listings
- **Index**: `custom_alias` (B-tree) - For custom alias lookups
- **Index**: `created_at` (B-tree) - For time-based queries
- **Index**: `expires_at` (B-tree) - For expiration cleanup jobs
- **GIN Index**: `tags` (PostgreSQL) - For tag-based filtering

---

### 3.2 Analytics Table (`url_analytics`)

```sql
CREATE TABLE url_analytics (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL,
    click_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,  -- IPv4/IPv6
    user_agent TEXT,
    referrer TEXT,
    country_code CHAR(2),  -- ISO 3166-1 alpha-2
    device_type VARCHAR(20),  -- mobile, desktop, tablet
    browser VARCHAR(50),
    os VARCHAR(50),
    
    -- Partitioning key (for time-based partitioning)
    click_date DATE GENERATED ALWAYS AS (DATE(click_timestamp)) STORED,
    
    -- Foreign key
    CONSTRAINT fk_short_code FOREIGN KEY (short_code) 
        REFERENCES url_mappings(short_code) ON DELETE CASCADE,
    
    -- Indexes
    INDEX idx_short_code_timestamp (short_code, click_timestamp),
    INDEX idx_click_date (click_date),
    INDEX idx_country_code (country_code)
) PARTITION BY RANGE (click_date);  -- Partition by date for scalability

-- Create monthly partitions (example)
CREATE TABLE url_analytics_2024_01 PARTITION OF url_analytics
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**Partitioning Strategy:**
- **Time-based partitioning**: Partition by `click_date` (monthly or weekly)
- **Benefits**: 
  - Faster queries on recent data
  - Easy archival of old partitions
  - Better index performance
  - Parallel query execution

**Indexing Strategy:**
- **Composite Index**: `(short_code, click_timestamp)` - For analytics queries
- **Index**: `click_date` - For partition pruning
- **Index**: `country_code` - For geographic analytics

---

### 3.3 Users Table (Optional - for authentication)

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);
```

---

## 4. URL Shortening Strategy

### 4.1 Why Base62?

**Base62 Encoding:**
- **Character Set**: `0-9`, `a-z`, `A-Z` (62 characters total)
- **URL-safe**: No special characters that need encoding
- **Case-sensitive**: Distinguishes between uppercase and lowercase
- **Compact**: More efficient than Base10 (decimal)

**Comparison:**
- **Base10 (decimal)**: 1,000,000 URLs = 7 characters
- **Base62**: 1,000,000 URLs = 4 characters (62^4 = 14,776,336)
- **Base64**: More compact but requires URL encoding (`+`, `/`, `=`)

**Example:**
- ID: `1,000,000` → Base62: `4c92` (4 chars)
- ID: `1,000,000,000` → Base62: `15ftGg` (6 chars)

---

### 4.2 ID Generation Strategy

**Option 1: Database Auto-Increment (Recommended for MVP)**
```java
// Pseudo-code
1. Insert long URL into database
2. Get auto-generated ID (e.g., 1,000,000)
3. Convert ID to Base62 (e.g., "4c92")
4. Check for collision (shouldn't happen with auto-increment)
5. Store short_code in database
```

**Pros:**
- Simple and reliable
- No collisions (sequential IDs)
- Easy to implement

**Cons:**
- Sequential IDs reveal total URL count
- Database becomes bottleneck at scale

**Option 2: Distributed ID Generation (Production Scale)**
```java
// Using Snowflake ID or similar
1. Generate distributed ID (e.g., Twitter Snowflake)
2. Convert to Base62
3. Check for collision (extremely rare)
4. Store in database
```

**Pros:**
- Scalable across multiple servers
- No single point of failure
- Non-sequential (better privacy)

**Cons:**
- More complex implementation
- Requires coordination service

**Option 3: Pre-generated Pool (High Performance)**
```java
// Pre-generate short codes in batches
1. Background job generates 10,000 Base62 codes
2. Store in Redis queue
3. Service pops code from queue when needed
4. If queue low, trigger batch generation
```

**Pros:**
- Fast (no computation during request)
- Can pre-validate uniqueness

**Cons:**
- Requires queue management
- Wasted codes if service shuts down

---

### 4.3 Collision Avoidance

**Strategy 1: Database Unique Constraint**
- `short_code` column has UNIQUE constraint
- Database enforces uniqueness
- Application handles `IntegrityConstraintViolationException`

**Strategy 2: Retry with New ID**
```java
// Pseudo-code
do {
    id = generateId();
    shortCode = base62Encode(id);
    try {
        saveToDatabase(shortCode, longUrl);
        break; // Success
    } catch (DuplicateKeyException e) {
        // Collision detected, retry with new ID
        id = generateNewId();
    }
} while (retries < MAX_RETRIES);
```

**Strategy 3: Hash-based with Collision Detection**
```java
// For custom aliases
1. User provides custom alias: "my-link"
2. Check if exists in database
3. If exists, return error or suggest alternative
4. If not, create mapping
```

**Collision Probability:**
- With 6-character Base62: 62^6 = 56,800,235,584 combinations
- Collision probability is negligible for auto-increment IDs
- For random generation: Use UUID or Snowflake to minimize collisions

---

## 5. Read & Write Flows

### 5.1 Shorten URL Flow (Write)

```
1. Client Request
   POST /api/v1/urls/shorten
   { "longUrl": "https://example.com/very/long/url" }

2. API Layer
   - Validate request
   - Validate URL format
   - Check rate limit (Redis)
   - Authenticate user (if required)
   - Sanitize input

3. Service Layer
   - Normalize URL (add https:// if missing, remove trailing slash)
   - Check if URL already shortened (optional: deduplication)
   - Generate short code:
     a. If custom alias provided: validate uniqueness
     b. Else: Generate from database ID or distributed ID
   - Convert ID to Base62
   - Check collision (database query or cache)

4. Database Write
   - Insert into url_mappings table
   - Get generated ID
   - If collision: retry with new ID

5. Cache Write (Redis)
   - Set key: "url:{shortCode}" → longUrl
   - Set TTL: Match expiration or default (7 days)
   - Set key: "url:meta:{shortCode}" → JSON metadata

6. Response
   - Return short URL to client
   - Status: 201 Created
```

**Optimization:**
- **Deduplication**: Check if long URL already exists (optional)
- **Async analytics**: Log creation event to queue
- **Batch writes**: For bulk operations

---

### 5.2 Redirect Flow (Read)

```
1. Client Request
   GET /api/v1/urls/{shortCode}
   (or direct: GET /{shortCode})

2. API Layer
   - Extract short code from path
   - Validate format (4-10 alphanumeric chars)
   - Check rate limit (Redis)

3. Cache Lookup (Redis) - L1 Cache
   - Key: "url:{shortCode}"
   - If found:
     a. Increment click counter (Redis atomic increment)
     b. Return 302 redirect to longUrl
     c. Async: Log click event to queue
     d. Return immediately (fast path)

4. Database Lookup - L2 Cache (if Redis miss)
   - Query url_mappings table by short_code
   - If found:
     a. Check if expired (expires_at < NOW)
     b. Check if active (is_active = TRUE)
     c. If valid:
        - Write to Redis cache (cache-aside pattern)
        - Increment click_count in database (async)
        - Log click event to analytics queue
        - Return 302 redirect
     d. If expired/inactive:
        - Return 410 Gone or 404 Not Found

5. Analytics Event (Asynchronous)
   - Extract metadata: IP, User-Agent, Referrer
   - Enrich: Geolocation (IP → country), Device type
   - Send to message queue (Kafka/RabbitMQ)
   - Consumer processes in batches:
     a. Insert into url_analytics table
     b. Update aggregated stats in Redis
     c. Update click_count in url_mappings (periodic batch)

6. Response
   - 302 Found with Location header
   - Or 404/410 if not found/expired
```

**Performance Optimizations:**
- **Cache-first**: 99%+ requests served from Redis
- **Warm cache**: Pre-load popular URLs
- **Async analytics**: Don't block redirect
- **Batch updates**: Aggregate click counts
- **CDN**: Cache redirects at edge (CloudFlare, CloudFront)

**Cache Strategy:**
- **TTL**: 7 days for active URLs, 1 day for expired
- **Eviction**: LRU (Least Recently Used)
- **Warm-up**: Pre-cache URLs with high click counts

---

### 5.3 Analytics Flow

```
1. Click Event Generation (Async)
   - Event: { shortCode, timestamp, IP, userAgent, referrer }
   - Send to message queue (Kafka topic: "url-clicks")

2. Event Enrichment (Consumer)
   - Resolve IP → Country (GeoIP database)
   - Parse User-Agent → Browser, OS, Device
   - Add metadata: country_code, browser, os, device_type

3. Batch Processing
   - Consumer processes events in batches (100-1000 events)
   - Insert into url_analytics table (bulk insert)
   - Update Redis counters:
     - "analytics:{shortCode}:total" → increment
     - "analytics:{shortCode}:{date}" → increment
     - "analytics:{shortCode}:country:{countryCode}" → increment

4. Aggregation (Periodic - Hourly/Daily)
   - Aggregate data by short_code, date, country, etc.
   - Store in analytics summary table (optional)
   - Update url_mappings.click_count

5. Query Analytics (API)
   - Check Redis for recent data (< 1 hour)
   - Query database for historical data
   - Merge and return to client
```

---

## 6. Additional Production Considerations

### Scalability
- **Horizontal scaling**: Stateless API servers behind load balancer
- **Database read replicas**: For analytics queries
- **Redis cluster**: For distributed caching
- **CDN**: For static assets and redirect caching

### Reliability
- **Database replication**: Master-slave setup
- **Redis persistence**: AOF + RDB snapshots
- **Health checks**: /health endpoint
- **Circuit breakers**: For external dependencies

### Security
- **Rate limiting**: Per IP/user (Redis-based)
- **URL validation**: Whitelist/blacklist domains
- **Malware scanning**: Check URLs against threat intelligence
- **HTTPS only**: Enforce secure connections
- **Input sanitization**: Prevent injection attacks

### Monitoring
- **Metrics**: Request rate, latency, error rate
- **Logging**: Structured logging (JSON)
- **Alerting**: Error rate thresholds, cache hit rate
- **Tracing**: Distributed tracing (Zipkin/Jaeger)

---

## 7. Technology Stack

- **Framework**: Spring Boot 3.x
- **Database**: PostgreSQL 14+ (or MySQL 8+)
- **Cache**: Redis 7+
- **Message Queue**: Kafka (or RabbitMQ)
- **Build Tool**: Maven or Gradle
- **Java Version**: Java 17+ (LTS)

---

## 8. Project Structure (Proposed)

```
url-shortener/
├── src/main/java/com/shortener/
│   ├── controller/        # REST controllers
│   ├── service/           # Business logic
│   ├── repository/        # Data access layer
│   ├── model/            # Entity classes
│   ├── dto/              # Data transfer objects
│   ├── config/           # Configuration classes
│   ├── util/             # Utilities (Base62, URL validation)
│   └── exception/        # Custom exceptions
├── src/main/resources/
│   ├── application.yml   # Configuration
│   └── db/migration/     # Flyway/Liquibase migrations
└── pom.xml               # Maven dependencies
```

---

This design provides a solid foundation for a production-grade URL shortener that can scale to handle millions of requests per day.

