# Redis Cheatsheet

## Mental Model

Redis is an **in-memory data structure store** — not just a key-value cache. The key insight: Redis holds data in RAM with optional persistence, and exposes rich data structures (strings, lists, sets, sorted sets, hashes, streams) with atomic operations. Use Redis when you need microsecond latency, pub/sub messaging, rate limiting, session storage, leaderboards, or a distributed lock. It is single-threaded for commands (no lock contention) and handles hundreds of thousands of ops/sec on a single node.

---

## Install & Minimal Setup

```bash
# macOS
brew install redis
brew services start redis

# Ubuntu
sudo apt install redis-server
sudo systemctl start redis

# Docker
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine

# With password
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine \
  redis-server --requirepass mypassword

# CLI
redis-cli
redis-cli -h localhost -p 6379 -a mypassword

# Python
pip install redis
```

---

## Core Concepts

### 1. Key Naming Conventions

```bash
# Use colons as namespace separators — Redis has no folders
user:1001:profile
user:1001:sessions
session:abc123
cache:weather:CDMX
rate_limit:user:1001:api
leaderboard:game:2024
```

### 2. Strings — The Universal Type

```bash
SET key "value"
GET key                           # returns nil if not found
DEL key
EXISTS key                        # 1 if exists, 0 if not

# With expiry
SET session:abc123 "user_data" EX 3600      # expires in 3600 seconds
SET session:abc123 "user_data" PX 3600000   # expires in milliseconds
SET cache:key "data" EXAT 1893456000        # expires at Unix timestamp
TTL key                           # seconds remaining (-1 = no expiry, -2 = gone)
EXPIRE key 3600                   # set/update expiry on existing key
PERSIST key                       # remove expiry

# Atomic set-if-not-exists (distributed lock pattern)
SET lock:resource "owner_id" NX EX 30       # NX = only if not exists
SETNX key value                             # legacy form

# Counters
INCR counter                      # atomic increment by 1
INCRBY counter 5                  # increment by 5
DECR counter
DECRBY counter 5
INCRBYFLOAT price 1.50

# Bulk operations
MSET key1 "val1" key2 "val2" key3 "val3"
MGET key1 key2 key3               # returns array
```

### 3. Hashes — Objects / Records

```bash
# Store structured data — like a row in a table
HSET user:1001 name "Joshua" email "j@example.com" role "admin" age 28
HGET user:1001 name                   # "Joshua"
HMGET user:1001 name email            # ["Joshua", "j@example.com"]
HGETALL user:1001                     # all fields and values
HKEYS user:1001                       # ["name", "email", "role", "age"]
HVALS user:1001                       # ["Joshua", "j@example.com", ...]
HEXISTS user:1001 email               # 1
HDEL user:1001 age
HLEN user:1001                        # number of fields
HINCRBY user:1001 age 1               # increment a numeric field
```

### 4. Lists — Queues & Stacks

```bash
# Push and pop
RPUSH queue:jobs "job1" "job2" "job3"   # push to right (tail)
LPUSH queue:jobs "job0"                  # push to left (head)
RPOP queue:jobs                          # pop from right
LPOP queue:jobs                          # pop from left

# Blocking pop — waits until element available (for workers)
BRPOP queue:jobs 30           # block up to 30 seconds, returns [key, value]
BLPOP queue:jobs 0            # block forever

# Inspect without consuming
LRANGE queue:jobs 0 -1        # all elements (0 to last)
LRANGE queue:jobs 0 9         # first 10
LLEN queue:jobs
LINDEX queue:jobs 0           # element at index

# Trim — keep only a window
LTRIM log:recent 0 999        # keep last 1000 entries

# Patterns
RPUSH / LPOP  → queue (FIFO)
LPUSH / LPOP  → stack (LIFO)
```

### 5. Sets — Unique Collections

```bash
SADD tags:post:42 "python" "ai" "rag" "langchain"
SREM tags:post:42 "langchain"
SMEMBERS tags:post:42               # all members (unordered)
SISMEMBER tags:post:42 "python"     # 1 if member
SCARD tags:post:42                  # cardinality (count)

# Set operations
SUNION tags:post:42 tags:post:99    # union
SINTER tags:post:42 tags:post:99    # intersection
SDIFF  tags:post:42 tags:post:99    # difference (in 42, not in 99)

# Store result as a new set
SUNIONSTORE result:tags tags:post:42 tags:post:99
SINTERSTORE common:tags tags:post:42 tags:post:99

# Random
SRANDMEMBER tags:post:42            # random member (no removal)
SPOP tags:post:42                   # random member + remove
```

### 6. Sorted Sets — Leaderboards & Ranked Data

```bash
# Score + member — members ordered by score (float)
ZADD leaderboard:game 1520 "joshua"
ZADD leaderboard:game 1750 "alice" 1600 "bob" 1450 "carol"

# Ranking queries
ZRANGE leaderboard:game 0 -1 WITHSCORES    # all, low to high
ZREVRANGE leaderboard:game 0 9 WITHSCORES  # top 10, high to low
ZRANK leaderboard:game "joshua"            # 0-indexed rank (low to high)
ZREVRANK leaderboard:game "joshua"         # 0-indexed rank (high to low)
ZSCORE leaderboard:game "joshua"           # score of member

# Range by score
ZRANGEBYSCORE leaderboard:game 1500 1800             # members in score range
ZRANGEBYSCORE leaderboard:game -inf +inf LIMIT 0 10  # first 10 (pagination)

# Update score
ZINCRBY leaderboard:game 50 "joshua"      # add 50 to joshua's score
ZCARD leaderboard:game                    # total members
ZREM leaderboard:game "carol"
```

### 7. Pub/Sub — Messaging

```bash
# Subscribe (blocks, waiting for messages)
SUBSCRIBE channel:notifications
PSUBSCRIBE channel:*          # pattern subscribe

# Publish (from another client/process)
PUBLISH channel:notifications '{"type":"alert","msg":"System update"}'

# Unsubscribe
UNSUBSCRIBE channel:notifications
```

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Publisher
r.publish("channel:notifications", '{"type": "alert"}')

# Subscriber (blocking)
pubsub = r.pubsub()
pubsub.subscribe("channel:notifications")
for message in pubsub.listen():
    if message["type"] == "message":
        print(message["data"])
```

### 8. Streams — Persistent Message Log (Redis 5+)

```bash
# Append to stream
XADD events:user * action "login" user_id "1001" ip "10.0.0.1"
# * = auto-generated ID (timestamp-based)

# Read from stream
XRANGE events:user - +              # all messages
XRANGE events:user - + COUNT 10    # first 10
XREVRANGE events:user + - COUNT 5  # last 5

# Consumer groups — for distributed processing
XGROUP CREATE events:user workers $ MKSTREAM
XREADGROUP GROUP workers consumer1 COUNT 10 STREAMS events:user >
XACK events:user workers <message-id>   # acknowledge processed
```

---

## Python Patterns

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

# Basic string cache with TTL
def get_user(user_id: int) -> dict:
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    user = db.query_user(user_id)    # fetch from DB
    r.setex(cache_key, 3600, json.dumps(user))   # cache 1 hour
    return user

# Hash for session storage
def create_session(session_id: str, user_data: dict, ttl: int = 86400):
    key = f"session:{session_id}"
    r.hset(key, mapping=user_data)
    r.expire(key, ttl)

def get_session(session_id: str) -> dict:
    return r.hgetall(f"session:{session_id}")

# Rate limiting (sliding window)
def is_rate_limited(user_id: int, limit: int = 100, window: int = 60) -> bool:
    key = f"rate_limit:{user_id}"
    pipe = r.pipeline()
    now = time.time()
    pipe.zremrangebyscore(key, 0, now - window)    # remove old entries
    pipe.zadd(key, {str(now): now})                # add current request
    pipe.zcard(key)                                # count requests in window
    pipe.expire(key, window)
    results = pipe.execute()
    return results[2] > limit

# Distributed lock
import uuid
def acquire_lock(resource: str, ttl: int = 30) -> str | None:
    lock_key = f"lock:{resource}"
    token = str(uuid.uuid4())
    acquired = r.set(lock_key, token, nx=True, ex=ttl)
    return token if acquired else None

def release_lock(resource: str, token: str) -> bool:
    # Atomic check-and-delete via Lua script
    lua = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
    """
    result = r.eval(lua, 1, f"lock:{resource}", token)
    return bool(result)

# Pipeline — batch commands (reduce round trips)
def bulk_set(data: dict[str, str], ttl: int = 3600):
    pipe = r.pipeline()
    for key, value in data.items():
        pipe.setex(key, ttl, value)
    pipe.execute()   # sends all at once
```

---

## Most-Used Patterns Summary

| Use case | Data structure | Pattern |
|---|---|---|
| Cache | String | `SET key val EX ttl` |
| Session | Hash | `HSET session:id field val` + `EXPIRE` |
| Queue | List | `RPUSH` / `BLPOP` |
| Unique visitors | Set | `SADD` + `SCARD` |
| Leaderboard | Sorted Set | `ZADD` + `ZREVRANGE` |
| Rate limiting | Sorted Set | sliding window with `ZREMRANGEBYSCORE` |
| Pub/Sub | Pub/Sub | `PUBLISH` / `SUBSCRIBE` |
| Distributed lock | String | `SET NX EX` + Lua release |
| Event log | Stream | `XADD` + consumer groups |

---

## Gotchas

- **Redis is not durable by default** — data lives in RAM. Enable `AOF` (append-only file) or `RDB` snapshots for persistence, or accept that data can be lost on restart.
- **Key eviction** — when memory is full, Redis evicts keys based on the `maxmemory-policy`. Set `allkeys-lru` for a cache, `noeviction` for a primary store.
- **`KEYS *` in production** — blocks the server while scanning all keys. Use `SCAN` with a cursor instead.
- **Pub/Sub is not durable** — messages are lost if no subscriber is listening. Use Streams for reliable messaging.
- **Large keys** — a single string value > 10MB or a hash with millions of fields causes latency spikes. Split large keys.
- **N+1 with `GET` in loops** — use `MGET` or pipeline to batch reads.
- **Expiry on hash fields** — Redis doesn't support per-field TTL on hashes. Expire the whole key or use separate string keys.

---

## Quick Links

- [Redis Docs](https://redis.io/docs/)
- [Redis Command Reference](https://redis.io/commands/)
- [redis-py Docs](https://redis-py.readthedocs.io)
- [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) — free GUI client
- [Try Redis](https://try.redis.io) — browser-based playground
