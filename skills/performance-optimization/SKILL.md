---
name: performance-optimization
description: "Senior-level performance optimization covering profiling, caching, database tuning, frontend performance, and scalability. Use this skill whenever optimizing slow code, profiling performance bottlenecks, implementing caching, tuning database queries, optimizing bundle size, improving page load times, or discussing performance benchmarks. Also trigger when the user mentions slow API, high latency, memory leak, CPU usage, load testing, benchmarking, Core Web Vitals, or any performance concern."
---

# Performance Optimization — Senior Engineer Standards

You are a senior performance engineer. Never optimize without measuring first. Never measure without understanding what matters.

## The Golden Rule

**Measure → Identify → Optimize → Verify → Repeat**

Never optimize based on intuition. Profile first, find the actual bottleneck, fix it, measure again to confirm improvement.

## Backend Performance

### Profiling first
- **Go**: `pprof` for CPU and memory profiling. `go tool trace` for concurrency.
- **Python**: `cProfile` + `snakeviz` for CPU. `memory_profiler` for memory. `py-spy` for production.
- **Node.js**: `--inspect` flag + Chrome DevTools. `clinic.js` for production diagnostics.

### Common bottlenecks (in order of frequency)
1. **Database queries**: N+1, missing indexes, full table scans, unoptimized joins
2. **External API calls**: No timeout, no caching, no circuit breaker
3. **Serialization**: Converting large objects to JSON repeatedly
4. **Memory allocation**: Creating objects in hot loops
5. **Blocking I/O**: Synchronous file reads, DNS lookups in request path

### Caching strategy
- **Cache-aside**: App checks cache → miss → fetch from source → populate cache. Most common.
- **Write-through**: Write to cache AND source simultaneously. Consistent but slower writes.
- **Write-behind**: Write to cache, async flush to source. Fast writes, risk of data loss.

Cache invalidation approaches:
- **TTL-based**: Set expiry. Simple, eventually consistent. Best default.
- **Event-based**: Invalidate on write events. More complex, more consistent.
- **Version-based**: Include version in cache key. Bust by incrementing version.

### Connection pooling
- Database: min 5, max 25 connections per instance. Tune based on load testing.
- HTTP clients: Reuse connections (`keep-alive`). Set pool size limits.
- Redis: Use connection pool, not new connection per request.

## Database Performance

### Query optimization checklist
1. Run `EXPLAIN ANALYZE` on any query touching > 1K rows
2. Check for sequential scans on large tables (add index)
3. Verify index usage (sometimes the planner ignores indexes)
4. Check for lock contention (`pg_stat_activity`)
5. Monitor slow query log (queries > 100ms)

### Indexing rules
- Index foreign keys, WHERE columns, JOIN columns, ORDER BY columns
- Composite indexes: most selective column first, range column last
- Partial indexes for filtered queries (`WHERE status = 'active'`)
- Cover indexes to avoid table lookups for read-heavy queries
- Don't over-index: each index slows writes and uses disk

### Pagination
- NEVER use `OFFSET` for large datasets. It scans and discards rows.
- Use cursor-based: `WHERE id > last_id ORDER BY id LIMIT 20`
- For complex sorting: `WHERE (sort_col, id) > (last_sort_val, last_id)`

## Frontend Performance

### Core Web Vitals targets
- **LCP (Largest Contentful Paint)**: < 2.5s. Optimize images, fonts, critical CSS.
- **INP (Interaction to Next Paint)**: < 200ms. Avoid long tasks, use `requestIdleCallback`.
- **CLS (Cumulative Layout Shift)**: < 0.1. Set dimensions on images/embeds, avoid dynamic content injection above the fold.

### Bundle optimization
- Code split at route level (`React.lazy` + `Suspense`)
- Tree-shake unused exports (check with `webpack-bundle-analyzer` or `source-map-explorer`)
- Lazy load heavy components: charts, maps, editors, video players
- Dynamic imports for conditional features: `const Editor = lazy(() => import('./Editor'))`
- Target < 200KB initial JS bundle (gzipped)

### Image optimization
- Use `next/image` or `<picture>` with srcset for responsive images
- WebP/AVIF format with JPEG fallback
- Lazy load below-fold images: `loading="lazy"`
- Specify width/height to prevent CLS
- Use CDN with automatic format conversion and resizing

### Network optimization
- Preload critical resources: `<link rel="preload">`
- Prefetch next-page data on hover/focus
- Use HTTP/2 or HTTP/3 for multiplexing
- Set proper cache headers: immutable for hashed assets, short TTL for HTML
- Compress responses: Brotli > gzip

## Load Testing

### When to load test
- Before launching a new service
- Before major traffic events (sales, campaigns)
- After significant architecture changes
- Regularly (monthly) for critical services

### What to measure
- Throughput: requests per second at acceptable latency
- Latency: p50, p95, p99 under load
- Error rate: at what load do errors start?
- Saturation point: where does the system break?

### Tools
- **k6**: Script-based, developer-friendly, good for CI integration
- **Locust**: Python-based, good for complex user flows
- **Artillery**: YAML config, good for quick tests

### Load test pattern
1. Baseline: current production traffic level
2. Ramp: gradually increase to 2x baseline
3. Spike: sudden burst to 5x baseline
4. Soak: sustained load at 1.5x baseline for 1 hour
5. Measure: latency, errors, resource utilization at each phase

## Memory Optimization

- **Memory leaks**: Event listeners not cleaned up, closures holding references, growing caches without eviction, timers not cleared
- **Detection**: Heap snapshots (Chrome DevTools), `process.memoryUsage()` trending, `pprof` heap profiles in Go
- **Prevention**: Weak references for caches, bounded data structures, explicit cleanup in component unmount/goroutine shutdown