# React + NestJS + MongoDB + Redis Cache Demo

A standalone cache-aside demo with a React + TypeScript UI, NestJS REST API, MongoDB product collection, and Redis TTL cache.

## Tech stack
- React + TypeScript + Vite
- MUI search/filter UI and AG Grid
- NestJS REST API + Swagger
- MongoDB 7 with a seed script for 1,000,000 products and indexes
- Redis 7 cache-aside with a default 60-second TTL and cache invalidation
- Docker Compose: web, api, mongodb, redis

## Run with Docker
```bash
docker compose up --build -d
docker compose exec api npm run seed
```
Seeding one million records can take time depending on disk and CPU. Progress is printed in the logs.

- UI: http://localhost:8080
- Swagger: http://localhost:3000/api/docs
- API: http://localhost:3000/api/products?page=1&limit=50
- MongoDB: localhost:27017
- Redis: localhost:6379

Stop:
```bash
docker compose down
```
To delete data volumes too, run `docker compose down -v`.

## Local development
Requires Node.js 20+ and running MongoDB/Redis instances.
```bash
npm install
npm run dev
```
Frontend dev server: http://localhost:5173, Swagger: http://localhost:3000/api/docs

## Cache behavior
1. `GET /api/products` calculates a cache key from search, filters, and pagination.
2. On a Redis HIT, cached data is returned without querying MongoDB.
3. On a MISS, MongoDB returns the page and total count; the result is stored in Redis with a TTL.
4. `DELETE /api/products/cache` invalidates keys matching `products:v1:*`.
5. If Redis is unavailable, the API skips caching and uses MongoDB.

Use `cacheHit` and `durationMs` in the response to compare HIT/MISS and API processing time. Search uses an example substring regex; for large production workloads, consider MongoDB Atlas Search or a dedicated indexed/tokenized search strategy. Pagination uses skip/limit for learning purposes; cursor-based pagination is usually better for very deep pages.

## Main endpoints
- `GET /api/products?page=1&limit=50&search=product&category=Books&status=active`
- `DELETE /api/products/cache`
- Swagger UI: `/api/docs`
