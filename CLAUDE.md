# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a learning repository that demonstrates how to build a CDN (Content Delivery Network) from scratch using nginx, lua, docker, prometheus, and grafana. The project incrementally builds from a simple backend service to a multi-node, observable CDN with load balancing and caching capabilities.

## Architecture

The CDN consists of multiple nginx-based services:

- **Backend services** (`backend`, `backend1`): Origin servers that generate JSON API responses with simulated latency
- **Edge services** (`edge`, `edge1`, `edge2`): CDN edge nodes that cache content from backends
- **Load balancer** (`loadbalancer`): Distributes requests across edge nodes using round-robin or consistent hashing
- **Monitoring**: Prometheus + Grafana stack for metrics collection and visualization

Key components:
- All services run nginx with OpenResty (lua integration)
- VTS module provides nginx metrics in Prometheus format
- Configuration is modularized using nginx `include` directives
- Lua code handles content generation, caching policies, and load balancing logic

## Common Commands

### Development and Testing
```bash
# Start all services (backend, edges, load balancer, monitoring)
docker-compose up

# Start specific service (e.g., just backend for testing)
docker-compose run --rm --service-ports backend

# Run load tests (uses wrk with lua scripts)
./load_test.sh

# Check specific git tag/configuration
git checkout 2.1.0  # or other version tags
docker-compose up
```

### Service Endpoints
- Backend: http://localhost:8080, http://localhost:8180
- Edges: http://localhost:8081, http://localhost:8082, http://localhost:8083
- Load balancer: http://localhost:18080
- Prometheus: http://localhost:9090
- Grafana: http://localhost:9091 (admin/admin)

### Metrics and Status
- nginx VTS status: `http://localhost:PORT/status/format/html`
- Prometheus metrics: `http://localhost:PORT/status/format/prometheus`

## Code Structure

- `/src/`: Lua modules for different components
  - `backend.lua`: Backend content generation with cache headers
  - `edge.lua`: Edge caching logic
  - `loadbalancer.lua`: Load balancing algorithms
  - `simulations.lua`: Latency simulation (percentile-based)
  - `load_tests.lua`: wrk load test scripts with long-tail distribution
- `/generic_conf/`: Reusable nginx configuration snippets
- `/config/`: Prometheus configuration
- Root nginx config files: `nginx_backend.conf`, `nginx_edge.conf`, `nginx_loadbalancer.conf`

## Key Concepts

- **Caching behavior**: Controlled via `Cache-Control` headers and nginx `proxy_cache` directives
- **Cache keys**: Uses `proxy_cache_key` for content identification
- **Load balancing**: Supports round-robin and consistent hashing via nginx upstream
- **Metrics collection**: VTS module exposes nginx metrics to Prometheus
- **Latency simulation**: Lua code simulates realistic response times using percentile profiles
- **Long-tail distribution**: Load tests model realistic traffic patterns (96% requests to top 5% content)

## Testing Different Configurations

The project uses git tags to demonstrate different CDN configurations:
- `1.0.0`: Basic backend service
- `2.1.0`: Edge caching enabled
- `4.0.0`: Round-robin load balancing
- `4.0.1`: Consistent hashing load balancing

Use `git checkout <tag>` followed by `docker-compose up` to test specific configurations.