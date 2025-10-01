# Kế hoạch POC Multi-Tenant CDN

## Mục tiêu
Chuyển đổi CDN hiện tại thành SaaS phục vụ nhiều customers, mỗi customer có origin URL riêng với khả năng cache và phục vụ nội dung qua custom domain.

## Flow hoạt động
```
1. Customer đăng ký → Cung cấp origin URL (vd: https://assets.customer-a.com)
2. Admin add vào hệ thống → Customer nhận được CDN URL (cdn.customer-a.com)
3. End user request → https://cdn.customer-a.com/image.jpg
4. Edge node:
   - Identify customer từ domain
   - Check cache với key: customer-a:/image.jpg
   - Nếu miss → Fetch từ https://assets.customer-a.com/image.jpg (với Basic Auth)
   - Cache lại và return
5. Request tiếp theo → Serve từ cache (fast)
```

## Kiến trúc Multi-Tenant

```
┌─────────────────────────────────────────────────────────────┐
│                    End Users                                 │
│  (cdn.customer-a.com)  (cdn.customer-b.com)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Edge Nodes (Multi-Tenant)                       │
│  - Customer identification (Host header)                     │
│  - Cache isolation (customer_id:uri)                         │
│  - Dynamic upstream routing                                  │
│  - Shared infrastructure                                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Customer A   │ │ Customer B   │ │ Customer C   │
│ Origin       │ │ Origin       │ │ Origin       │
│ (External)   │ │ (External)   │ │ (External)   │
│ + Basic Auth │ │ + Basic Auth │ │ + Basic Auth │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Key Features:
- **Customer Isolation**: Cache keys bao gồm customer_id để tránh conflict
- **Dynamic Routing**: Mỗi customer có origin URL riêng
- **Custom Domain**: Support subdomain hoặc custom domain
- **Basic Authentication**: Hỗ trợ Basic Auth khi fetch từ origin
- **Shared Infrastructure**: Tất cả customers dùng chung edge nodes

---

## Implementation Plan

### **Phase 1: Customer Configuration System (Tuần 1)**

#### 1.1 Customer Configuration File
**Tạo file: `config/customers.json`**

```json
{
  "customer-a": {
    "origin_url": "https://origin-a.example.com",
    "custom_domain": "cdn.customer-a.com",
    "cache_ttl": 3600,
    "basic_auth": {
      "username": "user_a",
      "password": "pass_a"
    }
  },
  "customer-b": {
    "origin_url": "https://origin-b.example.com",
    "custom_domain": "cdn.customer-b.com",
    "cache_ttl": 7200,
    "basic_auth": {
      "username": "user_b",
      "password": "pass_b"
    }
  },
  "customer-c": {
    "origin_url": "https://origin-c.example.com",
    "custom_domain": "cdn.customer-c.com",
    "cache_ttl": 1800,
    "basic_auth": null
  }
}
```

**Giải thích:**
- `customer_id`: Unique identifier (key của object)
- `origin_url`: Origin server URL của customer
- `custom_domain`: Custom domain mà customer sẽ sử dụng
- `cache_ttl`: Cache time-to-live (seconds)
- `basic_auth`: Credentials để fetch từ origin (optional)

#### 1.2 Customer Manager Module
**Tạo file: `src/customer_manager.lua`**

**Chức năng:**
- Load customers.json vào shared memory (init phase)
- Lookup customer config by domain
- Validate customer exists
- Generate cache key với customer isolation
- Encode Basic Auth credentials

**Pseudo code:**
```lua
local cjson = require "cjson"
local customer_manager = {}

-- Shared dict để cache customer configs
local customers_dict = ngx.shared.customers

function customer_manager.load_customers()
  -- Đọc file customers.json
  local file = io.open("/config/customers.json", "r")
  local content = file:read("*a")
  file:close()

  -- Parse JSON
  local customers = cjson.decode(content)

  -- Store vào shared dict (key = domain, value = config)
  for customer_id, config in pairs(customers) do
    config.customer_id = customer_id
    customers_dict:set(config.custom_domain, cjson.encode(config))
  end
end

function customer_manager.get_customer_by_domain(domain)
  local config_json = customers_dict:get(domain)
  if not config_json then
    return nil, "Customer not found"
  end
  return cjson.decode(config_json)
end

function customer_manager.get_cache_key(customer_id, uri)
  return customer_id .. ":" .. uri
end

function customer_manager.get_auth_header(customer_config)
  if not customer_config.basic_auth then
    return nil
  end

  local auth_string = customer_config.basic_auth.username .. ":" ..
                      customer_config.basic_auth.password
  local encoded = ngx.encode_base64(auth_string)
  return "Basic " .. encoded
end

return customer_manager
```

---

### **Phase 2: Multi-Tenant Edge Layer (Tuần 1-2)**

#### 2.1 Enhanced Edge Logic
**Sửa file: `src/edge.lua`**

**Thay đổi từ:**
```lua
local edge = {}

edge.simulate_load = function()
  simulations.for_work_longtail(simulations.profiles.edge)
end

return edge
```

**Thành:**
```lua
local simulations = require "simulations"
local customer_manager = require "customer_manager"

local edge = {}

edge.simulate_load = function()
  simulations.for_work_longtail(simulations.profiles.edge)
end

edge.handle_request = function()
  -- 1. Identify customer từ Host header
  local host = ngx.var.http_host

  -- 2. Lookup customer config
  local customer, err = customer_manager.get_customer_by_domain(host)
  if not customer then
    ngx.status = 404
    ngx.header["Content-Type"] = "application/json"
    ngx.say('{"error": "Customer not found", "domain": "' .. host .. '"}')
    return ngx.exit(404)
  end

  -- 3. Set origin URL cho proxy_pass
  ngx.var.origin_url = customer.origin_url .. ngx.var.request_uri

  -- 4. Generate cache key với customer isolation
  ngx.var.cache_key = customer_manager.get_cache_key(
    customer.customer_id,
    ngx.var.uri
  )

  -- 5. Set Basic Auth header nếu cần
  local auth_header = customer_manager.get_auth_header(customer)
  if auth_header then
    ngx.var.auth_header = auth_header
  end

  -- 6. Simulate load (optional, for testing)
  edge.simulate_load()
end

return edge
```

#### 2.2 Multi-Tenant Edge Nginx Config
**Sửa file: `nginx_edge.conf`**

**Thay đổi từ:**
```nginx
http {
  resolver 127.0.0.11 ipv6=off;
  include generic_conf/setup_logging.conf;
  include generic_conf/lua_path_setup.conf;
  include generic_conf/basic_vts_setup.conf;
  include generic_conf/setup_cache.conf;

  upstream backend {
    server backend:8080;
    server backend1:8080;
    keepalive 10;
  }

  server {
    listen 8080;

    location / {
      set_by_lua_block $cache_key {
        return ngx.var.uri
      }

      access_by_lua_block {
        local edge = require "edge"
        edge.simulate_load()
      }

      proxy_pass http://backend;
      include generic_conf/define_cache.conf;
      add_header X-Edge Server;
    }

    include generic_conf/basic_vts_location.conf;
  }
}
```

**Thành:**
```nginx
http {
  resolver 8.8.8.8 ipv6=off;  # External resolver cho customer origins
  include generic_conf/setup_logging.conf;
  include generic_conf/lua_path_setup.conf;
  include generic_conf/basic_vts_setup.conf;
  include generic_conf/setup_cache.conf;

  # Shared dict để cache customer configs
  lua_shared_dict customers 10m;

  # Load customer configs at startup
  init_by_lua_block {
    customer_manager = require "customer_manager"
    customer_manager.load_customers()
  }

  server {
    listen 8080;

    location / {
      # Variables để set từ Lua
      set $origin_url '';
      set $cache_key '';
      set $auth_header '';

      # Multi-tenant request handling
      access_by_lua_block {
        local edge = require "edge"
        edge.handle_request()
      }

      # Dynamic proxy pass tới customer origin
      proxy_pass $origin_url;

      # Set Basic Auth nếu có
      proxy_set_header Authorization $auth_header;

      # Preserve original headers
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # Caching với customer isolation
      include generic_conf/define_cache.conf;

      # Response headers
      add_header X-Edge MultiTenant;
      add_header X-Customer-ID $cache_key;
    }

    include generic_conf/basic_vts_location.conf;
  }
}
```

#### 2.3 Cache Configuration Enhancement
**Có thể sửa: `generic_conf/setup_cache.conf`**

Tăng cache size cho multi-tenant:
```nginx
proxy_cache_path /cache/ levels=2:2 keys_zone=zone_1:100m max_size=1g inactive=1h use_temp_path=off;
proxy_cache_lock_timeout 5s;
proxy_cache_use_stale error timeout updating;
proxy_read_timeout 10s;
proxy_send_timeout 10s;
proxy_ignore_client_abort on;
```

---

### **Phase 3: Admin API (Tuần 2-3)**

#### 3.1 Admin API Service (Python Flask)
**Tạo file: `admin_api/app.py`**

```python
from flask import Flask, request, jsonify
import json
import os
import subprocess

app = Flask(__name__)
CONFIG_FILE = '/app/config/customers.json'

def load_customers():
    """Load customers from JSON file"""
    with open(CONFIG_FILE, 'r') as f:
        return json.load(f)

def save_customers(customers):
    """Save customers to JSON file"""
    with open(CONFIG_FILE, 'w') as f:
        json.dump(customers, f, indent=2)

    # Reload nginx config (reload customer configs)
    # Note: Cần implement reload mechanism
    # Option 1: nginx reload (graceful)
    # Option 2: Shared dict reload via API call
    pass

@app.route('/api/customers', methods=['GET'])
def list_customers():
    """List all customers"""
    customers = load_customers()
    return jsonify({
        'customers': [
            {
                'customer_id': k,
                'origin_url': v.get('origin_url'),
                'custom_domain': v.get('custom_domain'),
                'cache_ttl': v.get('cache_ttl')
            }
            for k, v in customers.items()
        ]
    })

@app.route('/api/customers', methods=['POST'])
def create_customer():
    """Create new customer"""
    data = request.json

    # Validation
    required_fields = ['customer_id', 'origin_url', 'custom_domain']
    for field in required_fields:
        if field not in data:
            return jsonify({'error': f'Missing field: {field}'}), 400

    customers = load_customers()

    # Check if customer_id already exists
    if data['customer_id'] in customers:
        return jsonify({'error': 'Customer ID already exists'}), 409

    # Add customer
    customer_config = {
        'origin_url': data['origin_url'],
        'custom_domain': data['custom_domain'],
        'cache_ttl': data.get('cache_ttl', 3600),
        'basic_auth': data.get('basic_auth', None)
    }

    customers[data['customer_id']] = customer_config
    save_customers(customers)

    return jsonify({
        'message': 'Customer created successfully',
        'customer_id': data['customer_id'],
        'cdn_url': f"https://{data['custom_domain']}"
    }), 201

@app.route('/api/customers/<customer_id>', methods=['GET'])
def get_customer(customer_id):
    """Get customer details"""
    customers = load_customers()

    if customer_id not in customers:
        return jsonify({'error': 'Customer not found'}), 404

    customer = customers[customer_id]
    customer['customer_id'] = customer_id

    return jsonify(customer)

@app.route('/api/customers/<customer_id>', methods=['PUT'])
def update_customer(customer_id):
    """Update customer configuration"""
    customers = load_customers()

    if customer_id not in customers:
        return jsonify({'error': 'Customer not found'}), 404

    data = request.json

    # Update fields
    if 'origin_url' in data:
        customers[customer_id]['origin_url'] = data['origin_url']
    if 'custom_domain' in data:
        customers[customer_id]['custom_domain'] = data['custom_domain']
    if 'cache_ttl' in data:
        customers[customer_id]['cache_ttl'] = data['cache_ttl']
    if 'basic_auth' in data:
        customers[customer_id]['basic_auth'] = data['basic_auth']

    save_customers(customers)

    return jsonify({
        'message': 'Customer updated successfully',
        'customer_id': customer_id
    })

@app.route('/api/customers/<customer_id>', methods=['DELETE'])
def delete_customer(customer_id):
    """Delete customer"""
    customers = load_customers()

    if customer_id not in customers:
        return jsonify({'error': 'Customer not found'}), 404

    del customers[customer_id]
    save_customers(customers)

    return jsonify({
        'message': 'Customer deleted successfully',
        'customer_id': customer_id
    })

@app.route('/api/customers/<customer_id>/cache', methods=['DELETE'])
def purge_customer_cache(customer_id):
    """Purge cache for specific customer"""
    customers = load_customers()

    if customer_id not in customers:
        return jsonify({'error': 'Customer not found'}), 404

    # TODO: Implement cache purge logic
    # Option 1: Delete files matching pattern /cache/**/customer_id:*
    # Option 2: Use nginx cache purge module
    # Option 3: Clear via nginx API

    cache_path = '/cache'
    purged_count = 0

    # Simple implementation: find and delete cache files
    # In production, use proper cache purge mechanism
    try:
        import glob
        pattern = f"{cache_path}/**/*"
        files = glob.glob(pattern, recursive=True)

        for file in files:
            # Cache key format: customer_id:uri
            # Need to check cache metadata
            # Simplified: just count for now
            purged_count += 1

        return jsonify({
            'message': 'Cache purged successfully',
            'customer_id': customer_id,
            'purged_files': purged_count
        })
    except Exception as e:
        return jsonify({
            'error': 'Failed to purge cache',
            'detail': str(e)
        }), 500

@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint"""
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

#### 3.2 Admin API Dependencies
**Tạo file: `admin_api/requirements.txt`**

```
flask==2.3.0
gunicorn==21.2.0
```

#### 3.3 Admin API Dockerfile
**Tạo file: `admin_api/Dockerfile`**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

#### 3.4 Admin API Nginx Config
**Tạo file: `nginx_admin.conf`**

```nginx
events {
  worker_connections 1024;
}

error_log stderr;

http {
  include generic_conf/setup_logging.conf;

  server {
    listen 9000;

    location /api/ {
      proxy_pass http://admin_api:5000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }

    location /health {
      proxy_pass http://admin_api:5000/health;
    }
  }
}
```

#### 3.5 Update Docker Compose
**Sửa file: `docker-compose.yaml`**

Thêm services:
```yaml
  admin_api:
    build: ./admin_api
    volumes:
      - "./config/customers.json:/app/config/customers.json"
    environment:
      - FLASK_ENV=production

  admin:
    extends:
      service: nginx_base
    volumes:
      - "./nginx_admin.conf:/usr/local/openresty/nginx/conf/nginx.conf"
    ports:
      - "9000:9000"
    depends_on:
      - admin_api
```

Mount customers.json vào edge nodes:
```yaml
  edge:
    extends:
      service: nginx_base
    volumes:
      - "./nginx_edge.conf:/usr/local/openresty/nginx/conf/nginx.conf"
      - "./config/customers.json:/config/customers.json:ro"  # Add this
    depends_on:
      - backend
      - backend1
    ports:
      - "8081:8080"
```

---

### **Phase 4: Monitoring Per Customer (Tuần 3-4)**

#### 4.1 Enhanced Metrics Collection
**Sửa: `src/edge.lua`** (thêm vào `handle_request` function)

```lua
-- Log customer-specific metrics
ngx.log(ngx.INFO, string.format(
  'customer=%s uri=%s cache_status=%s origin=%s',
  customer.customer_id,
  ngx.var.uri,
  ngx.var.upstream_cache_status or 'N/A',
  customer.origin_url
))
```

#### 4.2 Prometheus Configuration
**Sửa: `config/prometheus.yml`**

Thêm relabeling để extract customer_id từ metrics:
```yaml
scrape_configs:
  - job_name: 'edge-nodes'
    static_configs:
      - targets:
        - 'edge:8080'
        - 'edge1:8080'
        - 'edge2:8080'
    metric_relabel_configs:
      - source_labels: [host]
        target_label: customer_domain
```

#### 4.3 Grafana Dashboard
**Tạo: `config/grafana/multi-tenant-dashboard.json`**

Dashboard panels:
1. **Request Rate per Customer**
   - Query: `rate(nginx_vts_server_requests_total{customer_domain!=""}[5m])`
   - Visualization: Bar chart

2. **Cache Hit Ratio per Customer**
   - Query:
     ```
     sum(rate(nginx_vts_server_cache_total{status="hit"}[5m])) by (customer_domain) /
     sum(rate(nginx_vts_server_cache_total[5m])) by (customer_domain)
     ```
   - Visualization: Gauge (0-100%)

3. **Bandwidth Usage per Customer**
   - Query: `rate(nginx_vts_server_bytes_total[5m])`
   - Visualization: Time series

4. **Top Customers by Traffic**
   - Query: `topk(10, sum(rate(nginx_vts_server_requests_total[5m])) by (customer_domain))`
   - Visualization: Table

5. **Origin Response Time per Customer**
   - Query: `histogram_quantile(0.95, nginx_vts_upstream_response_seconds)`
   - Visualization: Heatmap

6. **Active Customers**
   - Query: `count(count by (customer_domain) (nginx_vts_server_requests_total))`
   - Visualization: Stat

---

### **Phase 5: Testing & Demo Setup (Tuần 4)**

#### 5.1 Mock Customer Origins
**Tạo file: `nginx_mock_origin.conf`**

```nginx
# Mock origin server để test
events {
  worker_connections 1024;
}

http {
  server {
    listen 8080;
    server_name _;

    location / {
      # Simulate origin response
      add_header Content-Type application/json;
      add_header Cache-Control "public, max-age=3600";

      return 200 '{"origin": "$server_name", "uri": "$request_uri", "timestamp": "$time_iso8601"}';
    }
  }
}
```

Thêm vào docker-compose.yaml:
```yaml
  mock_origin_a:
    extends:
      service: nginx_base
    volumes:
      - "./nginx_mock_origin.conf:/usr/local/openresty/nginx/conf/nginx.conf"
    environment:
      - ORIGIN_NAME=customer-a
    ports:
      - "8090:8080"

  mock_origin_b:
    extends:
      service: nginx_base
    volumes:
      - "./nginx_mock_origin.conf:/usr/local/openresty/nginx/conf/nginx.conf"
    environment:
      - ORIGIN_NAME=customer-b
    ports:
      - "8091:8080"

  mock_origin_c:
    extends:
      service: nginx_base
    volumes:
      - "./nginx_mock_origin.conf:/usr/local/openresty/nginx/conf/nginx.conf"
    environment:
      - ORIGIN_NAME=customer-c
    ports:
      - "8092:8080"
```

#### 5.2 Test Customers Configuration
Update `config/customers.json` với mock origins:
```json
{
  "customer-a": {
    "origin_url": "http://mock_origin_a:8080",
    "custom_domain": "cdn.customer-a.local",
    "cache_ttl": 3600,
    "basic_auth": null
  },
  "customer-b": {
    "origin_url": "http://mock_origin_b:8080",
    "custom_domain": "cdn.customer-b.local",
    "cache_ttl": 1800,
    "basic_auth": null
  },
  "customer-c": {
    "origin_url": "http://mock_origin_c:8080",
    "custom_domain": "cdn.customer-c.local",
    "cache_ttl": 7200,
    "basic_auth": null
  }
}
```

#### 5.3 Multi-Tenant Load Test
**Tạo file: `load_test_multitenant.sh`**

```bash
#!/bin/bash

# Multi-tenant load test script

EDGE_URL="http://localhost:8081"
DURATION="30s"
CONNECTIONS=10
THREADS=2

echo "=== Multi-Tenant CDN Load Test ==="
echo ""

# Test Customer A
echo "Testing Customer A..."
wrk -t${THREADS} -c${CONNECTIONS} -d${DURATION} \
  -H "Host: cdn.customer-a.local" \
  ${EDGE_URL}/test/customer-a/asset.json

echo ""

# Test Customer B
echo "Testing Customer B..."
wrk -t${THREADS} -c${CONNECTIONS} -d${DURATION} \
  -H "Host: cdn.customer-b.local" \
  ${EDGE_URL}/test/customer-b/asset.json

echo ""

# Test Customer C
echo "Testing Customer C..."
wrk -t${THREADS} -c${CONNECTIONS} -d${DURATION} \
  -H "Host: cdn.customer-c.local" \
  ${EDGE_URL}/test/customer-c/asset.json

echo ""
echo "=== Concurrent Multi-Customer Test ==="

# Concurrent test với tất cả customers
cat > /tmp/multitenant_wrk.lua <<'EOF'
request = function()
  customers = {"cdn.customer-a.local", "cdn.customer-b.local", "cdn.customer-c.local"}
  paths = {"/image.jpg", "/video.mp4", "/data.json", "/style.css"}

  customer = customers[math.random(#customers)]
  path = paths[math.random(#paths)]

  return wrk.format("GET", path, {["Host"] = customer})
end
EOF

wrk -t${THREADS} -c${CONNECTIONS} -d${DURATION} \
  -s /tmp/multitenant_wrk.lua \
  ${EDGE_URL}

echo ""
echo "Test completed!"
```

#### 5.4 Test Scripts
**Tạo file: `test_admin_api.sh`**

```bash
#!/bin/bash

API_URL="http://localhost:9000/api"

echo "=== Testing Admin API ==="
echo ""

# 1. List all customers
echo "1. Listing all customers..."
curl -s ${API_URL}/customers | jq .
echo ""

# 2. Get specific customer
echo "2. Getting customer-a details..."
curl -s ${API_URL}/customers/customer-a | jq .
echo ""

# 3. Create new customer
echo "3. Creating new customer (test-customer)..."
curl -s -X POST ${API_URL}/customers \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "test-customer",
    "origin_url": "http://example.com",
    "custom_domain": "cdn.test-customer.local",
    "cache_ttl": 1800
  }' | jq .
echo ""

# 4. Update customer
echo "4. Updating test-customer..."
curl -s -X PUT ${API_URL}/customers/test-customer \
  -H "Content-Type: application/json" \
  -d '{
    "cache_ttl": 3600
  }' | jq .
echo ""

# 5. Purge cache
echo "5. Purging cache for test-customer..."
curl -s -X DELETE ${API_URL}/customers/test-customer/cache | jq .
echo ""

# 6. Delete customer
echo "6. Deleting test-customer..."
curl -s -X DELETE ${API_URL}/customers/test-customer | jq .
echo ""

echo "Admin API tests completed!"
```

#### 5.5 Integration Test
**Tạo file: `test_integration.sh`**

```bash
#!/bin/bash

EDGE_URL="http://localhost:8081"

echo "=== CDN Integration Tests ==="
echo ""

# Test 1: Cache isolation
echo "Test 1: Cache Isolation"
echo "Requesting same URI from different customers..."

curl -s -H "Host: cdn.customer-a.local" ${EDGE_URL}/test.json > /tmp/response_a.json
curl -s -H "Host: cdn.customer-b.local" ${EDGE_URL}/test.json > /tmp/response_b.json

echo "Customer A response:"
cat /tmp/response_a.json | jq .
echo ""
echo "Customer B response:"
cat /tmp/response_b.json | jq .
echo ""

# Test 2: Cache hit
echo "Test 2: Cache Hit Test"
echo "First request (should be MISS)..."
curl -s -I -H "Host: cdn.customer-a.local" ${EDGE_URL}/cached.json | grep X-Cache-Status

echo "Second request (should be HIT)..."
curl -s -I -H "Host: cdn.customer-a.local" ${EDGE_URL}/cached.json | grep X-Cache-Status
echo ""

# Test 3: Customer not found
echo "Test 3: Invalid Customer"
curl -s -H "Host: invalid.customer.local" ${EDGE_URL}/test.json
echo ""

# Test 4: Origin failover
echo "Test 4: Origin Error Handling"
curl -s -H "Host: cdn.customer-a.local" ${EDGE_URL}/non-existent-path
echo ""

echo "Integration tests completed!"
```

---

## Files Summary

### Tạo mới (15 files):

#### Configuration:
1. `config/customers.json` - Customer configurations
2. `config/grafana/multi-tenant-dashboard.json` - Grafana dashboard

#### Source Code:
3. `src/customer_manager.lua` - Customer management module

#### Admin API:
4. `admin_api/app.py` - Flask REST API
5. `admin_api/requirements.txt` - Python dependencies
6. `admin_api/Dockerfile` - Container image

#### Nginx Configs:
7. `nginx_admin.conf` - Admin API nginx config
8. `nginx_mock_origin.conf` - Mock origin servers

#### Testing:
9. `load_test_multitenant.sh` - Multi-tenant load tests
10. `test_admin_api.sh` - Admin API tests
11. `test_integration.sh` - Integration tests

#### Documentation:
12. `docs/MULTITENANT_POC_PLAN.md` - This file
13. `docs/MULTITENANT_API.md` - API documentation
14. `docs/CUSTOMER_ONBOARDING.md` - Customer onboarding guide
15. `docs/OPERATIONS.md` - Operations runbook

### Sửa đổi (4 files):

1. `src/edge.lua` - Multi-tenant request handling
2. `nginx_edge.conf` - Dynamic upstream & custom domains
3. `docker-compose.yaml` - Add services (admin_api, mock_origins)
4. `config/prometheus.yml` - Per-customer metrics labeling

---

## Timeline: 4 tuần

| Tuần | Nhiệm vụ | Deliverables | Giờ ước tính |
|------|----------|--------------|--------------|
| **1** | Customer Config + Edge Multi-Tenant | - customers.json<br>- customer_manager.lua<br>- edge.lua updates<br>- nginx_edge.conf updates<br>- Working customer lookup<br>- Cache isolation working | 32-40h |
| **2** | Admin API Development | - Flask API<br>- CRUD endpoints<br>- Docker setup<br>- nginx_admin.conf<br>- Basic API tests | 24-32h |
| **3** | Cache Purge + Monitoring | - Cache purge endpoint<br>- Grafana dashboard<br>- Per-customer metrics<br>- Prometheus config | 24-32h |
| **4** | Testing + Documentation | - Mock origins<br>- Load tests<br>- Integration tests<br>- API docs<br>- Onboarding guide<br>- Operations runbook<br>- 3-customer demo | 24-32h |

**Total: 104-136 giờ (13-17 ngày làm việc)**

---

## POC Success Criteria

### Functional Requirements:
- ✅ **3 customers working** với origins khác nhau
- ✅ **Cache isolation** - customer A cache không affect customer B
- ✅ **Custom domain** - cdn.customer-a.com routes correctly
- ✅ **Basic Auth** - Origin requests include correct credentials
- ✅ **Admin API** - Add/remove customers without code changes
- ✅ **Cache purge** - Can purge cache per customer
- ✅ **Monitoring** - Grafana shows per-customer metrics

### Performance Requirements:
- ✅ **Cache hit ratio** > 80% for static content
- ✅ **Response time** < 100ms for cached content
- ✅ **Concurrent customers** - Handle 10+ customers simultaneously
- ✅ **Load handling** - 1000+ req/s across all customers

### Operational Requirements:
- ✅ **Zero downtime** - Add/remove customers without restart
- ✅ **Easy debugging** - Clear logs with customer identification
- ✅ **Documentation** - Complete API docs và operation guides
- ✅ **Monitoring** - Per-customer dashboards working

---

## Testing Checklist

### Unit Tests:
- [ ] customer_manager.lua functions
- [ ] edge.lua request handling
- [ ] Cache key generation
- [ ] Basic Auth encoding

### Integration Tests:
- [ ] Customer identification from domain
- [ ] Dynamic proxy to correct origin
- [ ] Cache isolation between customers
- [ ] Basic Auth headers sent correctly
- [ ] Cache hit/miss behavior
- [ ] Error handling (customer not found, origin down)

### API Tests:
- [ ] Create customer
- [ ] List customers
- [ ] Get customer details
- [ ] Update customer
- [ ] Delete customer
- [ ] Purge customer cache
- [ ] Error responses (404, 409, 400)

### Load Tests:
- [ ] Single customer sustained load
- [ ] Multi-customer concurrent load
- [ ] Cache effectiveness
- [ ] Resource usage (CPU, memory, disk)

### Manual Tests:
- [ ] Custom domain routing
- [ ] Grafana dashboard displays correct data
- [ ] Log output includes customer_id
- [ ] Cache files have correct prefixes
- [ ] Config reload works without restart

---

## Demo Scenario

### Setup (Before Demo):
1. Start all services: `docker-compose up -d`
2. Verify 3 customers configured in customers.json
3. Open Grafana dashboard (http://localhost:9091)
4. Open Prometheus (http://localhost:9090)

### Demo Flow:

**1. Show Architecture (5 min)**
- Explain multi-tenant architecture
- Show customer isolation concept
- Explain cache key strategy

**2. Customer Onboarding (5 min)**
```bash
# Add new customer via API
curl -X POST http://localhost:9000/api/customers \
  -H "Content-Type: application/json" \
  -d '{
    "customer_id": "demo-customer",
    "origin_url": "http://mock_origin_a:8080",
    "custom_domain": "cdn.demo.local",
    "cache_ttl": 3600
  }'
```

**3. Test Request Routing (5 min)**
```bash
# Send request to customer A
curl -H "Host: cdn.customer-a.local" http://localhost:8081/test.json

# Send request to customer B (same URI, different origin)
curl -H "Host: cdn.customer-b.local" http://localhost:8081/test.json

# Show different responses → cache isolation working
```

**4. Cache Behavior (5 min)**
```bash
# First request - cache MISS
curl -I -H "Host: cdn.customer-a.local" http://localhost:8081/image.jpg
# X-Cache-Status: MISS

# Second request - cache HIT
curl -I -H "Host: cdn.customer-a.local" http://localhost:8081/image.jpg
# X-Cache-Status: HIT
```

**5. Monitoring Dashboard (5 min)**
- Open Grafana dashboard
- Show per-customer metrics:
  - Request rates
  - Cache hit ratios
  - Bandwidth usage
  - Active customers

**6. Cache Purge (3 min)**
```bash
# Purge cache for specific customer
curl -X DELETE http://localhost:9000/api/customers/customer-a/cache

# Verify next request is MISS
curl -I -H "Host: cdn.customer-a.local" http://localhost:8081/image.jpg
# X-Cache-Status: MISS
```

**7. Load Test (5 min)**
```bash
# Run multi-tenant load test
./load_test_multitenant.sh

# Show results in Grafana
```

**8. Management Operations (3 min)**
```bash
# List all customers
curl http://localhost:9000/api/customers

# Update customer config
curl -X PUT http://localhost:9000/api/customers/customer-a \
  -H "Content-Type: application/json" \
  -d '{"cache_ttl": 7200}'

# Delete customer
curl -X DELETE http://localhost:9000/api/customers/demo-customer
```

**Total Demo Time: ~35 minutes**

---

## Next Steps After POC

### Phase 2 Enhancements (Post-POC):

1. **Security & Authentication**
   - API authentication (API keys, JWT)
   - Customer-specific API keys for cache purge
   - Rate limiting per customer
   - DDoS protection
   - IP whitelisting per customer

2. **SSL/TLS Support**
   - Per-customer SSL certificates
   - Let's Encrypt integration
   - Automatic cert renewal
   - SNI support

3. **Advanced Caching**
   - Custom cache rules per customer (by path, headers)
   - Cache warming
   - Selective cache purge (by pattern, tags)
   - Cache size limits per customer
   - TTL override rules

4. **Geographic Distribution**
   - Multiple PoP locations
   - GeoDNS routing
   - Edge location selection based on client IP
   - Multi-region monitoring

5. **Customer Portal**
   - Self-service dashboard
   - Real-time analytics
   - Cache management UI
   - Usage reports
   - Billing integration

6. **Advanced Monitoring**
   - Real-time alerting per customer
   - SLA monitoring
   - Detailed analytics (top URLs, traffic patterns)
   - Cost allocation
   - Anomaly detection

7. **Performance Optimization**
   - HTTP/2, HTTP/3 support
   - Image optimization (resize, format conversion)
   - Compression (Brotli, gzip)
   - Pre-fetching and cache warming
   - Edge computing (Lua functions per customer)

8. **Reliability**
   - Multi-origin support with failover
   - Health checks per origin
   - Automatic retry logic
   - Graceful degradation
   - Origin shield layer

9. **Developer Experience**
   - SDKs for popular languages
   - Webhooks for events (cache purge, origin errors)
   - API versioning
   - Detailed API documentation
   - Terraform provider

10. **Compliance & Logging**
    - Access logs per customer
    - Audit logs for admin operations
    - GDPR compliance features
    - Log export to customer storage
    - Data retention policies

---

## Cost Estimation

### Infrastructure (per month):

**POC Environment:**
- 3x Edge nodes (2 CPU, 4GB RAM each): ~$30/month
- 1x Admin API (1 CPU, 1GB RAM): ~$5/month
- 1x Monitoring (2 CPU, 4GB RAM): ~$10/month
- **Total: ~$45/month**

**Production Environment (for 100 customers):**
- 10x Edge nodes (4 CPU, 8GB RAM each): ~$400/month
- 2x Admin API (HA): ~$20/month
- Monitoring (Prometheus + Grafana): ~$50/month
- Load Balancer: ~$20/month
- Storage (cache): ~$50/month
- **Total: ~$540/month**

### Scaling (per 100 additional customers):
- +2-3 Edge nodes: ~$80-120/month
- +Storage: ~$20/month
- **Total: ~$100-140/month per 100 customers**

---

## Risk Assessment

### Technical Risks:

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Cache key collision | Low | High | Use customer_id prefix, test thoroughly |
| Origin timeout affects all customers | Medium | High | Implement per-customer timeouts, circuit breaker |
| Config reload causes brief downtime | Medium | Medium | Use shared dict, hot reload without restart |
| Cache size exceeded | Medium | Medium | Implement LRU eviction, per-customer quotas |
| DNS resolution slow for origins | Low | Medium | Cache DNS results, use TTL |

### Operational Risks:

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Customer misconfigures origin URL | High | Low | Validate URLs in API, health checks |
| Cache purge API abused | Medium | Medium | Rate limiting, authentication |
| Monitoring dashboard overloaded | Low | Low | Pagination, time range limits |
| Customer origins go down | High | Low | Serve stale cache, alert customer |

### Business Risks:

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Customer demand exceeds capacity | Medium | High | Auto-scaling, capacity planning |
| One customer monopolizes resources | Medium | Medium | Per-customer quotas, rate limiting |
| Security breach | Low | High | Security audit, penetration testing |
| Data loss (cache) | Low | Medium | Redundant storage, backups |

---

## Support & Maintenance

### Daily Operations:
- Monitor dashboard for anomalies
- Check error logs
- Verify all customers healthy
- Review cache hit ratios

### Weekly:
- Analyze traffic patterns
- Review customer growth
- Check resource utilization
- Update documentation

### Monthly:
- Customer usage reports
- Capacity planning review
- Security updates
- Performance optimization

### On-Demand:
- Customer onboarding/offboarding
- Cache purge requests
- Troubleshooting issues
- Configuration changes

---

## Appendix

### A. Useful Commands

```bash
# Start POC environment
docker-compose up -d

# View edge logs
docker-compose logs -f edge

# Check customer config
cat config/customers.json | jq .

# Test customer routing
curl -H "Host: cdn.customer-a.local" http://localhost:8081/test

# Check cache status
ls -lh /var/lib/docker/volumes/cdn-cache/

# Reload nginx config
docker-compose exec edge nginx -s reload

# View metrics
curl http://localhost:8081/status/format/prometheus
```

### B. Troubleshooting

**Problem: Customer not found error**
- Check customers.json has correct domain
- Verify shared dict loaded: `nginx -T | grep lua_shared_dict`
- Check nginx error logs

**Problem: Cache not working**
- Verify Cache-Control headers from origin
- Check cache_key is unique per customer
- Verify cache path writable: `ls -la /cache`

**Problem: Origin timeout**
- Check origin URL is accessible from edge container
- Verify DNS resolution working
- Increase proxy_read_timeout if needed

**Problem: Slow performance**
- Check cache hit ratio in Grafana
- Verify cache size not exceeded
- Check network latency to origins

### C. Reference Links

- Nginx documentation: https://nginx.org/en/docs/
- OpenResty (Nginx + Lua): https://openresty.org/
- Prometheus: https://prometheus.io/docs/
- Grafana: https://grafana.com/docs/
- Flask: https://flask.palletsprojects.com/

---

**Document Version**: 1.0
**Last Updated**: 2025-09-30
**Author**: POC Team
**Status**: Draft