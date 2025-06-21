# High-Level Design (HLD) for Rate Limiter

Rate limiting helps us control how many requests a user or service can make in a given time window. This prevents abuse, protects our servers, and helps maintain fair usage.

---

## 🧠 Where Can You Implement a Rate Limiter?

It really depends on the use case, tech stack, and system design. Broadly, you can implement rate limiting in three places:

1. **Client-side:**  
   Not recommended! Users can tamper with the client, bypass the limits, or spoof requests.

2. **Server-side:**  
   A common approach. Rate limits are applied when the server processes a request.

3. **Middleware:**  
   Probably the best place. It sits between the client and your server. It blocks extra requests before they even reach your business logic.

> If you're using a microservice setup and already have things like authentication middleware, you can implement the rate limiter in a similar way.

---

## 🚦 Popular Rate Limiting Algorithms

### 1. **Token Bucket**

Used by Stripe!

- Imagine a bucket with a limited number of tokens.
- Every time a request comes in, it "takes" a token.
- If no token is available, the request is dropped.
- New tokens are added to the bucket at a fixed rate.

**Parameters:**
- `bucket size` — max number of tokens.
- `refill rate` — tokens added per second.

> Useful when you want to allow bursts, but still maintain an average rate.

---

### 2. **Leaky Bucket**

- Works like a queue (FIFO).
- Incoming requests are added to the queue.
- Requests are processed at a fixed, regular rate.
- If the queue is full, new requests are dropped (i.e., “leak”).

**Parameters:**
- `bucket size` — queue capacity.
- `outflow rate` — how fast requests are processed.

> Good for smoothing out traffic spikes.

---

### 3. **Sliding Window**

- Keeps track of timestamps of past requests.
- When a new request comes in:
  - Remove old timestamps.
  - Add new one.
  - If total timestamps are within the allowed count → accept request.
  - Else → reject.

> Usually implemented with Redis sorted sets for efficient timestamp tracking.

---

## 🧩 System Components for a Rate Limiter

1. **Config/Data Store** — where rules (limits per user, per route, etc.) are stored.
2. **Workers** — pull rate limit configs and keep them in memory/cache.
3. **Middleware** — applies the rules:
   - Fetches counters/timestamps from Redis.
   - Decides to allow or reject the request.

### If request is blocked:
- Option 1: Return `429 Too Many Requests`.
- Option 2: Push to a message queue to process later.

---

## 🏗 Scaling in a Distributed Setup

### 1. **Race Conditions**

Example: Two requests read the counter (say it's 3), each thinks it can proceed, both write back 4 — but actual should’ve been 5.

**Solution:** Use **Redis Lua scripts**:
- Atomic.
- Faster than locks.
- All commands inside the script run together.

---

### 2. **Synchronization**

With multiple rate limiter servers:

- Avoid sticky sessions (not scalable).
- Use a **centralized store** like Redis so all servers see the same data.

---

## 📈 Monitoring and Tuning

Once live, monitor things like:

- Success rate
- Number of requests blocked
- Latency
- Algorithm behavior during bursts (e.g. flash sale)

If too many valid requests are getting blocked:
- Loosen the rules.
- Or switch algorithms (e.g. Token Bucket handles bursts better than Leaky Bucket).

---

## ✅ Summary

| Algorithm       | Pros                          | Cons                          |
|----------------|-------------------------------|-------------------------------|
| Token Bucket    | Allows burst + steady rate    | Can be complex to tune        |
| Leaky Bucket    | Smooth, fixed rate            | Bursty traffic gets dropped   |
| Sliding Window  | Accurate per-window limiting  | May need more storage (timestamps) |

Choose based on traffic pattern, business use-case, and infrastructure.

---
