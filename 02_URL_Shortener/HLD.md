# High-Level Design (HLD) for URL Shortener

Design a TinyURL/Bitly-style service where users can shorten URLs. Some users may even want to create their **own custom short links** (like `sho.rt/mybrand`).

The system also tracks how many times a link is clicked, for analytics and ads.

---

## 🎯 What Are We Trying to Build?

### Functional Stuff:
- Users can create short links (either random or custom).
- Anyone clicking that link should be redirected to the original URL.
- Track how many times each short link is used (daily, region-wise, etc.).
- URLs should **never expire**.
- Custom aliases can be **up to 16 characters**.

### Non-Functional Goals:
- Handle **100 million** new shortened URLs every month.
- Redirections should be super fast ⚡.
- Should scale easily and be reliable.
- Support basic analytics for ads and reporting.

---

## Main Components
### 1. API Gateway
- Entry point for users.
- Handles routing and basic stuff like rate limiting or logging.

### 2. URL Shortening Service
- Handles logic for creating short URLs.
- Checks if a custom alias is available.
- Generates a unique short code if none is provided.
- Saves { short_code → long_url } mapping to the DB.

### 3. Redirection Service
- When someone clicks a short link, this service:
   - Looks up the original URL from DB or cache.
   - Tracks analytics like timestamp, user-agent, location, etc.
   - Sends a 302 redirect to the browser.

### 4. Storage (Database)
- Stores mappings and metadata.
- Could use something like:
   - SQL for strong consistency (custom_alias should be unique).
   - NoSQL (like DynamoDB) for scale with auto-generated short codes.

### 5. Cache (e.g. Redis)
- For super fast lookups of hot/viral links.
- Avoids hitting DB for every redirect.

### 6. Analytics System
- Listens to redirect events.
- Aggregates data: clicks per day, geo, device type.
- Helps with reports and targeted ads.

## Database Tables (Simplified)
### URL Mapping Table

| Field        | Type         | Notes                       |
|--------------|--------------|-----------------------------|
| short_code   | string (PK)  | Max 16 chars, unique        |
| long_url     | text         | Original full URL           |
| created_at   | timestamp    | When link was created       |
| creator_user | string       | (Optional) user ID/email    |

---

## How Do We Generate Short Codes?
### For Custom Aliases
- Just check if it’s available in DB.
- If not → return an error.

### For Auto-Generated
- Base62 encoding (A-Z, a-z, 0-9) of unique ID.
- Can use:
   - DB auto-increment IDs
   - Hashing (e.g. MD5, SHA-256) + truncation
   - Snowflake ID generators

### Scale + Performance Plan
- Use Load Balancers to distribute traffic.
- Use Read Replicas on DB for redirection reads.
- CDNs or edge caches to speed up global redirects.
- Partition data (sharding) if we grow super big.

### Metrics to Track
- Clicks per short URL (per day, hour).
   - **Storage Suggestion:** 
      - Use a NoSQL store like **MongoDB**, **DynamoDB**, or **Redis Sorted Sets**.
      - You can use a time-series DB like **InfluxDB** or **ClickHouse** for efficient aggregation.
   - **Example Document (MongoDB):**
      ```json
      {
         "short_code": "coollink123",
         "date": "2025-06-21",
         "hour": 13,
         "clicks": 142
      }
      ```
- Traffic source (geo, device).
- Top-performing links (popular today).
   - **Storage Suggestion:**
      - Maintain a sorted set (Redis) or daily top-N summary in Mongo/ClickHouse.
      - Use Redis Sorted Sets for real-time ranking.
- Bot detection or suspicious activity.
   - **What to Track:**
      - Unusual spikes in click volume.
      - Same IP hitting multiple links rapidly.
      - User-agents commonly used by bots (like curl, python-requests, etc.).

   **Storage Suggestion:**
   - Store request logs in a NoSQL DB (MongoDB or Elasticsearch).
   - Run batch detection jobs or real-time rules.
   - Use services like Cloudflare Bot Management or integrate with reCAPTCHA.