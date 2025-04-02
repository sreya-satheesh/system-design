# URL Shortener

A URL Shortener is a service that converts long URLs into short, unique links while maintaining redirection to the original URL when accessed. Popular services like Bit.ly and TinyURL operate on this principle.

## Understanding the Problem Statement

### Functional Requirements

✅ Generate a short, unique URL for a given long URL.

✅ Retrieve and redirect users to the original long URL when the short URL is accessed.

✅ Ensure uniqueness of short URLs.

✅ Support high read traffic efficiently.

### Non-Functional Requirements

✅ Scalability – The system should handle millions of URL requests.

✅ Low Latency – URL redirection should be near-instantaneous.

✅ High Availability – The system should remain operational under high traffic.

✅ Persistence – Data should be stored reliably for future retrieval.

### Constraints & Assumptions

- URLs are assumed to be valid and unique per user request.

- The short URL should be 6-8 characters long.

- The system is expected to handle 100M+ URL mappings efficiently.

## High-Level System Design

### Architecture Overview

The system consists of:

1️⃣ API Layer – Handles URL shortening and redirection requests.

2️⃣ Database – Stores short URL → long URL mappings.

3️⃣ Cache – Improves read performance for frequently accessed URLs.

4️⃣ Load Balancer – Distributes traffic across multiple servers.

### Component Interaction

#### 📌 Shorten URL Request:

- User submits a long URL via an API.

- The system generates a unique short code.

- Data is stored in a database and optionally cached for quick retrieval.

#### 📌 Redirection Request:

- User enters the short URL in their browser.

- The system looks up the corresponding long URL in cache or database.

- If found, the user is redirected to the original URL.

## Database Design

Since we need fast lookups and scalability, we can use either SQL or NoSQL.

### SQL Database (Relational Approach - MySQL, PostgreSQL)

```
CREATE TABLE urls (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_url VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
- Pros: ACID compliance, structured queries.

- Cons: May not scale well for extremely high traffic.

### NoSQL (Key-Value Store - Redis, DynamoDB)

- Key: short_url (e.g., "abc123")

- Value: long_url (e.g., "https://www.example.com/blog/how-to-code")

- Pros: Fast reads, horizontally scalable.

- Cons: Less structured than SQL.

**Decision: If we expect billions of URLs and need fast access, NoSQL (Redis/DynamoDB) is preferred. Otherwise, SQL works for smaller-scale deployments.**

## Generating Unique Short URLs

To create a unique short URL, we use Base62 Encoding (characters a-z, A-Z, 0-9) since it efficiently compresses numerical IDs into short strings.

### Approach 1: Base62 Encoding (Recommended)

- Convert an incrementing ID into a Base62 string.
- Example:
    - ID 12345 → Base62 encoding → dnh
- Advantages:
    - ✅ Generates unique, short URLs.
    - ✅ No collisions, since ID is unique.

```
import string

BASE62_ALPHABET = string.ascii_letters + string.digits

def encode_base62(num):
    result = []
    while num:
        num, rem = divmod(num, 62)
        result.append(BASE62_ALPHABET[rem])
    return ''.join(result[::-1])

```

## Redirecting Users

When a user accesses a short URL, the system should:

- Check Cache (Redis): If the URL mapping exists, return immediately.

- Query Database (If not in cache): Retrieve the long URL and update cache.

- Redirect the user to the long URL.

### API Endpoint (Express.js Example)

```
app.get('/:shortUrl', async (req, res) => {
    const shortUrl = req.params.shortUrl;
    
    // 1. Check Cache (Redis)
    let longUrl = await redis.get(shortUrl);
    
    // 2. If not found in cache, fetch from database
    if (!longUrl) {
        const record = await db.findOne({ short_url: shortUrl });
        if (!record) return res.status(404).send('URL not found');
        longUrl = record.long_url;
        await redis.set(shortUrl, longUrl, 'EX', 86400);  // Cache for 24 hours
    }

    // 3. Redirect to Long URL
    res.redirect(longUrl);
});
```

**Caching in Redis significantly improves performance by avoiding frequent database lookups.**

## Scaling the System

### 1️⃣ Caching (Redis)

- Store frequently accessed URLs in Redis to reduce database reads.

- Expiry set to 24 hours to prevent stale data.

### 2️⃣ Load Balancer

- NGINX or AWS Load Balancer distributes traffic across multiple API servers.

- Prevents single-point failure and improves response time.

### 3️⃣ Database Partitioning

- Use sharding (partitioning URLs by hash of short code) to distribute load.

- Example: URLs starting with a-m → DB1, n-z → DB2.

### 4️⃣ URL Expiry Mechanism

- Store a TTL (Time-to-Live) field in the database.

- Periodically delete expired URLs to free up space.

## Extending the System

### Additional Features

✅ Custom Short URLs – Allow users to specify custom aliases (e.g., short.ly/myarticle).

✅ Analytics & Tracking – Track click counts, user locations, and timestamps.

✅ User Authentication – Users can manage their shortened URLs.

### Tech Stack Recommendations

- Backend : Node.js (Express), Python (Django, Flask)
- Database : MySQL/PostgreSQL OR Redis/DynamoDB
- Cache - Redis
- Load Balancer - NGINX, AWS ALB
- Deployment - AWS, GCP, Azure

## Quick Recap

**What is a URL shortener?** A service that converts long URLs into short, unique links while preserving redirection.

**How to ensure uniqueness?** Use Base62 encoding on an auto-incrementing ID or generate random strings with collision checks.

**How to retrieve URLs efficiently?** Use Redis caching to speed up lookups.

**How to scale the system?** Load balancers, caching, sharding, and database replication.

**How to handle analytics?** Track click counts and user data for reporting.