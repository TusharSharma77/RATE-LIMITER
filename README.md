# In-Memory LLM API Rate Limiter

An in-memory rate limiter for an LLM API gateway with support for two runtime-configurable rate-limiting strategies:

* **Policy A — Sliding Window**
* **Policy B — Token Bucket**

The limiter maintains independent state for every `(client_id, endpoint)` pair and returns whether a request is allowed, the remaining capacity, and the estimated time until capacity becomes available.

## Problem

LLM APIs need rate limiting to prevent excessive usage and protect backend resources.

Build an in-memory rate limiter that supports:

1. Multiple clients and endpoints.
2. Two different rate-limiting strategies.
3. Per-client, per-endpoint state isolation.
4. Thread-safe request processing.
5. Consistent decision information for every request.

The main API is:

```text
check(
    client_id,
    endpoint,
    limit,
    window_ms,
    capacity,
    refill_rate
)
```

It returns:

```text
(allowed, remaining, reset_after_ms)
```

Where:

* `allowed` — whether the request is accepted.
* `remaining` — remaining capacity after the current decision.
* `reset_after_ms` — estimated milliseconds until capacity becomes available again.

---

## Policy A — Sliding Window

Policy A allows at most `limit` accepted requests during the most recent `window_ms` milliseconds.

For every `(client_id, endpoint)` pair, accepted request timestamps are stored.

### Example

```text
limit = 2
window = 10 seconds
```

Requests:

```text
t = 0s      → allowed
t = 3s      → allowed
t = 3.5s    → rejected
```

At `t = 10s`, the request from `t = 0s` expires from the window and capacity becomes available.

### State

```text
(client_id, endpoint)
        ↓
Deque of accepted timestamps
```

Expired timestamps are removed before every decision.

Rejected requests are not added to the history.

---

## Policy B — Token Bucket

Policy B maintains a token bucket for every `(client_id, endpoint)` pair.

The bucket:

* Starts with `capacity` tokens.
* Consumes one token for every accepted request.
* Refills continuously at `refill_rate` tokens per second.
* Never exceeds `capacity`.
* Allows fractional tokens internally.

### Example

```text
capacity = 5
refill_rate = 2 tokens/second
```

Initial state:

```text
5 tokens
```

After one accepted request:

```text
4 tokens
```

After `0.5` seconds:

```text
4 + (0.5 × 2)
= 5 tokens
```

If the bucket contains `0.4` tokens:

```text
Required = 1 - 0.4
         = 0.6 tokens

At 2 tokens/second:

0.6 / 2
= 0.3 seconds
= 300 ms
```

The request is rejected and `reset_after_ms` is approximately `300`.

---

## State Isolation

Each client and endpoint has independent rate-limit state.

For example:

```text
user1 + /chat
user1 + /completion
user2 + /chat
```

These are treated as three separate rate-limit buckets/windows.

One client's requests do not affect another client's capacity.

---

## Thread Safety

The limiter uses a lock around the complete `check()` operation.

This ensures that:

* State updates are atomic.
* Two concurrent requests cannot consume the same capacity.
* Aggregate statistics remain consistent.

---

## Statistics

The limiter exposes aggregate statistics:

```text
{
    "strategy": "...",
    "total_allowed": 0,
    "total_rejected": 0,
    "active_keys": 0
}
```

### Fields

| Field            | Description                                      |
| ---------------- | ------------------------------------------------ |
| `strategy`       | Currently active rate-limiting strategy          |
| `total_allowed`  | Total number of accepted requests                |
| `total_rejected` | Total number of rejected requests                |
| `active_keys`    | Number of tracked `(client_id, endpoint)` states |

---

## Decision Format

Every request returns:

```text
(allowed, remaining, reset_after_ms)
```

### Accepted request

```text
(True, remaining, reset_after_ms)
```

### Rejected request

```text
(False, 0, reset_after_ms)
```

Rejected requests do not consume capacity.

---

## Design

```text
                ┌─────────────────┐
                │      check()    │
                └────────┬────────┘
                         │
                  strategy selection
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
      Sliding Window            Token Bucket
        Policy A                  Policy B
             │                       │
             ▼                       ▼
       Timestamps                 Tokens
         Deque                   + Time
             │                       │
             └───────────┬───────────┘
                         ▼
              (allowed, remaining,
               reset_after_ms)
```

---

## Technologies

* Python
* Standard Library
* `threading`
* `collections.deque`
* `typing`
* In-memory state management

No external database or caching system is required.

---

## Files

```text
.
├── rate_limiter.py
├── REFLECTION.essay
└── README.md
```

---

## Running

Clone the repository:

```bash
git clone <your-repository-url>
cd <repository-name>
```

Run the implementation:

```bash
python rate_limiter.py
```

---

## Key Concepts

This implementation demonstrates:

* Sliding Window Rate Limiting
* Token Bucket Rate Limiting
* Per-key state management
* Thread-safe state updates
* Time-based expiration
* Continuous token refill
* API gateway rate-limiting concepts
* In-memory data structures

## License

This project is intended for educational and technical demonstration purposes.
