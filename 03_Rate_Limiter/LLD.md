# Low-Level Design: Ticket Booking System

### Token Bucket Algorithm – LLD

```python
# Constants
BUCKET_SIZE = 10  # max tokens in bucket
REFILL_RATE = 1   # tokens per second

# Redis Keys: "bucket:<user_id>" → {tokens, last_refill_timestamp}

def token_bucket_limit(user_id):
    current_time = current_unix_time()
    bucket_key = f"bucket:{user_id}"
    
    # Load state from Redis
    bucket = redis.get(bucket_key) or {"tokens": BUCKET_SIZE, "last_refill": current_time}
    
    # Calculate time elapsed since last refill
    elapsed = current_time - bucket["last_refill"]
    new_tokens = int(elapsed * REFILL_RATE)
    
    # Add tokens back (without exceeding bucket size)
    bucket["tokens"] = min(BUCKET_SIZE, bucket["tokens"] + new_tokens)
    bucket["last_refill"] = current_time
    
    # Check if we have a token
    if bucket["tokens"] > 0:
        bucket["tokens"] -= 1
        redis.set(bucket_key, bucket, ex=60)
        return True  # request allowed
    else:
        return False  # rate limit hit
```

### Sliding Window Log Algorithm – LLD

```python
# Constants
WINDOW_SIZE = 60         # in seconds
MAX_REQUESTS = 5         # max requests per window

# Redis Key: "log:<user_id>" → Sorted Set of timestamps

def sliding_window_limit(user_id):
    current_time = current_unix_time()
    window_start = current_time - WINDOW_SIZE
    log_key = f"log:{user_id}"
    
    # Remove old timestamps
    redis.zremrangebyscore(log_key, 0, window_start)
    
    # Get current request count in window
    request_count = redis.zcard(log_key)
    
    if request_count < MAX_REQUESTS:
        # Add new request timestamp
        redis.zadd(log_key, {current_time: current_time})
        redis.expire(log_key, WINDOW_SIZE)
        return True  # allow
    else:
        return False  # block
```

### Leaky Bucket Algorithm - LLD

```python
# Constants
BUCKET_SIZE = 10        # max pending requests in queue
LEAK_RATE = 1           # requests per second
PROCESS_INTERVAL = 1    # check once per second

# Redis Key: "leaky:<user_id>" → List of timestamps

def leaky_bucket_limit(user_id):
    now = current_unix_time()
    key = f"leaky:{user_id}"

    # Leak old entries based on leak rate
    while True:
        oldest_timestamp = redis.lindex(key, 0)
        if not oldest_timestamp:
            break
        
        if now - int(oldest_timestamp) >= (1 / LEAK_RATE):
            redis.lpop(key)  # leak one request
        else:
            break

    queue_length = redis.llen(key)
    
    if queue_length < BUCKET_SIZE:
        redis.rpush(key, now)  # enqueue this request
        redis.expire(key, 60)  # expire after idle time
        return True  # allowed
    else:
        return False  # too many requests in queue
```

### Integration as Middleware

```python
def rate_limiter_middleware(request):
    user_id = request.user_id
    
    if not token_bucket_limit(user_id):  # or use sliding_window_limit / leaky_bucket_limit
        return Response(429, "Too Many Requests")
    
    return forward_to_backend(request)
```