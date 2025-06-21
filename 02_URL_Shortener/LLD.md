# Low-Level Design: URL Shortener

### 1. Models / Entities

```python
class URLMapping:
    def __init__(self, short_code: str, long_url: str, creator_user: str = None, created_at=None):
        self.short_code = short_code
        self.long_url = long_url
        self.creator_user = creator_user
        self.created_at = created_at or current_timestamp()


class ClickEvent:
    """
        Represents a click event handler (No-SQL).
        This class is responsible for managing and processing click events triggered by user interactions. It provides methods to handle the event logic and execute associated actions.
    """
    def __init__(self, short_code: str, timestamp: str, user_ip: str, device_type: str, country: str):
        self.short_code = short_code
        self.timestamp = timestamp
        self.user_ip = user_ip
        self.device_type = device_type
        self.country = country


class AggregatedMetrics:
    """ 
        (No-SQL)
        Documentation for AggregatedMetrics: AggregatedMetrics is a component responsible for collecting, aggregating, and providing statistical data related to the URL shortener system. 
        It tracks metrics such as the number of URLs shortened, the number of redirects, and other performance indicators.
    """
    def __init__(self, short_code: str, date: str, hour: int):
        self.short_code = short_code
        self.date = date
        self.hour = hour
        self.clicks = 0
        self.unique_visitors = set()
        self.device_stats = defaultdict(int)
        self.country_stats = defaultdict(int)
```

### 2. Services

```python
class URLShortenerService:
    def __init__(self, db, cache):
        self.db = db            # DB interface (SQL/NoSQL)
        self.cache = cache      # Redis or in-memory

    def create_short_url(self, long_url: str, custom_alias: str = None, user_id: str = None) -> str:
        if custom_alias:
            if self.db.exists(custom_alias):
                raise ValueError("Alias already exists")
            short_code = custom_alias
        else:
            short_code = self._generate_unique_code()

        mapping = URLMapping(short_code, long_url, user_id)
        self.db.insert(mapping)
        self.cache.set(short_code, long_url)

        return short_code

    def _generate_unique_code(self) -> str:
        while True:
            code = base62_encode(random.randint(1, 1e12))[:8]  # Limit to 8 chars
            if not self.db.exists(code):
                return code


class RedirectionService:
    def __init__(self, db, cache, analytics_logger):
        self.db = db
        self.cache = cache
        self.analytics_logger = analytics_logger

    def redirect(self, short_code: str, request) -> str:
        long_url = self.cache.get(short_code)

        if not long_url:
            record = self.db.get(short_code)
            if not record:
                raise Exception("Short code not found")
            long_url = record.long_url
            self.cache.set(short_code, long_url)

        # Fire-and-forget logging (Async)
        self.analytics_logger.log_event(short_code, request)

        return long_url


class AnalyticsLogger:
    def __init__(self, queue):
        self.queue = queue  # Kafka, RabbitMQ, or just append to DB

    def log_event(self, short_code: str, request):
        ip = request.ip_address
        ua = request.user_agent
        event = ClickEvent(short_code, ip, ua)
        self.queue.send(event)  # Or save to NoSQL DB directly


class MetricsAggregator:
    def __init__(self, db):
        self.db = db

    def aggregate_clicks(self, date: str):
        """
        The `aggregate_clicks` function is designed to process a list of click events and compute the total number of clicks for each unique item.
        """
        pass
```