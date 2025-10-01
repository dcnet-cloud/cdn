# Multi-Tenant CDN - Basic Scenarios

Tài liệu này mô tả các kịch bản cơ bản và phổ biến nhất trong vận hành Multi-Tenant CDN.

---

## Scenario 1: First Request from New Customer

### Context
- Customer mới vừa được onboard
- Chưa có cache nào
- User đầu tiên truy cập nội dung

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ T0: Customer "acme-corp" vừa được thêm vào hệ thống             │
│                                                                  │
│ Config:                                                          │
│ {                                                                │
│   "acme-corp": {                                                │
│     "origin_url": "https://cdn.acme.com",                       │
│     "custom_domain": "cdn.acme-corp.net",                       │
│     "cache_ttl": 3600                                           │
│   }                                                              │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘

T1: First User Request
┌──────────────┐
│ User Browser │
└──────┬───────┘
       │
       │ GET https://cdn.acme-corp.net/logo.png
       ▼

┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Step 1: Identify Customer                                       │
│   Host: cdn.acme-corp.net                                       │
│   → Lookup in customers.json                                   │
│   → Found: acme-corp                                           │
│                                                                 │
│ Step 2: Generate Cache Key                                      │
│   cache_key = "acme-corp:/logo.png"                           │
│                                                                 │
│ Step 3: Check Cache                                             │
│   /cache/**/acme-corp:/logo.png                                │
│   Result: NOT FOUND (first request)                            │
│                                                                 │
│ Step 4: Fetch from Origin                                       │
│   proxy_pass https://cdn.acme.com/logo.png                     │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ HTTP GET /logo.png
                ▼
┌────────────────────────────────────────────────────────────────┐
│            Customer Origin (cdn.acme.com)                       │
│                                                                 │
│ Processing request...                                           │
│ Time: 250ms (cold start, database lookup)                      │
│                                                                 │
│ Response:                                                       │
│ HTTP/1.1 200 OK                                                │
│ Content-Type: image/png                                        │
│ Content-Length: 45230                                          │
│ Cache-Control: public, max-age=3600                            │
│                                                                 │
│ [PNG binary data...]                                           │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response (250ms)
                ▼
┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Step 5: Store in Cache                                          │
│   Write to: /cache/XX/YY/hash(acme-corp:/logo.png)            │
│   Metadata:                                                     │
│     - Key: acme-corp:/logo.png                                 │
│     - Size: 45230 bytes                                        │
│     - TTL: 3600s (expires at T1 + 1 hour)                      │
│     - Created: 2025-10-01 10:00:00                             │
│                                                                 │
│ Step 6: Return to User                                          │
│   X-Cache-Status: MISS                                         │
│   Total Time: 250ms                                            │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response (250ms total)
                ▼
┌──────────────────┐
│   User Browser   │
│                  │
│ ✓ Received logo  │
│ ✓ Rendered       │
└──────────────────┘

Result:
✓ User experience: 250ms (slow, but acceptable for first request)
✓ Cache populated
✓ Next requests will be fast
```

---

## Scenario 2: Subsequent Cached Request

### Context
- Same content đã được request trước đó
- Cache vẫn còn valid (chưa expire)

### Flow Diagram

```
T2: Second User Request (5 seconds later)
┌──────────────┐
│ User Browser │
│  (Different  │
│    User)     │
└──────┬───────┘
       │
       │ GET https://cdn.acme-corp.net/logo.png
       ▼

┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Step 1: Identify Customer                                       │
│   Host: cdn.acme-corp.net                                       │
│   → Lookup: acme-corp ✓                                        │
│                                                                 │
│ Step 2: Generate Cache Key                                      │
│   cache_key = "acme-corp:/logo.png"                           │
│                                                                 │
│ Step 3: Check Cache                                             │
│   /cache/**/acme-corp:/logo.png                                │
│                                                                 │
│   ┌─────────────────────────────────┐                         │
│   │ Cache Entry Found!              │                         │
│   │ - Key: acme-corp:/logo.png      │                         │
│   │ - Age: 5s                       │                         │
│   │ - TTL remaining: 3595s          │                         │
│   │ - Valid: YES                    │                         │
│   └─────────────────────────────────┘                         │
│                                                                 │
│ Step 4: Serve from Cache (NO origin request)                   │
│   Read from disk: ~5ms                                         │
│   X-Cache-Status: HIT                                          │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response (5ms - FAST!)
                ▼
┌──────────────────┐
│   User Browser   │
│                  │
│ ✓ Received logo  │
│ ✓ Rendered       │
│ ✓ Super fast!    │
└──────────────────┘

Comparison:
┌──────────────────────────────────────┐
│ First Request:  250ms (MISS)         │
│ Second Request:   5ms (HIT)          │
│ Speedup:         50x faster!         │
└──────────────────────────────────────┘

Origin Server Impact:
- First request:  1 request to origin
- Next 1000 requests in 1 hour: 0 requests to origin
- Origin load reduction: 99.9%
```

---

## Scenario 3: Multiple Customers, Same URI

### Context
- 2 customers khác nhau
- Request cùng URI path: `/logo.png`
- Nhưng đây là nội dung khác nhau (different origins)

### Flow Diagram

```
Parallel Timeline:

User A requests Customer A's logo:
┌──────────────┐
│   User A     │
└──────┬───────┘
       │ GET https://cdn.customer-a.com/logo.png
       ▼
┌────────────────────────────────────────────────────────────────┐
│                   Edge Node (Shared)                            │
│                                                                 │
│ Request A Processing:                                           │
│ ┌────────────────────────────────────────────────────────────┐│
│ │ Host: cdn.customer-a.com                                    ││
│ │ Customer: customer-a                                        ││
│ │ Cache Key: "customer-a:/logo.png"  ◄─── Note prefix!      ││
│ │                                                             ││
│ │ Cache Check: MISS                                           ││
│ │ Fetch from: https://origin-a.com/logo.png                  ││
│ │ Store with key: customer-a:/logo.png                       ││
│ │ Return: [Customer A's blue logo]                           ││
│ └────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────┘

(1 second later)

User B requests Customer B's logo:
┌──────────────┐
│   User B     │
└──────┬───────┘
       │ GET https://cdn.customer-b.com/logo.png
       ▼
┌────────────────────────────────────────────────────────────────┐
│                   Edge Node (Same Edge Node)                    │
│                                                                 │
│ Request B Processing:                                           │
│ ┌────────────────────────────────────────────────────────────┐│
│ │ Host: cdn.customer-b.com                                    ││
│ │ Customer: customer-b                                        ││
│ │ Cache Key: "customer-b:/logo.png"  ◄─── Different prefix! ││
│ │                                                             ││
│ │ Cache Check: MISS (different key!)                         ││
│ │ Fetch from: https://origin-b.com/logo.png                  ││
│ │ Store with key: customer-b:/logo.png                       ││
│ │ Return: [Customer B's red logo]                            ││
│ └────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────┘

Cache State After Both Requests:
┌────────────────────────────────────────────────────────────────┐
│                      /cache/ directory                          │
│                                                                 │
│  Entry 1:                                                       │
│  ┌──────────────────────────────────────────────────┐         │
│  │ Key: customer-a:/logo.png                        │         │
│  │ Content: [Blue logo - 45KB]                      │         │
│  │ Origin: https://origin-a.com                     │         │
│  └──────────────────────────────────────────────────┘         │
│                                                                 │
│  Entry 2:                                                       │
│  ┌──────────────────────────────────────────────────┐         │
│  │ Key: customer-b:/logo.png                        │         │
│  │ Content: [Red logo - 38KB]                       │         │
│  │ Origin: https://origin-b.com                     │         │
│  └──────────────────────────────────────────────────┘         │
│                                                                 │
│  ✓ No collision - different keys                               │
│  ✓ Isolated - customers don't see each other's content        │
└────────────────────────────────────────────────────────────────┘

Key Insight:
┌────────────────────────────────────────────────────────────────┐
│ Without customer_id prefix:                                     │
│   Cache key: /logo.png (COLLISION! Wrong logo served)          │
│                                                                 │
│ With customer_id prefix:                                        │
│   Customer A: customer-a:/logo.png ✓                           │
│   Customer B: customer-b:/logo.png ✓                           │
│   → Perfect isolation!                                          │
└────────────────────────────────────────────────────────────────┘
```

---

## Scenario 4: Cache Expiration & Refresh

### Context
- Content đã được cache
- TTL expire
- Next request triggers refresh

### Flow Diagram

```
Timeline:

T0 (10:00:00): First Request
┌──────────────────────────────────────────────────────────────┐
│ User requests: /data.json                                     │
│ Cache: MISS                                                   │
│ Fetch from origin → Store in cache                           │
│ Cache entry:                                                  │
│   - Key: customer-a:/data.json                               │
│   - TTL: 3600s (1 hour)                                      │
│   - Expires at: 11:00:00                                     │
└──────────────────────────────────────────────────────────────┘

T1 (10:30:00): Request during valid period
┌──────────────────────────────────────────────────────────────┐
│ User requests: /data.json                                     │
│ Cache age: 1800s (30 minutes)                                │
│ TTL remaining: 1800s                                         │
│ Status: VALID                                                │
│ Result: HIT (serve from cache) ✓                             │
└──────────────────────────────────────────────────────────────┘

T2 (11:00:01): Request after expiration
┌──────────────────────────────────────────────────────────────┐
│                     Edge Node                                 │
│                                                               │
│ User requests: /data.json                                     │
│                                                               │
│ Step 1: Check Cache                                           │
│   Key: customer-a:/data.json                                 │
│   Found: YES                                                 │
│   Age: 3601s                                                 │
│   TTL: 3600s                                                 │
│   Status: EXPIRED ⏰                                          │
│                                                               │
│ Step 2: Cache Invalid - Must Revalidate                       │
│   Option A: If-Modified-Since request to origin              │
│   Option B: Fetch fresh copy                                 │
│                                                               │
│ Step 3: Fetch from Origin                                     │
│   GET https://origin-a.com/data.json                         │
│   If-Modified-Since: Mon, 01 Oct 2025 10:00:00 GMT          │
│                                                               │
└───────────────┬───────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────────┐
│              Customer Origin                                  │
│                                                               │
│ Check if file modified since 10:00:00                         │
│                                                               │
│ Case A: File NOT modified                                     │
│ ┌────────────────────────────────────────────────┐          │
│ │ HTTP/1.1 304 Not Modified                      │          │
│ │ → Edge keeps existing cached content           │          │
│ │ → Update TTL to new 3600s                      │          │
│ │ → Serve stale content (still valid)            │          │
│ └────────────────────────────────────────────────┘          │
│                                                               │
│ Case B: File WAS modified                                     │
│ ┌────────────────────────────────────────────────┐          │
│ │ HTTP/1.1 200 OK                                │          │
│ │ Content: [New updated data]                    │          │
│ │ → Edge replaces old cached content             │          │
│ │ → Store new content with fresh TTL             │          │
│ │ → Serve new content                            │          │
│ └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘

Visual Timeline:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ 10:00  10:15  10:30  10:45  11:00  11:15  11:30               │
│   │     │     │     │     │     │     │                        │
│   │◄────────── TTL = 3600s ─────►│                            │
│   │                              │                             │
│   ▼                              ▼                             │
│ First                         Expires                          │
│ Request                       Must refresh                     │
│ (MISS)                        (MISS or                         │
│                                REVALIDATE)                     │
│                                                                 │
│        │◄── All HITs ──►│                                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Cache Behavior:
┌────────────────────────────────────────────────────────────────┐
│ Time    Cache Age    Status         Action                     │
├────────────────────────────────────────────────────────────────┤
│ 10:00   0s          MISS           Fetch from origin           │
│ 10:30   1800s       HIT (valid)    Serve from cache           │
│ 10:59   3540s       HIT (valid)    Serve from cache           │
│ 11:00   3600s       EXPIRED        Revalidate with origin     │
│ 11:01   3660s       STALE          Fetch fresh from origin    │
└────────────────────────────────────────────────────────────────┘
```

---

## Scenario 5: Admin Purges Customer Cache

### Context
- Customer update nội dung ở origin
- Muốn xóa cache để users thấy version mới
- Admin thực hiện cache purge

### Flow Diagram

```
Initial State:
┌────────────────────────────────────────────────────────────────┐
│                  Cache Contents                                 │
│                                                                 │
│ customer-a:/logo.png      (cached 10 minutes ago)             │
│ customer-a:/style.css     (cached 5 minutes ago)              │
│ customer-a:/script.js     (cached 15 minutes ago)             │
│ customer-b:/logo.png      (cached 20 minutes ago)             │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Step 1: Customer Updates Origin Content
┌────────────────────────────────────────────────────────────────┐
│              Customer A Origin Server                           │
│                                                                 │
│ Admin updates logo.png:                                         │
│ Old: blue-logo.png (45KB)                                      │
│ New: red-logo.png (52KB)                                       │
│                                                                 │
│ Problem: Edge nodes still serving old blue logo from cache!    │
└────────────────────────────────────────────────────────────────┘

Step 2: Admin Initiates Cache Purge
┌──────────────┐
│    Admin     │
│   Terminal   │
└──────┬───────┘
       │
       │ curl -X DELETE http://admin-api:9000/api/customers/customer-a/cache
       ▼
┌────────────────────────────────────────────────────────────────┐
│                    Admin API                                    │
│                                                                 │
│ DELETE /api/customers/customer-a/cache                         │
│                                                                 │
│ Step 1: Validate customer exists                               │
│   customer_id = "customer-a" ✓                                │
│                                                                 │
│ Step 2: Find cache files                                        │
│   Scan /cache/ directory                                       │
│   Pattern: customer-a:*                                        │
│                                                                 │
│   Found files:                                                 │
│   ┌─────────────────────────────────────────┐                │
│   │ /cache/a1/b2/hash1 → customer-a:/logo.png              │
│   │ /cache/c3/d4/hash2 → customer-a:/style.css             │
│   │ /cache/e5/f6/hash3 → customer-a:/script.js             │
│   └─────────────────────────────────────────┘                │
│                                                                 │
│   NOT purged (different customer):                             │
│   ┌─────────────────────────────────────────┐                │
│   │ /cache/g7/h8/hash4 → customer-b:/logo.png (skip)       │
│   └─────────────────────────────────────────┘                │
│                                                                 │
│ Step 3: Delete cache files                                      │
│   rm /cache/a1/b2/hash1  ✓                                    │
│   rm /cache/c3/d4/hash2  ✓                                    │
│   rm /cache/e5/f6/hash3  ✓                                    │
│                                                                 │
│ Step 4: Optionally notify edge nodes                           │
│   (If using shared dict, update immediately)                   │
│                                                                 │
│ Response:                                                       │
│ {                                                               │
│   "message": "Cache purged successfully",                      │
│   "customer_id": "customer-a",                                 │
│   "purged_files": 3                                            │
│ }                                                               │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response
                ▼
┌──────────────────┐
│      Admin       │
│                  │
│ ✓ Purge complete │
│ ✓ 3 files deleted│
└──────────────────┘

Cache State After Purge:
┌────────────────────────────────────────────────────────────────┐
│                  Cache Contents                                 │
│                                                                 │
│ customer-a:/logo.png      ✗ DELETED                            │
│ customer-a:/style.css     ✗ DELETED                            │
│ customer-a:/script.js     ✗ DELETED                            │
│ customer-b:/logo.png      ✓ Still cached (not affected)        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Step 3: Next User Request
┌──────────────┐
│     User     │
└──────┬───────┘
       │ GET https://cdn.customer-a.com/logo.png
       ▼
┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Cache key: customer-a:/logo.png                                │
│ Cache check: NOT FOUND (was purged)                            │
│                                                                 │
│ Action: MISS → Fetch from origin                               │
│   GET https://origin-a.com/logo.png                            │
│   Response: [NEW red logo - 52KB]                              │
│                                                                 │
│ Store in cache with fresh TTL                                   │
│ Return to user: NEW red logo ✓                                 │
└────────────────────────────────────────────────────────────────┘

Timeline View:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ Before Purge:                                                   │
│ ┌────────────────────────────────────┐                        │
│ │ User → Edge → Cache HIT            │                        │
│ │              → Old blue logo       │                        │
│ └────────────────────────────────────┘                        │
│                                                                 │
│ Admin purges cache ─────────────────────────► ✗               │
│                                                                 │
│ After Purge:                                                    │
│ ┌────────────────────────────────────┐                        │
│ │ User → Edge → Cache MISS           │                        │
│ │              → Fetch from origin   │                        │
│ │              → NEW red logo ✓      │                        │
│ └────────────────────────────────────┘                        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Key Points:
✓ Only customer-a's cache is purged
✓ Other customers unaffected
✓ Next request fetches fresh content
✓ Zero downtime (stale content served during purge if needed)
```

---

## Scenario 6: Origin Server Down - Serve Stale

### Context
- User request arrives
- Origin server không available (timeout/error)
- Edge có stale cached content

### Flow Diagram

```
Initial State:
┌────────────────────────────────────────────────────────────────┐
│ Cache Entry:                                                    │
│ - Key: customer-a:/product.json                                │
│ - Age: 4000s (expired, TTL was 3600s)                         │
│ - Status: STALE but still on disk                             │
└────────────────────────────────────────────────────────────────┘

User Request:
┌──────────────┐
│     User     │
└──────┬───────┘
       │ GET https://cdn.customer-a.com/product.json
       ▼
┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Step 1: Check Cache                                             │
│   Key: customer-a:/product.json                                │
│   Found: YES                                                   │
│   Status: STALE (expired 400s ago)                            │
│                                                                 │
│ Step 2: Must Revalidate with Origin                            │
│   Attempting: GET https://origin-a.com/product.json            │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ HTTP GET
                ▼
┌────────────────────────────────────────────────────────────────┐
│              Customer Origin (DOWN!)                            │
│                                                                 │
│         ✗✗✗ CONNECTION TIMEOUT ✗✗✗                            │
│                                                                 │
│ Possible causes:                                                │
│ - Server crashed                                                │
│ - Network issue                                                │
│ - Database down                                                │
│ - DDoS attack                                                  │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Timeout after 10s
                ▼
┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Origin Response: TIMEOUT (10s elapsed)                         │
│                                                                 │
│ Decision Tree:                                                  │
│ ┌──────────────────────────────────────────────────┐          │
│ │ Has stale cache?                                 │          │
│ │   YES → Serve stale content (proxy_cache_use_stale)        │
│ │   NO  → Return 502 Bad Gateway                   │          │
│ └──────────────────────────────────────────────────┘          │
│                                                                 │
│ In this case: Stale cache available ✓                         │
│                                                                 │
│ Action: Serve Stale Content                                    │
│ ┌──────────────────────────────────────────────────┐          │
│ │ Response:                                        │          │
│ │ HTTP/1.1 200 OK                                  │          │
│ │ X-Cache-Status: STALE                            │          │
│ │ Warning: 110 Response is stale                   │          │
│ │                                                   │          │
│ │ [Stale product.json content from cache]          │          │
│ └──────────────────────────────────────────────────┘          │
│                                                                 │
│ Background:                                                     │
│ - Log error about origin timeout                               │
│ - Trigger alert to admin                                       │
│ - Continue serving stale content until origin recovers         │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response with stale content
                ▼
┌──────────────────┐
│      User        │
│                  │
│ ✓ Gets content   │
│ ✓ Slightly old   │
│   but usable     │
│ ✓ Better than    │
│   error page     │
└──────────────────┘

Nginx Configuration:
┌────────────────────────────────────────────────────────────────┐
│ http {                                                          │
│   proxy_cache_use_stale error timeout updating;                │
│   #                     ^^^^^                                   │
│   # Serve stale on origin timeout/error                        │
│                                                                 │
│   proxy_cache_background_update on;                            │
│   # Try to update cache in background                          │
│ }                                                               │
└────────────────────────────────────────────────────────────────┘

Timeline:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ T0: Normal operation                                            │
│     User → Edge → Origin (200ms) → Cache → User               │
│                                                                 │
│ T1: Origin goes down 💥                                        │
│                                                                 │
│ T2: User request arrives                                        │
│     User → Edge → Origin (timeout 10s)                         │
│                   └─► Check stale cache                        │
│                       └─► Serve stale ✓                        │
│                           └─► User (10.05s, but got content)   │
│                                                                 │
│ T3: Origin recovers 🔧                                         │
│                                                                 │
│ T4: Next request                                                │
│     User → Edge → Origin (200ms) → Fresh cache → User         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Comparison:
┌────────────────────────────────────────────────────────────────┐
│ Strategy A: Serve Stale                                         │
│   - User gets slightly outdated content                        │
│   - Response time: 10s (timeout) + 5ms (cache read)           │
│   - User Experience: Acceptable                                 │
│   - Availability: HIGH                                          │
│                                                                 │
│ Strategy B: Return Error                                        │
│   - User sees 502 Bad Gateway                                  │
│   - Response time: 10s (timeout)                               │
│   - User Experience: POOR                                       │
│   - Availability: LOW                                           │
│                                                                 │
│ ✓ Strategy A is better for most use cases                      │
└────────────────────────────────────────────────────────────────┘
```

---

## Scenario 7: High Traffic Spike - Cache Saves Origin

### Context
- Viral content → traffic spike 100x
- Cache hit ratio cao
- Origin không bị overwhelm

### Flow Diagram

```
Normal Traffic Baseline:
┌────────────────────────────────────────────────────────────────┐
│ Time: 10:00 - 11:00                                            │
│ Traffic: 100 req/s                                             │
│ Cache hit ratio: 85%                                           │
│                                                                 │
│ Breakdown:                                                      │
│ - Cache HITs:  85 req/s → Served from edge (fast)             │
│ - Cache MISSes: 15 req/s → Go to origin                       │
│                                                                 │
│ Origin load: 15 req/s (manageable)                            │
└────────────────────────────────────────────────────────────────┘

Spike Event:
┌────────────────────────────────────────────────────────────────┐
│ 11:00: Content goes viral! 🔥                                  │
│ - Featured on homepage                                          │
│ - Shared on social media                                       │
│ - Traffic increases 100x                                       │
└────────────────────────────────────────────────────────────────┘

High Traffic Period:
┌────────────────────────────────────────────────────────────────┐
│ Time: 11:00 - 12:00                                            │
│ Traffic: 10,000 req/s (100x increase!)                        │
│ Cache hit ratio: 95% (popular content cached)                 │
│                                                                 │
│ Breakdown:                                                      │
│ - Cache HITs:  9,500 req/s → Edge servers ✓                   │
│ - Cache MISSes:  500 req/s → Go to origin                     │
│                                                                 │
│ Origin load: 500 req/s                                         │
│   (33x increase, but still manageable!)                       │
└────────────────────────────────────────────────────────────────┘

Visual Comparison:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ WITHOUT CDN (direct to origin):                                │
│                                                                 │
│ Normal:  100 req/s   ████                                      │
│ Spike:  10,000 req/s ████████████████████████████████████████ │
│                      ^ Origin CRASHES 💥                       │
│                                                                 │
│ WITH CDN:                                                       │
│                                                                 │
│ Edge Layer:                                                     │
│ Normal:  100 req/s   ████                                      │
│ Spike:  10,000 req/s ████████████████████████████████████████ │
│                      ^ Edge handles easily ✓                   │
│                                                                 │
│ Origin Load:                                                    │
│ Normal:  15 req/s    █                                         │
│ Spike:   500 req/s   ██████████████ ✓ Manageable              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Request Flow During Spike:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ 10,000 concurrent users requesting /viral-video.mp4            │
│                                                                 │
│         ▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼  ▼                        │
│   ┌─────────────────────────────────────┐                     │
│   │     Load Balancer / GeoDNS          │                     │
│   └──────────────┬──────────────────────┘                     │
│                  │                                              │
│         ┌────────┼────────┐                                    │
│         ▼        ▼        ▼                                    │
│     ┌──────┐ ┌──────┐ ┌──────┐                               │
│     │Edge 1│ │Edge 2│ │Edge 3│                               │
│     │3.3K  │ │3.3K  │ │3.3K  │ req/s each                    │
│     └───┬──┘ └───┬──┘ └───┬──┘                               │
│         │        │        │                                    │
│         │ 95% HITs (from cache)                               │
│         │ 5% MISSes (to origin)                               │
│         │        │        │                                    │
│         └────────┼────────┘                                    │
│                  │ 500 req/s total                            │
│                  ▼                                              │
│           ┌─────────────┐                                      │
│           │   Origin    │                                      │
│           │   Server    │                                      │
│           │  (Healthy)  │                                      │
│           └─────────────┘                                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Metrics During Spike:
┌────────────────────────────────────────────────────────────────┐
│ Metric                  Before      During      Improvement    │
├────────────────────────────────────────────────────────────────┤
│ User requests/s         100         10,000      100x          │
│ Cache hit ratio         85%         95%         +10%          │
│ Origin requests/s       15          500         33x (not 100x!)│
│ Avg response time       50ms        55ms        ~ Same        │
│ Origin CPU usage        10%         35%         Manageable    │
│ Origin crashes          No          No          ✓ Stable      │
│                                                                 │
│ Cost Impact:                                                    │
│ - Origin: Same infrastructure handles 100x traffic            │
│ - Edge: Scales horizontally (more edge nodes if needed)       │
│ - Total savings: ~95% less origin bandwidth                   │
└────────────────────────────────────────────────────────────────┘

Cache Efficiency:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ Why is cache hit ratio higher during spike?                    │
│                                                                 │
│ Normal traffic:                                                 │
│ - Diverse content (many different files)                       │
│ - Lower hit ratio (85%)                                        │
│                                                                 │
│ Spike traffic:                                                  │
│ - Single piece of viral content (same file)                    │
│ - All users request same /viral-video.mp4                      │
│ - First few requests: MISS                                     │
│ - Next 9,990+ requests: HIT (same cached file) ✓              │
│ - Very high hit ratio (95%+)                                   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Key Takeaway:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ CDN's primary value: Origin protection during traffic spikes   │
│                                                                 │
│ Without CDN:                                                    │
│ - 10,000 req/s directly to origin                              │
│ - Origin crashes                                                │
│ - Downtime = lost revenue                                      │
│                                                                 │
│ With CDN:                                                       │
│ - 500 req/s to origin (95% absorbed by cache)                 │
│ - Origin stays healthy                                          │
│ - Users get fast responses                                     │
│ - Business continues operating                                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Scenario 8: Basic Auth to Origin

### Context
- Customer's origin requires authentication
- Edge must provide credentials when fetching

### Flow Diagram

```
Customer Configuration:
┌────────────────────────────────────────────────────────────────┐
│ customers.json:                                                 │
│                                                                 │
│ {                                                               │
│   "secure-customer": {                                         │
│     "origin_url": "https://private.customer.com",              │
│     "custom_domain": "cdn.secure-customer.com",                │
│     "cache_ttl": 3600,                                         │
│     "basic_auth": {                                            │
│       "username": "cdn_user",                                  │
│       "password": "secret_P@ssw0rd"                            │
│     }                                                           │
│   }                                                             │
│ }                                                               │
└────────────────────────────────────────────────────────────────┘

User Request:
┌──────────────┐
│     User     │
│ (Public)     │
└──────┬───────┘
       │ GET https://cdn.secure-customer.com/private-data.json
       │ (No auth needed from user - CDN handles it)
       ▼
┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Step 1: Identify Customer                                       │
│   Host: cdn.secure-customer.com                                │
│   → Customer: secure-customer ✓                               │
│                                                                 │
│ Step 2: Check Cache                                             │
│   Key: secure-customer:/private-data.json                      │
│   Result: MISS (need to fetch)                                 │
│                                                                 │
│ Step 3: Load Customer Config                                    │
│   Has basic_auth: YES                                          │
│   Username: cdn_user                                           │
│   Password: secret_P@ssw0rd                                    │
│                                                                 │
│ Step 4: Encode Credentials                                      │
│   Plain: "cdn_user:secret_P@ssw0rd"                           │
│   Base64: "Y2RuX3VzZXI6c2VjcmV0X1BAc3N3MHJk"                  │
│   Header: "Basic Y2RuX3VzZXI6c2VjcmV0X1BAc3N3MHJk"           │
│                                                                 │
│ Step 5: Request to Origin with Auth                            │
│   GET https://private.customer.com/private-data.json           │
│   Authorization: Basic Y2RuX3VzZXI6c2VjcmV0X1BAc3N3MHJk       │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ HTTPS + Auth Header
                ▼
┌────────────────────────────────────────────────────────────────┐
│           Customer Private Origin Server                        │
│                                                                 │
│ Request received:                                               │
│   GET /private-data.json                                       │
│   Authorization: Basic Y2RuX3VzZXI6c2VjcmV0X1BAc3N3MHJk       │
│                                                                 │
│ Step 1: Validate Authorization                                  │
│   Decode base64: "cdn_user:secret_P@ssw0rd"                   │
│   Check against database:                                      │
│   - Username: cdn_user ✓                                       │
│   - Password: secret_P@ssw0rd ✓                               │
│   - Permissions: read access ✓                                 │
│                                                                 │
│ Step 2: Auth Success - Return Data                             │
│   HTTP/1.1 200 OK                                              │
│   Content-Type: application/json                               │
│   Cache-Control: private, max-age=3600                         │
│                                                                 │
│   {"sensitive": "data", "balance": 1000000}                    │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response
                ▼
┌────────────────────────────────────────────────────────────────┐
│                     Edge Node                                   │
│                                                                 │
│ Step 6: Store in Cache                                          │
│   Key: secure-customer:/private-data.json                      │
│   Content: {"sensitive": "data", ...}                          │
│   TTL: 3600s                                                   │
│                                                                 │
│ Step 7: Return to User                                          │
│   HTTP/1.1 200 OK                                              │
│   X-Cache-Status: MISS                                         │
│   (DO NOT include Authorization header to user!)              │
│                                                                 │
│   {"sensitive": "data", "balance": 1000000}                    │
│                                                                 │
└───────────────┬────────────────────────────────────────────────┘
                │
                │ Response (no auth headers exposed)
                ▼
┌──────────────────┐
│      User        │
│                  │
│ ✓ Gets data      │
│ ✓ No auth needed │
│   from user side │
└──────────────────┘

Security Flow:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │                    Public Internet                        │  │
│ │  User ──(no auth)──► Edge CDN                            │  │
│ └──────────────────┬───────────────────────────────────────┘  │
│                    │                                            │
│                    │ CDN adds auth                             │
│                    ▼                                            │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │             Private Origin Network                        │  │
│ │  Edge ──(with auth)──► Origin Server                     │  │
│ │  (HTTPS + Basic Auth header)                             │  │
│ └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

What if Auth Fails?
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│ Scenario: Wrong password in config                             │
│                                                                 │
│ Edge → Origin (wrong auth)                                     │
│                                                                 │
│ Origin Response:                                                │
│   HTTP/1.1 401 Unauthorized                                    │
│   WWW-Authenticate: Basic realm="Private"                      │
│                                                                 │
│ Edge Action:                                                    │
│ 1. Log error: "Auth failed for secure-customer"               │
│ 2. Alert admin: "Check basic_auth config"                     │
│ 3. Return 502 to user: "Origin authentication failed"         │
│ 4. DO NOT cache 401 response                                   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Lua Code Snippet:
┌────────────────────────────────────────────────────────────────┐
│ -- src/customer_manager.lua                                    │
│                                                                 │
│ function get_auth_header(customer_config)                      │
│   if not customer_config.basic_auth then                       │
│     return nil  -- No auth needed                             │
│   end                                                           │
│                                                                 │
│   local username = customer_config.basic_auth.username         │
│   local password = customer_config.basic_auth.password         │
│                                                                 │
│   local credentials = username .. ":" .. password              │
│   local encoded = ngx.encode_base64(credentials)               │
│                                                                 │
│   return "Basic " .. encoded                                   │
│ end                                                             │
│                                                                 │
│ -- src/edge.lua                                                │
│                                                                 │
│ local auth_header = customer_manager.get_auth_header(customer) │
│ if auth_header then                                            │
│   ngx.var.auth_header = auth_header                           │
│ end                                                             │
└────────────────────────────────────────────────────────────────┘
```

---

## Summary Table

| Scenario | Key Concept | User Impact | Origin Impact |
|----------|-------------|-------------|---------------|
| 1. First Request | Cache MISS | Slow (origin latency) | 1 request |
| 2. Cached Request | Cache HIT | Fast (5ms) | 0 requests |
| 3. Same URI, Different Customers | Cache isolation | No conflicts | Separate origins |
| 4. Cache Expiration | TTL & revalidation | Transparent refresh | Periodic requests |
| 5. Cache Purge | Manual invalidation | Gets fresh content | 1 request after purge |
| 6. Origin Down | Serve stale | Gets old but valid content | 0 requests (down) |
| 7. Traffic Spike | Cache efficiency | Fast responses | Minimal increase |
| 8. Basic Auth | Transparent auth | No auth needed | Secure access |

---

## Key Insights

### Cache Hit Ratio Impact:
```
┌──────────────────────────────────────────────────────────────┐
│ Cache Hit Ratio | User Experience | Origin Load              │
├──────────────────────────────────────────────────────────────┤
│ 50%            | Mixed (slow)    | 50% of traffic          │
│ 80%            | Good            | 20% of traffic          │
│ 90%            | Great           | 10% of traffic          │
│ 95%            | Excellent       | 5% of traffic           │
│ 99%            | Outstanding     | 1% of traffic           │
└──────────────────────────────────────────────────────────────┘
```

### Response Time Distribution:
```
┌──────────────────────────────────────────────────────────────┐
│ Request Type  | Avg Response Time | % of Requests           │
├──────────────────────────────────────────────────────────────┤
│ Cache HIT     | 5-20ms           | 85-95%                  │
│ Cache MISS    | 100-500ms        | 5-15%                   │
│ Cache STALE   | 50-100ms         | <1% (only if origin down)│
└──────────────────────────────────────────────────────────────┘
```

---

**Document Version**: 1.0
**Last Updated**: 2025-10-01
**Covers**: 8 fundamental scenarios for multi-tenant CDN operations
