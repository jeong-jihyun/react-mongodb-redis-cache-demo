# React + NestJS + MongoDB + Redis 캐시 데모

React 기반 상품 조회 화면과 NestJS API를 통해 **MongoDB 조회 결과를 Redis에 캐싱하는 Cache-Aside 패턴**을 확인하는 독립 실행형 예제입니다. 검색·필터·페이지네이션을 바꿔가며 캐시 HIT/MISS와 응답 시간을 비교할 수 있습니다.

## 1. 주요 기능

- **React + TypeScript + Vite**: 프런트엔드 개발 서버 및 프로덕션 빌드
- **MUI + AG Grid**: 검색, 카테고리/상태 필터, 서버 페이지네이션을 갖춘 상품 테이블
- **NestJS REST API**: 상품 조회 및 캐시 무효화 엔드포인트
- **Swagger UI**: API를 브라우저에서 확인하고 직접 호출
- **MongoDB 7**: 상품 문서 저장 및 조회용 인덱스
- **Redis 7**: 조회 결과 캐시, TTL 만료, 캐시 무효화
- **Docker Compose**: 프런트엔드, API, MongoDB, Redis를 함께 실행
- **Seed 스크립트**: 기본 설정으로 최대 1,000,000개 상품 데이터 생성

## 2. 아키텍처

```text
Browser
  |
  v
React + Vite / MUI / AG Grid
  |  HTTP /api/*
  v
Nginx (Docker web, port 8080)
  |  reverse proxy
  v
NestJS REST API (port 3000)
  |                         |
  | Redis GET/SET/SCAN      | MongoDB find/count
  v                         v
Redis (TTL cache)         MongoDB (products)
```

상품 목록 요청은 먼저 Redis를 확인합니다. 캐시가 있으면 MongoDB 조회를 생략하고, 캐시가 없으면 MongoDB에서 페이지 데이터와 전체 건수를 조회한 뒤 결과를 Redis에 저장합니다. Redis가 일시적으로 사용 불가능하면 캐시를 건너뛰고 MongoDB 결과를 반환하도록 구성되어 있습니다.

## 3. 빠른 시작: Docker Compose

필수 조건: Docker Desktop 또는 Docker Engine과 Docker Compose가 설치되어 있어야 합니다.

저장소를 내려받고 프로젝트 디렉터리로 이동합니다.

```bash
git clone https://github.com/jeong-jihyun/react-mongodb-redis-cache-demo.git
cd react-mongodb-redis-cache-demo
```

이미지 빌드와 컨테이너 실행:

```bash
docker compose up --build -d
```

기본 시드 스크립트는 1,000,000개 문서를 생성하도록 설정되어 있습니다. 초기 데이터가 필요하면 다음 명령을 실행합니다.

```bash
docker compose exec api npm run seed
```

데이터 생성은 디스크 속도와 CPU에 따라 시간이 걸릴 수 있습니다. 진행 상황은 명령 출력에서 확인할 수 있습니다. 이미 목표 수량 이상의 문서가 있으면 시드 작업은 건너뜁니다.

실행 상태와 로그 확인:

```bash
docker compose ps
docker compose logs -f api
docker compose logs -f web
```

### 접속 주소

| 서비스 | 주소 | 설명 |
|---|---|---|
| 웹 화면 | http://localhost:8080 | 상품 검색 및 캐시 성능 확인 |
| Swagger UI | http://localhost:3000/api/docs | API 명세 및 테스트 |
| 상품 목록 API | http://localhost:3000/api/products?page=1&limit=50 | JSON 응답 확인 |
| MongoDB | localhost:27017 | 상품 데이터베이스 |
| Redis | localhost:6379 | 캐시 저장소 |

## 4. 화면에서 확인할 내용

1. 상품 화면을 열고 첫 페이지를 조회합니다.
2. 동일한 검색어·필터·페이지 조건으로 다시 요청합니다.
3. 응답의 `cacheHit` 값이 첫 요청에서는 일반적으로 `false`, 캐시가 저장된 후 같은 요청에서는 `true`인지 확인합니다.
4. `durationMs`를 비교합니다. 캐시 HIT 요청은 대체로 더 빠르지만, 실제 시간은 실행 환경과 네트워크 상황에 따라 달라집니다.
5. 검색어, 카테고리, 상태 또는 페이지 번호를 변경해 다른 캐시 키가 사용되는지 확인합니다.
6. 캐시 삭제 API를 호출한 뒤 같은 조건을 다시 조회해 MISS가 발생하는지 확인합니다.

성능 비교는 같은 요청 조건으로 여러 번 실행하는 것이 좋습니다. 첫 실행은 데이터베이스/OS 캐시 상태, 인덱스, 컨테이너 초기화 등에 영향을 받을 수 있으므로 단일 측정값만으로 성능을 판단하지 마세요.

## 5. API 명세

### 상품 목록 조회

`GET /api/products`

| 쿼리 파라미터 | 필수 | 기본값 | 설명 |
|---|---:|---:|---|
| `page` | 아니요 | `1` | 1부터 시작하는 페이지 번호 |
| `limit` | 아니요 | `50` | 페이지당 문서 수, 최대 200 |
| `search` | 아니요 | 없음 | 상품명 또는 SKU 부분 검색 |
| `category` | 아니요 | 없음 | 카테고리 필터 |
| `status` | 아니요 | 없음 | `active` 또는 `inactive` 상태 필터 |

예시:

```bash
curl "http://localhost:3000/api/products?page=1&limit=50"
curl "http://localhost:3000/api/products?page=1&limit=20&search=Product%20100"
curl "http://localhost:3000/api/products?page=2&limit=25&category=Books&status=active"
```

응답에는 페이지 데이터와 전체 건수, 페이지 정보, 캐시 HIT 여부, API 처리 시간, 설정된 캐시 TTL이 포함됩니다. 전체 응답 스키마와 직접 실행 가능한 요청 예시는 Swagger UI에서 확인할 수 있습니다.

### 상품 캐시 무효화

`DELETE /api/products/cache`

```bash
curl -X DELETE "http://localhost:3000/api/products/cache"
```

이 API는 `products:v1:*` 패턴에 해당하는 Redis 캐시 키를 찾아 삭제하고 삭제된 키 수를 반환합니다. MongoDB 상품 데이터 자체를 삭제하거나 수정하지는 않습니다.

## 6. Cache-Aside 동작 방식

1. API가 검색어, 필터, 페이지 번호, 페이지 크기를 정규화해 캐시 키를 만듭니다.
2. Redis에서 해당 키를 조회합니다.
3. **HIT**: 캐시된 페이지 결과를 반환하고 MongoDB 조회를 생략합니다.
4. **MISS**: MongoDB에서 현재 페이지의 문서와 전체 건수를 조회합니다.
5. 조회 결과를 Redis에 저장하고 TTL이 지나면 자동 만료되도록 설정합니다.
6. Redis 연결이나 캐시 작업이 실패하면 캐시를 우회해 데이터베이스 조회를 진행합니다.
7. 캐시 무효화 API는 상품 조회 캐시 키를 삭제합니다.

기본 TTL은 60초이며 `CACHE_TTL_SECONDS` 환경 변수로 변경할 수 있습니다. Redis 캐시는 데이터베이스와 별도이므로, 상품 데이터가 다른 경로에서 변경될 때 캐시 일관성을 어떻게 유지할지도 함께 고려해야 합니다.

## 7. 로컬 개발 실행

Docker 대신 로컬에서 실행하려면 Node.js 20 이상과 MongoDB/Redis 인스턴스가 필요합니다. 기본 연결 주소는 로컬 MongoDB와 Redis입니다.

```bash
npm install
npm run dev
```

`npm run dev`는 NestJS API와 Vite 프런트엔드를 함께 실행합니다.

- Vite 개발 화면: http://localhost:5173
- Swagger UI: http://localhost:3000/api/docs

환경 변수 예시:

```bash
PORT=3000
MONGODB_URI=mongodb://localhost:27017/cache_demo
REDIS_URL=redis://localhost:6379
CACHE_TTL_SECONDS=60
SEED_COUNT=1000000
```

환경 변수는 실행 환경에 맞춰 설정하세요. Docker Compose를 사용할 때 API는 컨테이너 서비스 이름인 `mongodb`와 `redis`를 통해 각 서비스에 연결됩니다.

## 8. 데이터와 인덱스

Seed 스크립트는 상품명, SKU, 카테고리, 가격, 재고, 상태 등을 갖는 테스트 데이터를 생성합니다. 기본 목표량은 1,000,000개이며 `SEED_COUNT`로 변경할 수 있습니다.

초기화 과정에서는 상품명, 카테고리, 상태 기반 조회와 SKU 검색에 사용할 인덱스를 생성합니다. 대량 데이터 생성은 메모리, 디스크 용량 및 처리 시간에 영향을 주므로 운영 데이터베이스에서 무심코 실행하지 마세요. 개발 환경에서는 필요한 경우 `SEED_COUNT`를 작게 설정해 먼저 검증할 수 있습니다.

## 9. 종료 및 데이터 삭제

컨테이너만 중지하고 데이터 볼륨은 유지:

```bash
docker compose down
```

컨테이너와 함께 MongoDB/Redis 데이터 볼륨까지 삭제:

```bash
docker compose down -v
```

**주의:** `docker compose down -v`는 이 Compose 프로젝트의 MongoDB 및 Redis 영구 데이터를 삭제합니다. 보존해야 할 데이터가 없는지 확인한 뒤 실행하세요.

## 10. 문제 해결

- **웹 화면은 열리지만 API 요청이 실패함:** `docker compose ps`와 `docker compose logs -f api web`로 상태와 로그를 확인하세요.
- **MongoDB 또는 Redis 연결 오류:** 컨테이너가 healthy 상태인지 확인하고, 컨테이너 내부 연결 문자열이 `mongodb://mongodb:27017/cache_demo`, `redis://redis:6379`인지 확인하세요.
- **처음 실행했을 때 상품이 없음:** `docker compose exec api npm run seed`를 실행하세요.
- **캐시 HIT가 나오지 않음:** 동일한 검색어, 필터, 페이지, 페이지 크기로 재요청했는지 확인하세요. 파라미터가 달라지면 별도 캐시 키가 사용됩니다.
- **시드 작업이 오래 걸림:** 100만 건 생성은 환경에 따라 오래 걸릴 수 있습니다. 개발 시에는 `SEED_COUNT`를 줄여 검증하세요.

## 11. 범위와 운영 환경 적용 시 고려사항

이 프로젝트는 캐싱과 페이지네이션을 학습·시연하기 위한 예제입니다. 검색은 부분 문자열 정규식 기반이며, 데이터가 커지면 검색 방식과 인덱스 전략을 재검토해야 합니다. 페이지네이션은 `skip/limit`를 사용하므로 아주 깊은 페이지에서는 비용이 커질 수 있고, 대규모 서비스에서는 커서 기반 페이지네이션이 더 적합할 수 있습니다.

실제 운영 환경에 적용할 때는 인증·인가, 요청 제한, 비밀 정보 관리, 관측성, 장애 복구, 캐시 무효화 정책, 데이터 변경 시 일관성, 인덱스 및 쿼리 실행 계획, 컨테이너 자원 제한 등을 별도로 검토하세요. 이 저장소의 기본 설정은 프로덕션 배포에 대한 보안·성능 보증을 의미하지 않습니다.
