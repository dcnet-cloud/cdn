# Multi-Tenant CDN Architecture & Workflow Diagrams

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         END USERS                                    │
│  User A → cdn.customer-a.com/image.jpg                              │
│  User B → cdn.customer-b.com/video.mp4                              │
│  User C → cdn.customer-c.com/data.json                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ DNS Resolution
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    EDGE LAYER (Multi-Tenant)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │  Edge Node  │  │  Edge Node  │  │  Edge Node  │                │
│  │      1      │  │      2      │  │      3      │                │
│  │             │  │             │  │             │                │
│  │  - Nginx    │  │  - Nginx    │  │  - Nginx    │                │
│  │  - Lua      │  │  - Lua      │  │  - Lua      │                │
│  │  - Cache    │  │  - Cache    │  │  - Cache    │                │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                │
└─────────┼─────────────────┼─────────────────┼────────────────────────┘
          │                 │                 │
          │ Shared Config   │                 │
          └────────┬────────┴─────────────────┘
                   │
          ┌────────▼──────────┐
          │  customers.json   │
          │  (Config Store)   │
          └───────────────────┘
                   │
          ┌────────┴────────────────────────────────────┐
          │                                             │
          │  Customer Configs:                          │
          │  - customer-a → origin-a.com                │
          │  - customer-b → origin-b.com + BasicAuth    │
          │  - customer-c → origin-c.com                │
          └─────────────────────────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ▼              ▼              ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Customer │  │Customer │  │Customer │
│   A     │  │   B     │  │   C     │
│ Origin  │  │ Origin  │  │ Origin  │
│ Server  │  │ Server  │  │ Server  │
└─────────┘  └─────────┘  └─────────┘
(External)   (External)   (External)
```

---

## 2. Request Flow - Step by Step

### **Step 1: User Request**
```
┌──────────┐
│   User   │
│  Browser │
└────┬─────┘
     │
     │ GET https://cdn.customer-a.com/image.jpg
     │ Host: cdn.customer-a.com
     ▼
```

### **Step 2: DNS Resolution**
```
┌──────────────┐
│     DNS      │
│   (GeoDNS)   │
└──────┬───────┘
       │
       │ Returns: Edge Node IP (nearest)
       │ Example: 203.0.113.10
       ▼
```

### **Step 3: Request Arrives at Edge**
```
┌─────────────────────────────────────────────────────┐
│              Edge Node (Nginx)                       │
│                                                      │
│  HTTP Request:                                       │
│  GET /image.jpg HTTP/1.1                            │
│  Host: cdn.customer-a.com                           │
│  User-Agent: Mozilla/5.0                            │
│                                                      │
└──────────────────┬──────────────────────────────────┘
                   │
                   │ Pass to Nginx
                   ▼
```

### **Step 4: Nginx Processing**
```
┌─────────────────────────────────────────────────────┐
│              nginx_edge.conf                         │
│                                                      │
│  server {                                            │
│    listen 8080;                                      │
│                                                      │
│    location / {                                      │
│      set $origin_url '';                            │
│      set $cache_key '';                             │
│      set $auth_header '';                           │
│                                                      │
│      # Call Lua handler                             │
│      access_by_lua_block {                          │
│        local edge = require "edge"                  │
│        edge.handle_request()  ◄─────────────┐      │
│      }                                       │      │
│    }                                         │      │
│  }                                           │      │
└──────────────────────────────────────────────┼──────┘
                                               │
                                               │
                                               ▼
```

### **Step 5: Lua Request Handler**
```
┌──────────────────────────────────────────────────────────────┐
│                    src/edge.lua                               │
│                  handle_request()                             │
│                                                               │
│  1. Extract Host header                                       │
│     host = ngx.var.http_host                                 │
│     → "cdn.customer-a.com"                                   │
│                                                               │
│  2. Lookup customer config                                    │
│     customer = customer_manager.get_customer_by_domain(host) │
│     ┌────────────────────────────────┐                       │
│     │  Call customer_manager.lua     │                       │
│     └────────┬───────────────────────┘                       │
│              │                                                │
│              ▼                                                │
└──────────────┼───────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────┐
│              src/customer_manager.lua                         │
│            get_customer_by_domain()                           │
│                                                               │
│  1. Check shared dict (in-memory cache)                       │
│     local config = customers_dict:get(domain)                │
│                                                               │
│  2. If found:                                                 │
│     ┌──────────────────────────────────────┐                │
│     │ {                                     │                │
│     │   "customer_id": "customer-a",        │                │
│     │   "origin_url": "https://origin-a.com"│                │
│     │   "cache_ttl": 3600,                  │                │
│     │   "basic_auth": {                     │                │
│     │     "username": "user_a",             │                │
│     │     "password": "pass_a"              │                │
│     │   }                                   │                │
│     │ }                                     │                │
│     └──────────────────────────────────────┘                │
│                                                               │
│  3. Return config to edge.lua                                │
└──────────────┬────────────────────────────────────────────────┘
               │
               │ Return customer config
               ▼
┌──────────────────────────────────────────────────────────────┐
│                    src/edge.lua                               │
│                  handle_request() (continued)                 │
│                                                               │
│  3. Validate customer found                                   │
│     if not customer then                                      │
│       return 404 "Customer not found"                        │
│     end                                                       │
│                                                               │
│  4. Generate cache key with isolation                         │
│     cache_key = "customer-a:/image.jpg"                      │
│     ngx.var.cache_key = cache_key                            │
│                                                               │
│  5. Set origin URL                                            │
│     ngx.var.origin_url = "https://origin-a.com/image.jpg"   │
│                                                               │
│  6. Set Basic Auth header (if needed)                         │
│     if customer.basic_auth then                              │
│       auth = base64(user_a:pass_a)                           │
│       ngx.var.auth_header = "Basic " .. auth                 │
│     end                                                       │
│                                                               │
│  7. Return to Nginx                                           │
└──────────────┬────────────────────────────────────────────────┘
               │
               │ Variables set:
               │ - $origin_url = https://origin-a.com/image.jpg
               │ - $cache_key = customer-a:/image.jpg
               │ - $auth_header = Basic dXNlcl9hOnBhc3NfYQ==
               ▼
```

### **Step 6: Cache Check**
```
┌──────────────────────────────────────────────────────────────┐
│                    Nginx Cache Layer                          │
│                                                               │
│  proxy_cache zone_1;                                          │
│  proxy_cache_key $cache_key;  ◄── "customer-a:/image.jpg"   │
│                                                               │
│  Check cache:                                                 │
│  /cache/XX/YY/customer-a:/image.jpg                          │
│                                                               │
└──────────────┬───────────────────────────┬────────────────────┘
               │                           │
        ┌──────▼──────┐           ┌───────▼────────┐
        │ CACHE HIT   │           │  CACHE MISS    │
        └──────┬──────┘           └────────┬───────┘
               │                           │
               │                           ▼
               │                  ┌─────────────────┐
               │                  │ Go to Step 7    │
               │                  │ (Fetch Origin)  │
               │                  └─────────────────┘
               │
               ▼
        ┌──────────────────┐
        │ Return cached    │
        │ content to user  │
        │                  │
        │ Headers:         │
        │ X-Cache-Status:  │
        │   HIT            │
        └──────┬───────────┘
               │
               ▼
        ┌──────────────┐
        │  END (Fast)  │
        └──────────────┘
```

### **Step 7: Origin Fetch (Cache Miss)**
```
┌──────────────────────────────────────────────────────────────┐
│                 Nginx Proxy to Origin                         │
│                                                               │
│  proxy_pass $origin_url;                                      │
│  → https://origin-a.com/image.jpg                            │
│                                                               │
│  proxy_set_header Authorization $auth_header;                │
│  → Authorization: Basic dXNlcl9hOnBhc3NfYQ==                │
│                                                               │
│  proxy_set_header Host cdn.customer-a.com;                   │
│  proxy_set_header X-Real-IP 1.2.3.4;                         │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ HTTP Request
                   ▼
┌──────────────────────────────────────────────────────────────┐
│              Customer A Origin Server                         │
│              (origin-a.com)                                   │
│                                                               │
│  1. Receive request with Basic Auth                           │
│  2. Validate credentials                                      │
│  3. Return image.jpg                                          │
│                                                               │
│  Response:                                                    │
│  HTTP/1.1 200 OK                                             │
│  Content-Type: image/jpeg                                    │
│  Content-Length: 524288                                      │
│  Cache-Control: public, max-age=3600                         │
│                                                               │
│  [Image binary data...]                                      │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ Response
                   ▼
```

### **Step 8: Cache Storage**
```
┌──────────────────────────────────────────────────────────────┐
│                    Nginx Cache Layer                          │
│                                                               │
│  1. Store response in cache:                                  │
│     Key: customer-a:/image.jpg                               │
│     Location: /cache/XX/YY/hash(customer-a:/image.jpg)      │
│                                                               │
│  2. Cache metadata:                                           │
│     - TTL: 3600 seconds (from Cache-Control)                 │
│     - Size: 524288 bytes                                     │
│     - Created: 2025-10-01 10:30:00                           │
│                                                               │
│  3. Add response headers:                                     │
│     X-Cache-Status: MISS                                     │
│     X-Customer-ID: customer-a:/image.jpg                     │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ Response with headers
                   ▼
```

### **Step 9: Response to User**
```
┌──────────────────────────────────────────────────────────────┐
│                    Edge Node Response                         │
│                                                               │
│  HTTP/1.1 200 OK                                             │
│  Content-Type: image/jpeg                                    │
│  Content-Length: 524288                                      │
│  Cache-Control: public, max-age=3600                         │
│  X-Cache-Status: MISS                                        │
│  X-Edge: MultiTenant                                         │
│  X-Customer-ID: customer-a:/image.jpg                        │
│                                                               │
│  [Image binary data...]                                      │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ HTTP Response
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    User Browser                               │
│                                                               │
│  ✓ Received image.jpg                                        │
│  ✓ Display to user                                           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Cache Isolation Example

### Scenario: Same URI, Different Customers

```
┌─────────────────────────────────────────────────────────────┐
│                      Timeline                                │
└─────────────────────────────────────────────────────────────┘

Time T1:
┌──────────────────────────────────────────────────────────────┐
│ Request 1: User A                                             │
│ GET https://cdn.customer-a.com/logo.png                      │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Cache Key Generated │
         │ "customer-a:/logo.png" │
         └─────────┬───────────┘
                   │
                   ▼ MISS
         ┌─────────────────────┐
         │ Fetch from          │
         │ origin-a.com        │
         └─────────┬───────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Store in cache:     │
         │ /cache/a1/b2/...    │
         │ Key: customer-a:    │
         │      /logo.png      │
         └─────────────────────┘

Time T2 (5 seconds later):
┌──────────────────────────────────────────────────────────────┐
│ Request 2: User B                                             │
│ GET https://cdn.customer-b.com/logo.png                      │
│ (Same URI: /logo.png but different customer!)                │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Cache Key Generated │
         │ "customer-b:/logo.png" │  ◄── DIFFERENT KEY!
         └─────────┬───────────┘
                   │
                   ▼ MISS (different key)
         ┌─────────────────────┐
         │ Fetch from          │
         │ origin-b.com        │  ◄── Different origin!
         └─────────┬───────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Store in cache:     │
         │ /cache/c3/d4/...    │
         │ Key: customer-b:    │
         │      /logo.png      │
         └─────────────────────┘

Result:
┌─────────────────────────────────────────────────────────────┐
│                    Cache Contents                            │
│                                                              │
│  Entry 1:                                                    │
│  Key: customer-a:/logo.png                                  │
│  Content: [Customer A's logo.png from origin-a.com]         │
│                                                              │
│  Entry 2:                                                    │
│  Key: customer-b:/logo.png                                  │
│  Content: [Customer B's logo.png from origin-b.com]         │
│                                                              │
│  ✓ Both logos cached separately                             │
│  ✓ No collision or conflict                                 │
│  ✓ Perfect isolation                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Customer Onboarding Workflow

```
┌──────────────────────────────────────────────────────────────┐
│ Step 1: Admin Creates Customer via API                       │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │   Admin Terminal    │
         │                     │
         │ $ curl -X POST      │
         │   /api/customers    │
         │   -d '{             │
         │     "customer_id":  │
         │       "acme-corp",  │
         │     "origin_url":   │
         │       "https://     │
         │        cdn.acme.com"│
         │   }'                │
         └─────────┬───────────┘
                   │
                   │ HTTP POST
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    Admin API (Flask)                          │
│                  POST /api/customers                          │
│                                                               │
│  1. Validate input                                            │
│     ✓ customer_id unique                                     │
│     ✓ origin_url valid format                                │
│     ✓ custom_domain not taken                                │
│                                                               │
│  2. Load current customers.json                               │
│     customers = json.load(file)                              │
│                                                               │
│  3. Add new customer                                          │
│     customers["acme-corp"] = {                               │
│       "origin_url": "https://cdn.acme.com",                  │
│       "custom_domain": "cdn.acme-corp.net",                  │
│       "cache_ttl": 3600,                                     │
│       "basic_auth": null                                     │
│     }                                                         │
│                                                               │
│  4. Save to customers.json                                    │
│     json.dump(customers, file)                               │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│              config/customers.json (Updated)                  │
│                                                               │
│  {                                                            │
│    "customer-a": { ... },                                    │
│    "customer-b": { ... },                                    │
│    "acme-corp": {                                            │
│      "origin_url": "https://cdn.acme.com",                   │
│      "custom_domain": "cdn.acme-corp.net",                   │
│      "cache_ttl": 3600,                                      │
│      "basic_auth": null                                      │
│    }                                                          │
│  }                                                            │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ File updated
                   ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 2: Reload Edge Nodes Configuration                      │
│                                                               │
│ Option A: Nginx reload (graceful)                            │
│   $ nginx -s reload                                          │
│                                                               │
│ Option B: Shared dict hot reload (no restart)                │
│   - Watch file changes                                       │
│   - Update shared dict                                       │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Edge Nodes          │
         │                     │
         │ ✓ Config reloaded   │
         │ ✓ New customer      │
         │   recognized        │
         └─────────┬───────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 3: Customer Configures DNS                              │
│                                                               │
│ CNAME Record:                                                 │
│ cdn.acme-corp.net → edge-node.yourcdn.com                    │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Customer Ready!     │
         │                     │
         │ Users can now       │
         │ access:             │
         │ cdn.acme-corp.net   │
         └─────────────────────┘
```

---

## 5. Cache Purge Workflow

```
┌──────────────────────────────────────────────────────────────┐
│ Scenario: Customer A wants to purge cache                    │
│ (e.g., updated logo.png at origin)                           │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │   Admin Terminal    │
         │                     │
         │ $ curl -X DELETE    │
         │   /api/customers/   │
         │   customer-a/cache  │
         └─────────┬───────────┘
                   │
                   │ HTTP DELETE
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    Admin API (Flask)                          │
│          DELETE /api/customers/:id/cache                      │
│                                                               │
│  1. Validate customer exists                                  │
│     customer = customers.get("customer-a")                   │
│     if not customer: return 404                              │
│                                                               │
│  2. Find cache files for this customer                        │
│     pattern = "/cache/**/customer-a:*"                       │
│                                                               │
│  3. Scan cache directory                                      │
│     ┌──────────────────────────────┐                         │
│     │ /cache/                      │                         │
│     │   a1/                        │                         │
│     │     b2/                      │                         │
│     │       hash1 ◄── customer-a:/logo.png                  │
│     │       hash2 ◄── customer-a:/image.jpg                 │
│     │   c3/                        │                         │
│     │     d4/                      │                         │
│     │       hash3 ◄── customer-b:/logo.png (skip)           │
│     └──────────────────────────────┘                         │
│                                                               │
│  4. Delete matching files                                     │
│     os.remove("/cache/a1/b2/hash1")                          │
│     os.remove("/cache/a1/b2/hash2")                          │
│     purged_count = 2                                         │
│                                                               │
│  5. Return result                                             │
│     {"message": "Purged 2 files"}                            │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ Response
                   ▼
         ┌─────────────────────┐
         │   Admin Terminal    │
         │                     │
         │ Response:           │
         │ {                   │
         │   "message":        │
         │     "Cache purged"  │
         │   "purged_files": 2 │
         │ }                   │
         └─────────┬───────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│               Next Request Behavior                           │
│                                                               │
│ User requests: cdn.customer-a.com/logo.png                   │
│                                                               │
│ Cache check: customer-a:/logo.png                            │
│ Result: MISS (file was deleted)                              │
│                                                               │
│ Action: Fetch fresh from origin-a.com                        │
│ Cache: Store new version                                     │
│                                                               │
│ ✓ User gets updated logo.png                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Monitoring & Metrics Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    Request Processing                         │
│                                                               │
│  Every request through edge nodes generates metrics:          │
│                                                               │
│  - customer_id                                                │
│  - request URI                                                │
│  - cache status (HIT/MISS)                                   │
│  - response time                                              │
│  - bytes transferred                                          │
│  - origin response time                                       │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ Metrics exposed via VTS module
                   ▼
┌──────────────────────────────────────────────────────────────┐
│              Edge Node Metrics Endpoint                       │
│          http://edge:8080/status/format/prometheus           │
│                                                               │
│  nginx_vts_server_requests_total{                            │
│    host="cdn.customer-a.com",                                │
│    method="GET"                                              │
│  } 1523                                                       │
│                                                               │
│  nginx_vts_server_cache_total{                               │
│    host="cdn.customer-a.com",                                │
│    status="hit"                                              │
│  } 1220                                                       │
│                                                               │
│  nginx_vts_server_cache_total{                               │
│    host="cdn.customer-a.com",                                │
│    status="miss"                                             │
│  } 303                                                        │
│                                                               │
│  nginx_vts_server_bytes_total{                               │
│    host="cdn.customer-a.com",                                │
│    direction="out"                                           │
│  } 15728640                                                   │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ Scrape interval: 15s
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                      Prometheus                               │
│            http://prometheus:9090                             │
│                                                               │
│  1. Scrape metrics from all edge nodes                        │
│     - edge:8080/status/format/prometheus                     │
│     - edge1:8080/status/format/prometheus                    │
│     - edge2:8080/status/format/prometheus                    │
│                                                               │
│  2. Store time-series data                                    │
│     - Retention: 24 hours (configurable)                     │
│                                                               │
│  3. Aggregate metrics                                         │
│     - Sum across all edge nodes                              │
│     - Calculate rates, percentiles                           │
│                                                               │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ Prometheus queries
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                      Grafana                                  │
│              http://grafana:3000                              │
│                                                               │
│  Dashboard: Multi-Tenant CDN Monitoring                       │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Panel 1: Request Rate per Customer                       ││
│  │                                                           ││
│  │ Query:                                                    ││
│  │   rate(nginx_vts_server_requests_total[5m])             ││
│  │                                                           ││
│  │ Chart:                                                    ││
│  │   customer-a: ████████████ 150 req/s                    ││
│  │   customer-b: ██████ 80 req/s                            ││
│  │   customer-c: ████ 45 req/s                              ││
│  └───────────────────────────────────────────────────────────┘│
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Panel 2: Cache Hit Ratio per Customer                    ││
│  │                                                           ││
│  │ Query:                                                    ││
│  │   sum(rate(cache_total{status="hit"}[5m])) /            ││
│  │   sum(rate(cache_total[5m]))                            ││
│  │                                                           ││
│  │ Gauges:                                                   ││
│  │   customer-a: 85% ████████▌                              ││
│  │   customer-b: 92% █████████▎                             ││
│  │   customer-c: 78% ███████▊                               ││
│  └───────────────────────────────────────────────────────────┘│
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Panel 3: Bandwidth Usage (24h)                           ││
│  │                                                           ││
│  │ Time series graph showing GB transferred per customer    ││
│  │                                                           ││
│  │    GB                                                     ││
│  │    │                                                      ││
│  │ 50─┤     ╭─╮          ╭─╮                               ││
│  │    │    ╭╯ ╰╮        ╭╯ ╰╮       customer-a             ││
│  │ 30─┤   ╭╯   ╰─╮    ╭─╯   ╰╮                             ││
│  │    │  ╭╯      ╰────╯      ╰╮                            ││
│  │ 10─┤──╯                    ╰────  customer-b, c         ││
│  │    └─────────────────────────────► Time                 ││
│  └───────────────────────────────────────────────────────────┘│
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 7. Error Handling Flows

### 7.1 Customer Not Found
```
User Request → Edge Node
                  │
                  ▼
         Lua: handle_request()
                  │
                  ▼
         Lookup customer by domain
                  │
                  ▼
         Customer = nil (not found)
                  │
                  ▼
         ┌─────────────────────┐
         │ Return 404          │
         │                     │
         │ {                   │
         │   "error":          │
         │     "Customer not   │
         │      found",        │
         │   "domain":         │
         │     "unknown.com"   │
         │ }                   │
         └─────────┬───────────┘
                   │
                   ▼
         User sees error message
```

### 7.2 Origin Server Down
```
User Request → Edge Node
                  │
                  ▼
         Cache check: MISS
                  │
                  ▼
         Proxy to origin
                  │
                  ▼
         Origin timeout / connection refused
                  │
                  ▼
         ┌─────────────────────┐
         │ Check stale cache   │
         │ (if available)      │
         └─────────┬───────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
    Stale cache          No stale
     available            cache
         │                   │
         ▼                   ▼
┌─────────────────┐  ┌─────────────────┐
│ Serve stale     │  │ Return 502      │
│ content         │  │ Bad Gateway     │
│                 │  │                 │
│ Headers:        │  │ Log error       │
│ X-Cache-Status: │  │ Alert admin     │
│   STALE         │  └─────────────────┘
└─────────────────┘
```

### 7.3 Basic Auth Failure
```
Edge Node → Origin (with Basic Auth)
                  │
                  ▼
         Origin validates auth
                  │
                  ▼
         Returns 401 Unauthorized
                  │
                  ▼
         ┌─────────────────────┐
         │ Edge receives 401   │
         │                     │
         │ Options:            │
         │ 1. Pass to user     │
         │ 2. Log error        │
         │ 3. Alert admin      │
         │    (config issue)   │
         └─────────┬───────────┘
                   │
                   ▼
         User sees 401 error
         Admin gets alert about
         misconfigured auth
```

---

## 8. Complete System Diagram

```
                                    ┌─────────────────┐
                                    │  Admin Portal   │
                                    │  (Web UI)       │
                                    └────────┬────────┘
                                             │
                                             │ Manage customers
                                             ▼
                                    ┌─────────────────┐
                                    │   Admin API     │
                                    │   (Flask)       │
                                    │   Port: 9000    │
                                    └────────┬────────┘
                                             │
                                             │ CRUD operations
                                             ▼
                                    ┌─────────────────┐
                                    │ customers.json  │
                                    │ (Config Store)  │
                                    └────────┬────────┘
                                             │
                       ┌─────────────────────┼─────────────────────┐
                       │                     │                     │
                       ▼                     ▼                     ▼
              ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
              │   Edge Node 1   │  │   Edge Node 2   │  │   Edge Node 3   │
              │                 │  │                 │  │                 │
              │ - Nginx         │  │ - Nginx         │  │ - Nginx         │
              │ - Lua           │  │ - Lua           │  │ - Lua           │
              │ - Cache         │  │ - Cache         │  │ - Cache         │
              │ - VTS Metrics   │  │ - VTS Metrics   │  │ - VTS Metrics   │
              │                 │  │                 │  │                 │
              │ Port: 8081      │  │ Port: 8082      │  │ Port: 8083      │
              └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
                       │                     │                     │
                       └─────────────────────┼─────────────────────┘
                                             │
                              ┌──────────────┼──────────────┐
                              │              │              │
                              ▼              ▼              ▼
                     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                     │  Customer A  │ │  Customer B  │ │  Customer C  │
                     │   Origin     │ │   Origin     │ │   Origin     │
                     │  (External)  │ │  (External)  │ │  (External)  │
                     └──────────────┘ └──────────────┘ └──────────────┘
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       │                                           │
                       ▼                                           ▼
              ┌─────────────────┐                         ┌─────────────────┐
              │   Prometheus    │◄────────────────────────│    Grafana      │
              │                 │  Queries                │                 │
              │ Port: 9090      │                         │ Port: 9091      │
              │                 │                         │                 │
              │ - Scrapes       │                         │ - Dashboards    │
              │   metrics from  │                         │ - Alerts        │
              │   edge nodes    │                         │ - Visualization │
              └─────────────────┘                         └─────────────────┘
                       │
                       │ Scrape metrics
                       │ every 15s
                       │
                       └────────► /status/format/prometheus
```

---

## Summary

### Key Workflows Covered:

1. ✅ **High-Level Architecture** - Overall system components
2. ✅ **Request Flow** - Step-by-step request processing (9 steps)
3. ✅ **Cache Isolation** - How multi-tenant caching works
4. ✅ **Customer Onboarding** - Adding new customers
5. ✅ **Cache Purge** - Selective cache invalidation
6. ✅ **Monitoring** - Metrics collection and visualization
7. ✅ **Error Handling** - Various error scenarios
8. ✅ **Complete System** - All components together

### Key Concepts:

- **Customer Identification**: Via Host header
- **Cache Isolation**: customer_id prefix in cache keys
- **Dynamic Routing**: Lua sets upstream URL per request
- **Basic Auth**: Transparent proxy authentication
- **Monitoring**: Per-customer metrics via VTS module
- **Management**: REST API for CRUD operations

Các diagram này giúp hiểu rõ toàn bộ flow từ request đến response, và cách các components tương tác với nhau!
