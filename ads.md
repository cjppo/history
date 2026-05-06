# ads-service — Design Document

> **Audience:** engineers (new joiners, on-call, integrators), partner teams (Shop, Risk, Privacy Vault, NIFT), product & SRE.
> **Status:** Living doc. Generated from source on 2026-05-06. Cross-reference with `knowledge-base/` for narrative-style notes.
> **Repo:** `github.com/squareup/ads-service` (CODEOWNERS: `@jiecaoap @zhiyong-yang1 @Minhaz-Abdullah-Afterpay`)

---

## Table of Contents

1. [Product Context](#1-product-context)
2. [Terminology / Glossary](#2-terminology--glossary)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Module Map](#4-module-map)
5. [Ad Decision Engine (Core Path)](#5-ad-decision-engine-core-path)
6. [Domain Model](#6-domain-model)
7. [API Design](#7-api-design)
8. [Data Layer & Persistence](#8-data-layer--persistence)
9. [Caching Strategy](#9-caching-strategy)
10. [Eventing (Kafka & SQS)](#10-eventing-kafka--sqs)
11. [Background Jobs / Workers](#11-background-jobs--workers)
12. [External Integrations](#12-external-integrations)
13. [Privacy, PII & Compliance](#13-privacy-pii--compliance)
14. [Observability](#14-observability)
15. [Resilience & Failure Modes](#15-resilience--failure-modes)
16. [Security & Access Control](#16-security--access-control)
17. [Configuration & Feature Flags](#17-configuration--feature-flags)
18. [Multi-Region Behavior](#18-multi-region-behavior)
19. [Build, Test, Deploy](#19-build-test-deploy)
20. [Corner Cases & Known Pitfalls](#20-corner-cases--known-pitfalls)
21. [Open Questions / Future Work](#21-open-questions--future-work)

---

## 1. Product Context

`ads-service` is the advertising backbone for **Afterpay** (Block). It exposes APIs for:

- **Consumers** (Afterpay app users) — receive personalized ads in shop feeds, mastheads, reward sections, and category tiles.
- **Advertisers / Merchants** — onboard, run campaigns, set budgets and targeting, monitor performance.
- **Internal partners** — Privacy Vault (deletion/export), NIFT (gift rewards eligibility), Jira (auto-campaign creation), Risk Underwriting (eligibility filters), Audience Service (segments).

**Primary problems solved**

| Problem | How it's solved |
|---|---|
| Real-time ad serving at scale | Cached campaign retrieval + filter→rank→select pipeline targeting sub-100ms |
| Personalization without violating consent | Subscription system + Privacy Vault export/forget + LaunchDarkly feature gates |
| Multi-region operation | Per-country config (US, AU, UK, NZ, CA) for risk filter, ranking algo, currency, locale, house-advertiser |
| Reliable accounting of impressions/clicks/actions/GMV | Redis hot path → 10-min flush to PostgreSQL with idempotent aggregation |
| Risk-aware ad delivery | Per-consumer disallowed-store lists from UDP risk service applied at filter stage |
| Cross-product loyalty (NIFT) | Independent worker pipeline that ingests payment files, computes eligibility, surfaces it as a targeting signal |

**SLO targets** (informal in current docs): sub-100ms ad delivery; 99.9% availability for the decision API.

---

## 2. Terminology / Glossary

| Term | Definition |
|---|---|
| **Advertiser** | Merchant entity that owns campaigns. Has a `storeId`, `merchantId`, region, and feature flags (`adsEnabled`, `affiliatesEnabled`, …). |
| **Campaign** | Time-bounded ad program for one advertiser. Has placements, targeting, creative, pricing model, budget caps, and a `Priority`. |
| **Ad** | A single creative unit attached to a campaign. Has an `AdType` and `Creative`. |
| **Placement** | A `(position, optional-slot)` pair that says where an ad will be shown. |
| **AdPosition** | A logical slot location, e.g. `SHOP_POSITION_1`, `SHOP_POSITION_MASTHEAD`, `REWARD_POSITION_1`, dynamic `CATEGORY_{id}`. |
| **AdType** | Visual format with width/height/weight: `MARQUEE_S/M/L`, `MASTHEAD`, `TILE`, plus their `_COMP` (component) variants. |
| **Slot** | An optional integer reserving a fixed rank within a position (used for "pinned" sponsorships). |
| **Creative** | Either `ImageCreative` (URL+w+h) or `ComponentCreative` (rich: headline/caption/terms/avatar/font color). |
| **Priority** | Campaign tier with a weight and selection algorithm: `SPONSORSHIP` (100, lottery), `PREMIUM` (200, auction), `STANDARD` (250, lottery), `HOUSE` (300, lottery). Lower weight = higher priority. |
| **Rate** | Pricing model: `CPC`, `CPM`, `CPA`, `FLAT`. |
| **CapType** | Budget cap dimension: `BUDGET`, `IMPRESSION`, `CLICK`. |
| **CampaignStatus** | `ACTIVE`, `INACTIVE` (with derived/displayed states `DRAFT`, `PAUSED`, `COMPLETED`, `SCHEDULED` exposed via API). |
| **Country** | One of `US`, `AU`, `UK`, `NZ`, `CA`. Drives currency, locale, risk-filter toggle, ranking-algo toggle. |
| **House advertiser** | Per-region fallback advertiser used to fill placements with internal/promotional ads. ID configured via `ads.region-to-house-advertiser-id`. |
| **Lottery** | Random-shuffle selection within a priority bucket. |
| **Auction (relevancy)** | `score = relevancy × eCPC`, ties broken randomly. Used by `PREMIUM` priority. |
| **NIFT** | Third-party gift-rewards platform. Consumers earn digital gifts after on-time payments. ads-service tracks eligibility and uses it as a targeting signal. |
| **UDP** | Internal recommendation/risk service that returns disallowed merchant lists per consumer. |
| **Audience Service** | Internal segmentation service used for `audience` targeting expressions. |
| **Privacy Vault** | Block's central PII store and data-rights orchestrator (forget/export). |
| **Aggregated Stat** | DB row holding `(totalImpressions, totalClicks, totalActions, totalGMV)` per campaign; updated from Redis every 10 min. |
| **Subscription** | Consumer-level opt-in/out for personalized ads: `SUBSCRIBED`, `UNSUBSCRIBED`, or `DEFAULT/UNDECIDED`. |

Models referenced above live under [`src/main/kotlin/com/afterpay/ads/model/`](../src/main/kotlin/com/afterpay/ads/model/).

---

## 3. High-Level Architecture

```
                 ┌──────────────────────────────────────────────────┐
                 │                Afterpay App                      │
                 │       (consumer, advertiser admin tools)         │
                 └────────────┬─────────────────────────────────────┘
                              │ HTTPS (REST, JSON)
                              ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                          ads-service                                 │
   │  Spring Boot 3.5.6 / Kotlin 1.9.25 / JVM 17                          │
   │                                                                      │
   │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐      │
   │  │ Controllers│  │  Services  │  │ Repositories│  │ External   │      │
   │  │  v1 / v2   │─▶│  (decision,│─▶│  Spring    │  │ Clients    │─────┐│
   │  │  /admin    │  │   admin,   │  │  Data JPA  │  │ (Feign)    │     ││
   │  │            │  │   nift)    │  │            │  │            │     ││
   │  └────────────┘  └────────────┘  └────────────┘  └────────────┘     ││
   │        ▲                ▲              ▲                ▲           ││
   │        │                │              │                │           ││
   │  ┌─────┴──────────────────────────────────────────────────────────┐ ││
   │  │  Cross-cutting: AOP logging, MDC filter, exception advice,     │ ││
   │  │  PII decryption, observability metrics, user-context          │ ││
   │  └──────────────────────────────────────────────────────────────┘  ││
   │                                                                     ││
   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐           ││
   │  │ Kafka        │  │ SQS          │  │ Scheduled Jobs   │           ││
   │  │ Consumers    │  │ Listeners    │  │ (cache, NIFT,    │           ││
   │  │ (commerce/   │  │ (PV forget/  │  │  Jira, stats)    │           ││
   │  │  order/      │  │  export)     │  │                  │           ││
   │  │  profile)    │  │              │  │                  │           ││
   │  └──────────────┘  └──────────────┘  └──────────────────┘           ││
   └──────────────────────────────────────────────────────────────────────┘│
              │              │             │                  │           │
              ▼              ▼             ▼                  ▼           ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────┐
        │ Postgres │  │ Redis    │  │ Caffeine │  │ Kafka        │  │ AWS        │
        │  (RDS)   │  │ (Elasti- │  │ (in-proc)│  │ (commerce,   │  │ S3/SQS/SNS │
        │ schema   │  │  Cache)  │  │          │  │  ads, advert)│  │ Secrets    │
        │  "ads"   │  │          │  │          │  │              │  │            │
        └──────────┘  └──────────┘  └──────────┘  └──────────────┘  └────────────┘

External downstream callouts (Feign + circuit breakers):
  Shop Service, Shop Product Service, UDP, Risk Underwriting, Privacy Vault,
  Audience Service, Data Service, NIFT (US/GB), Airtable, Jira, Affiliate API
```

**Style:** classic 3-tier (Controller → Service → Repository) layered service, with an event-driven sidecar (Kafka/SQS) and a worker pipeline for offline processing.

**Key runtime topology**
- One Spring Boot deployable, JIB-built container (`base/jre-17-wolfi`), running on Kubernetes (us-east-1 omega + beta envs configured under `infra/k8s/app/envs/`).
- Postgres RDS schema `ads`. Migrations under `src/main/resources/db/migration/V*.sql` (V1–V21 currently).
- Redis used for distributed cache, distributed locks (Redisson-style locks for jobs), heartbeats, and hot stats.
- AWS Secrets Manager auto-config is **explicitly excluded** from Spring Boot to avoid auto-pull behavior; secrets are wired in by env.

---

## 4. Module Map

Source root: [`src/main/kotlin/com/afterpay/ads/`](../src/main/kotlin/com/afterpay/ads/)

| Package | Purpose |
|---|---|
| `AdsServiceApplication.kt` | Boot main. `@EnableScheduling` + `@EnableAspectJAutoProxy`. Excludes `SecretsManagerAutoConfiguration`. |
| `configuration/` | All `@Configuration` beans: Kafka, SQS/SNS/S3 clients, Redis, external Feign clients, pagination resolver, PII decryption, web config, categories cache startup gate. |
| `controller/` | REST endpoints. Subpackages `controller/admin/` (advertiser CRUD), `controller/v2/` (campaign CRUD + admin v2). |
| `service/` | Business logic. Notable subpackages below. |
| `service/ads/decision/` | The decision pipeline: retrieval, filter chain, ranking strategies, selector, decision context. |
| `service/ads/admin/` | Campaign create/update/list, validation. |
| `service/ads/helper/` | `CustomerRetriever`, `CampaignCacheHelper`, `AggregatedStatsCacheHelper`, `ClickEventCacheHelper`, `CampaignBuilder`. |
| `service/advertiser/` | `AdvertiserService` — CRUD, sync, refresh, list. |
| `service/consumer/` | Subscription mgmt, Privacy Vault data export/deletion logic. |
| `service/nift/` | `NiftService`, `PaymentDataProcessingService`, `NiftSQSHelper`, payment/eligible-user file models. |
| `service/handler/` | LaunchDarkly feature flag handlers (`AdvertiserFeatureFlagHandler`, `AffiliateFlagHandler`, `PromoFlagHandler`). |
| `service/aws/` | `AWSS3Service`, `AWSSQSService` thin wrappers. |
| `service/data/` | `DataService` — calls internal Data Service. |
| `service/model/` | `ModelService` — calls ML model service for relevancy/inference. |
| `data/model/` | JPA entities (advertiser, consumer subs, campaigns, ads, aggregated stats, event log, consent, jira mapping). |
| `data/repository/` | Spring Data JPA repos + native query builders for campaigns. |
| `data/converter/` | JPA converters (e.g. `SubscriptionStatusConverter`). |
| `model/` | Pure domain models (`Country`, `Priority`, `AdType`, `AdPosition`, `Placement`, `Targeting`, `Campaign`, `Ad`, `Customer`, creatives). |
| `external/` | Feign clients & DTOs grouped by external system: `nift/`, `privacyvault/`, `udp/`, `shop/`, `shopproductservice/`, `airtable/`, `jira/`, `affiliate/`, `audience/`, `risk/`, `data/`, `model/`. |
| `consumer/` | Kafka `EventConsumer` + SQS listeners (`PrivacyVaultDeletionListener`, `PrivacyVaultExportListener`). |
| `event/` | Kafka publishers `AdsEventPublisher`, `AdvertiserEventPublisher`. |
| `job/` | Scheduled jobs: aggregated stats, campaign cache refresh, NIFT payment worker, Jira creator. |
| `common/` | AOP `LogAspect`, `MDCFilter`, exception types + `GlobalExceptionHandler`, fluent logging (`KFluentLogger`, `StructuredLogEntry`), `ObservabilityMetrics`. |
| `helper/` | Generic util helpers. |
| `docs/` | OpenAPI annotations / supporting types for swagger generation. |

---

## 5. Ad Decision Engine (Core Path)

**Entry:** `POST /ads/v1/consumers/{uuid}/ads` ([`controller/AdsDecisionController.kt`](../src/main/kotlin/com/afterpay/ads/controller/AdsDecisionController.kt)) → orchestrated by [`service/ads/decision/AdsDecisionService.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/AdsDecisionService.kt).

```
 1. Request           2. Context          3. Retrieve         4. Filter           5. Rank             6. Select & Return
 ─────────            ─────────           ──────────          ────────            ──────              ─────────────────
  consumer UUID       Disallowed stores   Active campaigns    Targeting filter    Group by AdType     Pin slot if any
  placements          Relevancy scores    matching country/   (device + audience) Group by Priority   Promote hero (M/L)
  ad types            NIFT eligibility    position/types      Risk filter         Lottery / Auction   Top N per placement
  flags (explainer,   (Redis-cached)      (cache or DB)       (UDP + NIFT)        per priority bucket Group by position
   risk filter)
```

**Step 1 — Request reception**

- Body contains placement names + ad types.
- Special placements: `CATEGORIES` expands into dynamic `CATEGORY_{id}` positions for shop tiles.
- Optional flags toggle explainer mode and override the risk filter (`AdsDecisionContext` ThreadLocal).

**Step 2 — Customer context** ([`service/ads/helper/CustomerRetriever.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/helper/CustomerRetriever.kt))

Builds a `Customer` profile:
- **Disallowed store IDs** — fetched from UDP, cached in Redis. If UDP times out and risk filter is required, behavior is per-country (see §15).
- **Per-merchant relevancy scores** — from `DataService`, cached in Redis.
- **NIFT eligibility list** — from `NiftService`, cached in Redis.

All three are loaded in parallel and tolerate partial failure (filters degrade gracefully — see §20).

**Step 3 — Campaign retrieval** ([`AdsRetrievalService.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/AdsRetrievalService.kt))

- Queries are served primarily from Redis (`CampaignCacheHelper`), backed by the `RefreshCampaignsCacheJob` (every 30 min).
- Filters to `ACTIVE` campaigns where `now ∈ [startDate, endDate]`, and that match `(country, position, adTypes)`.
- Pinned-slot campaigns retained even if their priority bucket is otherwise empty.

**Step 4 — Filtering**

Chain pattern. Two main filters today:

| Filter | Purpose | Source data | Notes |
|---|---|---|---|
| `AdsTargetingFilter` | Match `Targeting.deviceType` (iOS/Android) and audience expression via Audience Service | request headers + Audience Service | Audience eval is feature-flagged per region. |
| `AdsRiskStoreFilter` | Drop campaigns whose `storeId` is on consumer's disallowed list; check NIFT eligibility for NIFT campaigns | UDP, NIFT cache | Only enabled when `Country.riskFilterEnabled == true` (US, AU, UK; not NZ/CA). |

**Step 5 — Ranking**

Implemented as the `AdsRankingService` orchestrating two strategies:

- `LotteryRanking` — random shuffle. Every campaign in the bucket has equal probability.
- `RelevancyAuctionRanking` — `score = relevancyScore × eCPC` (defaults: `relevancy=500`, `eCPC=0.01`). Tie-break by random.

Order of processing within a placement:
1. **By AdType weight (asc)** — larger ads (lower weight) checked first.
2. **By Priority weight (asc)** — `SPONSORSHIP (100) → PREMIUM (200) → STANDARD (250) → HOUSE (300)`.
3. **Apply algo** for that priority's bucket.

**Step 6 — Selection** ([`AdsSelector.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/AdsSelector.kt))

- Resolve **pinned slots** (campaigns with `Placement.slot != null`) into reserved positions.
- Promote **hero ads** (`MARQUEE_M` or `MARQUEE_L`) to the front when at least one is present in a position's candidate set.
- Take **top N** (default 5) per placement.
- Format response: list of decisions per position, each with `campaignId`, `storeId`, `clickUrl`, `adType`, `slot`, and creative payload.

---

## 6. Domain Model

### 6.1 Class diagram (essential entities)

```
┌──────────────┐ 1     n ┌──────────────┐ 1     n ┌──────────────┐
│  Advertiser  │────────▶│   Campaign   │────────▶│      Ad      │
│ (id, name,   │         │ (id, name,   │         │ (id, type,   │
│  storeId,    │         │  priority,   │         │  clickUrl,   │
│  status,     │         │  rate,       │         │  creative)   │
│  features)   │         │  capType,    │         └──────────────┘
└──────────────┘         │  capValue,   │                │
        │                │  status,     │                ▼
        │                │  country,    │      ┌──────────────────┐
        │                │  startDate,  │      │ Creative (sealed)│
        │                │  endDate,    │      │  ImageCreative   │
        │                │  targeting,  │      │  ComponentCreative│
        │                │  placements) │      │  TemplateCreative│
        │                └──────────────┘      └──────────────────┘
        │                       │
        │                       ▼
        │                ┌──────────────┐
        │                │  Placement   │ (n)
        │                │ (position,   │
        │                │  slot?)      │
        │                └──────────────┘
        ▼
┌──────────────────────┐         ┌────────────────────────┐
│ AggregatedStat       │         │ AdsConsumerSubscription│
│ (campaignId,         │         │ Status                 │
│  totalImpressions,   │         │ (consumerUuid, status, │
│  totalClicks,        │         │  source, …)            │
│  totalActions,       │         └────────────────────────┘
│  totalGMV)           │         ┌────────────────────────┐
└──────────────────────┘         │ AdsConsumerLatestInst- │
                                 │ allment                │
                                 │ (consumerUuid, latest- │
                                 │  paymentDateTime, …)   │
                                 └────────────────────────┘
```

### 6.2 Enums (canonical values)

| Enum | Values |
|---|---|
| `Country` | `US`, `AU`, `UK`, `NZ`, `CA` (each with `locale`, `currency`, `riskFilterEnabled`, `adsRankingEnabled`, MCC code) |
| `Priority` | `SPONSORSHIP(weight=100, alg=LOTTERY)`, `PREMIUM(200, AUCTION)`, `STANDARD(250, LOTTERY)`, `HOUSE(300, LOTTERY)` |
| `Algorithm` | `LOTTERY`, `AUCTION`, `SLOT` |
| `AdType` | `MARQUEE_S` (343×148), `MARQUEE_M` (343×200), `MARQUEE_L` (343×250), `MASTHEAD` (750×750), `TILE` (1×1), and `_COMP` variants |
| `Rate` | `CPC`, `CPM`, `CPA`, `FLAT` |
| `CapType` | `BUDGET`, `IMPRESSION`, `CLICK` |
| `CampaignStatus` (DB) | `ACTIVE`, `INACTIVE`. (API surface adds derived `DRAFT`, `PAUSED`, `COMPLETED`, `SCHEDULED`.) |
| `AdvertiserStatus` | `ACTIVE`, `INACTIVE` |
| `Publisher` | `AP` (Afterpay), `SHOP` |
| `CustomerSubscriptionStatus` | `SUBSCRIBED`, `UNSUBSCRIBED`, `DEFAULT` |
| `SubscriptionSource` | `USER`, `SYSTEM`, `PRIVACY_VAULT` |
| `EventType` (Kafka audit) | `CREATED`, `UPDATED`, `DELETED` |
| `ServerRegion` | `US`, `GB`, `AU`, `NZ`, `CA` |

---

## 7. API Design

**Conventions**
- Base paths `/ads/v1/...` and `/ads/v2/...` (URL-versioned). v2 is the current target for **campaign management**; v1 is current for **decision** and **subscription** APIs.
- JSON request/response. Multipart upload for campaign creative.
- OpenAPI/Swagger UI at `/swagger-ui.html`, raw spec at `/v3/api-docs`.
- Pagination via `PaginationArgumentResolver` → `(offset, limit)` query params, `Page<T>` response (`items`, `total`, `pageInfo`).
- Standard error response shape from `GlobalExceptionHandler`:
  ```json
  { "error": { "code": "ERROR_CODE", "message": "...", "details": {...} } }
  ```

### 7.1 Consumer-facing endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/ads/v1/consumers/{uuid}/ads` | **Primary decision API** — get personalized ads for placements. Supports `explainer` + `riskFilter` flags. |
| `GET`  | `/ads/v1/consumers/{uuid}/relevancyscore` | Debug/admin: relevancy scores for that consumer. |
| `GET`  | `/ads/v1/consumers/{consumerId}/subscriptions?includeSource` | Get personalized-ad opt-in status. |
| `POST` | `/ads/v1/consumers/{consumerId}/subscriptions` | Update opt-in/out. |
| `GET`  | `/ads/v1/subscriptions/consumerIds` | Paginated list of consumerIds matching a `SubscriptionFilter`. |
| `GET`  | `/ads/v1/consumers/{consumerId}/nifteligibility?email&region` | Check NIFT gift eligibility. |
| `GET`  | `/ads/v1/consumers/{consumerId}/eligibility?country&audienceTargeting` | Check audience eligibility. |
| `GET`  | `/ads/v1/advertiser?storeId&publisher` | Look up an active advertiser by store. |
| `GET`  | `/ads/v1/advertisers/{advertiserId}` | Get advertiser by id (with shop attributes). |

### 7.2 Admin / advertiser endpoints (v1)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/ads/v1/admin/advertiser` | Create advertiser. |
| `PUT`  | `/ads/v1/admin/advertiser/{advertiserId}` | Update advertiser. |
| `GET`  | `/ads/v1/admin/advertisers` | List/search (filters: `q`, `countryCodes`, `locale`, `storeIds`, `publisher`, `status`, `adsEnabled`, paging). |
| `GET`  | `/ads/v1/admin/advertiser/{advertiserId}` | Get one. |
| `GET`  | `/ads/v1/admin/advertisers/bulk?ids=...` | Bulk get by IDs. |
| `POST` | `/ads/v1/admin/sync/advertisers/all` | (Deprecated) Sync all from upstream. |
| `POST` | `/ads/v1/admin/sync/advertisers/{advertiserId}` | (Deprecated) Sync single. |
| `POST` | `/ads/v1/admin/advertiser/refresh` | Refresh advertiser cache (by advertiserId or storeId). |

### 7.3 Admin / campaign endpoints (v2)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/ads/v2/admin/advertisers/{advertiserId}/campaigns` | Create campaign (multipart — creative upload). |
| `PUT`  | `/ads/v2/admin/advertisers/{advertiserId}/campaigns/{campaignId}` | Update campaign. |
| `POST` | `/ads/v2/admin/campaigns/{campaignId}/campaign_status` | Change status (ACTIVE/INACTIVE). |
| `GET`  | `/ads/v2/admin/campaigns` | List/search campaigns (filters: `advertiserId`, `country`, `status`, `position`, `adType`, `slot`, `priority`, `startDate`, `endDate`, `q`, paging). |
| `GET`  | `/ads/v2/admin/campaigns/{campaignId}` | Get one. |
| `PUT`  | `/ads/v2/admin/campaigns/cache/refresh?countries=US,AU` | Force cache rebuild for given countries. |
| `GET`  | `/ads/v2/admin/tile/categories?locale=...` | Get available tile categories for a locale. |
| `POST` | `/ads/v2/admin/consumers/{consumerId}/subscriptions` | Admin-side subscription update with explicit source. |

### 7.4 Operational / system endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/readiness` | 503 until categories cache is populated; 200 otherwise. |
| `GET` | `/liveness` | Always 200 "ok". |
| `GET` | `/ads/v1/privacy-vault/status` | PV status (only when SQS listener enabled). |
| `GET` | `/ads/v1/backfill/payment` | Replay a payment file from S3 → SQS. |
| `GET` | `/ads/v1/backfill/nift_eligibility_user?paymentDate&region` | Re-run eligibility for a specific date+region. |
| `GET` | `/ads/v1/backfill/decrypt?ciphertext=...` | Decrypt a base64 PII ciphertext (admin-only debug). |
| `GET` | `/ads/v1/backfill/jira` | Force a Jira poll. |

### 7.5 Example request/response

**Get ads — request**

```http
POST /ads/v1/consumers/c5b8e9a4-…/ads
Content-Type: application/json

{
  "country": "US",
  "locale": "en-US",
  "deviceType": "IOS",
  "placements": [
    { "name": "SHOP_POSITION_1", "adTypes": ["MARQUEE_M", "MARQUEE_L"] },
    { "name": "SHOP_POSITION_MASTHEAD", "adTypes": ["MASTHEAD"] },
    { "name": "CATEGORIES", "adTypes": ["TILE"] }
  ]
}
```

**Response**

```json
{
  "decisions": {
    "SHOP_POSITION_1": [
      {
        "campaignId": 4711,
        "storeId": 980,
        "clickUrl": "https://afterpay.com/click?...",
        "adType": "MARQUEE_M",
        "slot": null,
        "data": { "imageUrl": "https://cdn/.../hero.png", "width": 343, "height": 200 }
      }
    ],
    "SHOP_POSITION_MASTHEAD": [ /* … */ ],
    "CATEGORY_42": [ /* … */ ]
  }
}
```

---

## 8. Data Layer & Persistence

**Database:** PostgreSQL (RDS), schema `ads`. Hibernate dialect, `open-in-view: false`, HikariCP with `maxPoolSize=20`.

**Migration tool:** Flyway-style versioned SQL files at [`src/main/resources/db/migration/V*.sql`](../src/main/resources/db/migration/) (V1–V21 currently).

### 8.1 Key tables (entities)

| Table / Entity | Purpose | Key fields |
|---|---|---|
| `advertiser` (`Advertiser.kt`) | Merchant catalog | `id`, `name`, `storeId`, `merchantId`, `remoteAdvertiserId`, `status`, `countryCode`, `currency`, `logoUrl`, `publisher`, `category`, `adsEnabled`, `affiliatesEnabled`, `promotionsEnabled`, `storefrontEnabled`, audit |
| `ads_campaign` (`CampaignEntity.kt`) | Campaigns | `id`, `advertiserId`, `name`, `priority`, `placements (JSON)`, `targeting (JSON)`, `status`, `startDate`, `endDate`, `country`, `rate`, `price`, `capType`, `capValue`, audit. Indexed (V13). |
| `ad` (`AdEntity.kt`) | Creatives keyed by campaign | `id`, `campaignId`, `adType`, `clickUrl`, `creative (JSON)` |
| `aggregated_stats` (`AggregatedStat.kt`) | Per-campaign accumulators | `campaignId`, `totalImpressions`, `totalClicks`, `totalActions`, `totalGMV` (added V18) |
| `ads_consumer_subscription_status` | Personalized-ad opt-in | `consumerUuid (PK)`, `status`, `source` (V21), audit |
| `ads_consumer_latest_installment` (V10, V20) | NIFT eligibility input | `consumerUuid`, `installmentId`, `latestPaymentDateTime`, region columns |
| `event_log` | Audit of Kafka events emitted | `topicName`, `partitionKey`, `eventType`, `payload`, `createdDateTime` |
| `consent` | Consumer consent records | per-purpose flags |
| `jira_campaign_mapping` | Jira issue ↔ ads-campaign id | mapping table for deduplication |
| `advertiser_affiliate_network_category_mapping` (V19) | Affiliate config per advertiser | for affiliate program routing |

### 8.2 Repository conventions

- Spring Data JPA interfaces under [`data/repository/`](../src/main/kotlin/com/afterpay/ads/data/repository/).
- Campaign repository has additional native query helpers:
  - `CampaignNativeRepository` + `CampaignNativeQueryBuilder` for filterable list endpoints.
  - `CampaignDbReader` / `CampaignDbWriter` to encapsulate complex multi-step transactions.
- JSON columns (placements, targeting, creative) use `hibernate-types-60` mappers.
- Custom JPA converters live in `data/converter/` (e.g. enum string converters).

---

## 9. Caching Strategy

```
                ┌──────────────────────────────────────────────┐
                │                Application                   │
                │                                              │
                │  ┌──────────┐    ┌──────────┐    ┌────────┐  │
                │  │ Caffeine │    │ Local    │    │ Locks  │  │
   Hot reads → │  │ (advert. │    │ caches   │    │ (Redis │  │
                │  │  stats…) │    │          │    │  TTL)  │  │
                │  └────┬─────┘    └────┬─────┘    └────┬───┘  │
                └───────┼───────────────┼───────────────┼──────┘
                        ▼               ▼               ▼
                ┌──────────────────────────────────────────────┐
                │                   Redis                      │
                │  campaign cache (per country) │ stats hot    │
                │  customer-ctx cache (UDP/Data/NIFT)          │
                │  job locks + heartbeats │ readiness flags    │
                └──────┬───────────────────────────────────────┘
                       ▼
                ┌──────────────────────────────────────────────┐
                │                 Postgres                     │
                └──────────────────────────────────────────────┘
```

| Cache | Location | Populated by | TTL / refresh |
|---|---|---|---|
| Live + upcoming campaigns (per country) | Redis (key per country) | `RefreshCampaignsCacheJob` | every 30 min, plus admin-triggered `PUT /ads/v2/admin/campaigns/cache/refresh` |
| Active advertisers | Caffeine in-process via `AdvertiserLocalCache` | Lazy + bulk `getAdvertisersById()` per stats refresh | size/time-eviction |
| Customer disallowed stores (UDP) | Redis | `CustomerRetriever` fetch path | configurable TTL |
| Customer relevancy | Redis | `CustomerRetriever` | configurable TTL |
| NIFT eligibility | Redis | `NiftService` | configurable TTL |
| Aggregated stats hot path | Redis | event consumers + `AggregatedStatsCacheHelper` | flushed every 10 min by `AggregatedStatsCacheJob` |
| Click events | Redis | `ClickEventCacheHelper` | rolling buffer |
| Categories | Caffeine | `CategoriesCacheStartupConfig` (blocks `/readiness`) | refreshed at startup; subsequent updates via Shop service |

**Coherency rule:** writes go to Postgres first, then enqueue cache invalidation (admin endpoints exist for explicit refresh). Stats are an exception — they accumulate in Redis and are reconciled into Postgres on a 10-min cadence (eventually consistent).

---

## 10. Eventing (Kafka & SQS)

### 10.1 Kafka — inbound (consumers)

Configured in [`consumer/EventConsumer.kt`](../src/main/kotlin/com/afterpay/ads/consumer/EventConsumer.kt).

| Topic | Handler | Purpose |
|---|---|---|
| `event.commerce-event-tracking.v1` | `CommerceEventHandler` | Commerce events (orders / clicks / impressions / actions) — drives counters in Redis. |
| `event.paylater.order.v1` | `OrderEventHandler` | Order lifecycle events (used for GMV + CPA accounting). |
| `event.paylater.consumer.profile.v1` | `ProfileChangeEventHandler` | Consumer profile changes (subscription/PII updates). |

Consumer-group: `market-tech-event-kafka-consumer`. Avro deserialization via Confluent Schema Registry. `auto-offset-reset: earliest`. `max-poll-records: 50`, `pollDuration: 500ms`.

### 10.2 Kafka — outbound (publishers)

| Publisher | Topic env var | Payload (Avro schema) |
|---|---|---|
| `AdsEventPublisher` | `ADS_CAMPAIGN_TOPIC` | `AdsCampaign` lifecycle (CREATED/UPDATED/DELETED), partition key = `campaignId`. Also persisted to `event_log` table. |
| `AdvertiserEventPublisher` | `ADS_ADVERTISER_TOPIC` | `Advertiser` lifecycle, partition key = `advertiserId`. Also persisted to `event_log`. |

Schemas come from Confluent + the `com.touchcorp.paylater.event:schema-*` libs (versions in `gradle.properties`).

### 10.3 SQS — inbound listeners

Privacy Vault listeners (only enabled when `cloud.aws.sqs.listener.auto-startup=true`):

| Listener | Queues | Behavior |
|---|---|---|
| `PrivacyVaultDeletionListener` | `PRIVACY_VAULT_FORGET_ENTITY_{NA,EU,ANZ}_QUEUE_NAME` | Receives `ForgetEntityRequest` proto, deletes consumer PII from local DB, sends audit log to PV. |
| `PrivacyVaultExportListener`   | `PRIVACY_VAULT_EXPORT_FOR_ENTITY_{NA,EU,ANZ}_QUEUE_NAME` | Receives `ExportForEntityRequest`, exports `AdsExportData` proto (subscription, latest payment, region) back to PV. |

Consumer worker SQS:

| Job → Queue | Direction | Purpose |
|---|---|---|
| `PaymentEventProcessWorker` ← `martech-payment-event-queue-beta[-gb]` | inbound | Triggered by S3 `Object Created` events; drives NIFT payment ingestion. |
| `NiftSQSHelper` → eligible-users SQS | outbound | Notifies downstream of generated CSVs. |

---

## 11. Background Jobs / Workers

All jobs live under [`src/main/kotlin/com/afterpay/ads/job/`](../src/main/kotlin/com/afterpay/ads/job/) and are activated by `@EnableScheduling` on `AdsServiceApplication`.

| Job | Schedule | What it does | Concurrency control |
|---|---|---|---|
| `AggregatedStatsCacheJob` | `0 */10 * * * *` (every 10 min UTC) | Read incremental counts (`impressions`, `clicks`, `actions`, `GMV`) from Redis → persist into `aggregated_stats` → emit DataDog `ap.ads.campaign.{impressions,clicks}` tagged with `advertiser_id` + normalized `advertiser_name`. | None needed (idempotent diff, single emitter pattern) |
| `RefreshCampaignsCacheJob` | `0 0/30 * * * *` (every 30 min) | Rebuild Redis campaign cache for each enabled country in parallel; supports manual refresh via API. | Per-country distributed Redis lock |
| `PaymentEventProcessWorker.updateLatestPaymentDate` | `fixedDelay=1h, initialDelay=0` | Pull S3 event notifications from US + GB SQS queues; download payment CSV; upsert `AdsConsumerLatestInstallment`; record latest file location in Redis. | Idempotent at row level; SQS visibility timeout used |
| `PaymentEventProcessWorker.findEligibleUser` (per region) | Cron `paymentProcessor.cron-eligible-users-calculation-{us,gb}` (default `0 0 10 * * *`) | Process latest payment file → call NIFT API in parallel (10-thread pool) → write CSVs (≤ 50k rows × ≤ 100 files) to S3. | Redis distributed lock per region (5-min TTL) + heartbeat (10s interval, 60s stale threshold) + daily completion key |
| `JiraAdsCampaignCreationJob` | every 1 min | Poll Jira for `AD_CAMPAIGN` issues; auto-create advertisers/campaigns/ads (carousel + tile flows); label issues with errors. | `JiraCampaignMapping` dedupes |

**Distributed locking pattern (NIFT eligibility job)**

1. Check `nift-eligible-latest-success-date_{region}` — if today, skip.
2. `SET NX EX 300` on `nift-eligible-user-job-lock_{region}` — fail = another instance already running.
3. Read `nift-eligible-user-job-heartbeat-{date}-{region}` — if fresh (<60s), assume live worker; bail.
4. Start heartbeat thread updating every 10s.
5. Run pipeline. Always cleanup heartbeat in `finally`. Mark success only on clean exit.

---

## 12. External Integrations

All Feign clients live under [`src/main/kotlin/com/afterpay/ads/external/`](../src/main/kotlin/com/afterpay/ads/external/). Wiring is in `configuration/ExternalClientConfig.kt`. OkHttp transport. Resilience4j circuit breakers wrap calls (default config: 50% failure rate, 50% slow-call rate at 3s, 30s open state — see `application.yml`).

| Integration | Client | Purpose | CB instance(s) | AWS auth? |
|---|---|---|---|---|
| Shop Service | `ShopServiceClient` | Store metadata, store categories | `shopServiceClientGetCategoriesCircuitBreaker` | – |
| Shop Product Service | `ShopProductServiceAPIClient` | Product↔collection assignments (promotions) | – | – |
| UDP (recs/risk) | `UdpClient` (+ `UdpClientFacade`) | Per-consumer risk-merchant filter | – | basic auth (per region creds) |
| Risk Underwriting | `RiskUnderwritingClient` (+ facade) | Gift-card/NIFT eligibility risk check | – | – |
| Audience Service | `AudienceServiceClient` (+ factory) | Evaluate audience expressions | – | per-region client |
| Data Service | `DataClient` | Generic consumer-data lookups (relevancy) | – | – |
| NIFT (US/GB) | `NiftClient` | OAuth2 client_credentials → gift status query | `niftClientAuthCircuitBreaker`, `niftClientGiftStatusCircuitBreaker` | OAuth2 |
| Privacy Vault | `PrivacyVaultApiClient` | Audit-log + status (RPC over HTTP) | – | (internal mTLS / signed) |
| Airtable | `AirtableClient` | Sync data for ops/reporting | – | API key |
| Jira | `JiraClient` | Search/update/comment on AD_CAMPAIGN issues | – | basic / token |
| Affiliate API | `AffiliateAPIClient` | Disable affiliate per advertiser | – | – |
| Model Service | `ModelService` | ML inference (relevancy) | – | – |
| AWS S3 | SDK v2 | NIFT files, exports | – | IAM role (IRSA) |
| AWS SQS | SDK v2 + Spring Cloud AWS | Privacy Vault + NIFT queues | – | IAM role |
| AWS SNS | SDK v2 | Cross-service notifications | – | IAM role |
| AWS Secrets Manager | SDK v2 | Pull credentials when needed (auto-config disabled) | – | IAM role |
| Confluent Schema Registry | – | Avro schema lookup for Kafka | – | Basic / mTLS |
| Redis | Spring Data Redis | Cache, locks, heartbeats | – | – |
| LaunchDarkly | LD SDK 5.9.0 | Feature flags (per-advertiser, per-region) | – | SDK key |
| DataDog | DogStatsD client 3.0.1 | Metrics emission | – | UDP statsd |

---

## 13. Privacy, PII & Compliance

**Goals:** GDPR-compliant export & deletion; encrypt sensitive fields at rest; auditable Kafka event log.

### 13.1 PII handling

- **Encryption library:** `com.squareup.cash.incentivedatalib:encryptor` configured by `PiiDecryptionConfiguration`.
- **KMS key:** `aws-kms://arn:aws:kms:us-east-1:.../afterpay-ads-pii-udf` (per `cash.data.key` config).
- **PII-protection on event payloads:** `com.touchcorp.paylater.event:pii-protection:4.3.0` wraps Kafka events.

### 13.2 Privacy Vault flow

```
PV ─[ForgetEntityRequest]─▶ SQS (NA/EU/ANZ) ─▶ PrivacyVaultDeletionListener
                                                          │
                                                          ▼
                                              Delete from ads tables
                                              (subscriptions, payments, …)
                                                          │
                                                          ▼
                                          PrivacyVaultApiClient.sendAuditLog()


PV ─[ExportForEntityRequest]─▶ SQS (NA/EU/ANZ) ─▶ PrivacyVaultExportListener
                                                          │
                                                          ▼
                                                  Build AdsExportData proto
                                                  (subscription, latest payment, region)
                                                          │
                                                          ▼
                                                  Send back to PV
```

`AdsExportData` is defined at [`proto/afterpay/ads/ads_export_data.proto`](../proto/afterpay/ads/ads_export_data.proto). The shape is governed by Square's central `squareup/privacyvault/*.proto` schema and `governance/v0/*` annotations (consumer/merchant/payment-card data classifications).

### 13.3 Subscription & opt-out

- Default for new consumers is `DEFAULT/UNDECIDED` (not `SUBSCRIBED`).
- `SubscriptionSource.PRIVACY_VAULT` records changes that originated from a PV-driven action — distinguishable from explicit user actions for audit.

---

## 14. Observability

### 14.1 Metrics (DataDog via DogStatsD)

| Metric | Tags | Source |
|---|---|---|
| `ap.ads.campaign.impressions` (count) | `advertiser_id`, `advertiser_name` (normalized) | `AggregatedStatsCacheJob` |
| `ap.ads.campaign.clicks` (count) | `advertiser_id`, `advertiser_name` (normalized) | `AggregatedStatsCacheJob` |
| (job/health) NIFT job success/failure counters | region | `PaymentEventProcessWorker` |
| (decision) per-stage timings (logged) | – | `AdsDecisionService` |

`advertiser_name` normalization: lowercase → trim → non-alphanum→`_` → collapse `__+` → trim `_` → cap 100 chars → fallback `"unknown"`.

### 14.2 Logging

- `logback-spring.xml` + Logstash JSON encoder.
- `KFluentLogger` provides a Kotlin DSL: `logger.info("...").field("k", v).log()`.
- `MDCFilter` sets request correlation ID on every inbound request.
- `LogAspect` (AOP) auto-logs method entry/exit + timing.
- Notable structured event names: `nift_gift_status`, `nift_auth_failure`, `nift-payment-data-job`, `nift-eligibility-job`, `decision_explainer`.

### 14.3 Health

- `/liveness` — always 200.
- `/readiness` — 503 until Categories cache populated (gate by `CategoriesCacheStartupConfig`).
- DB health implicitly via Hikari pool metrics + Spring Boot actuator (when enabled).

---

## 15. Resilience & Failure Modes

| Failure | Behavior | Mitigation |
|---|---|---|
| External service down (NIFT, Shop, UDP) | Resilience4j circuit breaker opens after 50% failure or 50% slow-call rate (≥10 calls in window of 20). Half-open after 30s. | Filter falls back: empty disallowed list → no risk filtering for that consumer that request; missing relevancy → default `500`; missing NIFT → exclude NIFT campaigns. |
| Postgres temporarily unavailable | Hikari connection-timeout 30s; requests fail. | Decision API is cache-first, but admin/write endpoints surface 5xx. |
| Redis unavailable | Cache reads/writes fail; campaign retrieval and customer context degrade. | Decision returns empty / default-only set. Job locks fail-safe (skip vs. duplicate-run trade-off — see §20). |
| Kafka publish failure | Event emit logged + retried via Spring Kafka retry semantics; row still written to `event_log` for replay. | Re-emit can be triggered by replaying log rows. |
| SQS message bad payload | Caught + logged; **message still deleted** to avoid poison-message redrive loops in payment processing. | Errors counted in moving window; threshold trips error halt. |
| Payment file partial corruption | Per-line errors counted in 10k-row sliding window; if `errorRate > 0.001` (`paymentProcessor.error-rate-threshold`), processing aborts. | Manual replay via `/ads/v1/backfill/payment`. |
| Eligibility job worker crash | Heartbeat goes stale (>60s) → next instance picks up. | Lock TTL 5min ensures recovery. |
| Cache stampede on campaign refresh | Per-country distributed lock prevents simultaneous rebuilds. | Stale data continues to serve while rebuild in progress. |
| Audience Service eval timeout | Targeting filter fails open or closed depending on flag. | Configured per region. |

---

## 16. Security & Access Control

- **TLS:** All inbound is HTTPS via the K8s ingress (Block platform).
- **AuthZ:** Consumer endpoints rely on consumer identity propagated by upstream gateway → `AdUserContext`. Admin endpoints currently rely on internal-network + identity headers (no in-process role check is visible — verify with platform-iam team for policy).
- **AWS access:** IAM role attached via IRSA in `infra/k8s/app/envs/*.yaml`. To request additional S3 buckets etc., update the role policy in [platform-iam](https://github.com/AfterpayTouch/platform-iam) (cross-ref question #4 in `knowledge-base/questions.md`).
- **Secrets:** AWS Secrets Manager (auto-config explicitly disabled to prevent surprise lookups; secrets injected as env).
- **PII at rest:** KMS-encrypted. PII fields in events are encrypted via `pii-protection` lib.
- **AWS SigV4 for upstream calls:** `feign-aws-sigv4-sdkv2` 3.0.4.
- **Schema-Registry credentials:** via env (`KAFKA_SCHEMA_REGISTRY_URL`).

---

## 17. Configuration & Feature Flags

### 17.1 Spring profiles & files

- `application.yml` — base.
- `application-prod.yml` — prod overrides.
- Local profile keeps `cloud.aws.sqs.listener.auto-startup=false` to avoid cloud listener startup.

### 17.2 Selected env vars (non-exhaustive)

```
DB_HOST / DB_PORT / DB_USERNAME / DB_PASSWORD
KAFKA_BOOTSTRAP_SERVERS / KAFKA_SCHEMA_REGISTRY_URL
ADS_CAMPAIGN_TOPIC / ADS_ADVERTISER_TOPIC
NIFT_API_ENDPOINT_{US,GB} / NIFT_AUTH_CLIENT{ID,SECRET,SCOPE}_{US,GB} / NIFT_REFERRAL_CODE_{US,GB}
NIFT_AWS_SQS_Queue_Name_{US,GB}
NIFT_ELIGIBLE_USER_CALCULATION_CRON_{US,GB}
PAYMENT_EVENT_WORKER_ENABLED
PRIVACY_VAULT_FORGET_ENTITY_{NA,EU,ANZ}_QUEUE_NAME
PRIVACY_VAULT_EXPORT_FOR_ENTITY_{NA,EU,ANZ}_QUEUE_NAME
SQS_LISTENER_ENABLED
UDP_USER / UDP_PASSWORD / UDP_API_{CONNECTION,SOCKET}_TIMEOUT
DATA_API_{CONNECTION,SOCKET}_TIMEOUT
PAYMENT_PROCESSOR_ERROR_RATE_THRESHOLD / PAYMENT_PROCESSOR_CHECKING_WINDOW_SIZE
LD_SDK_KEY    # LaunchDarkly
```

### 17.3 LaunchDarkly flags (handlers under `service/handler/`)

| Handler | Flags |
|---|---|
| `AdvertiserFeatureFlagHandler` | per-advertiser ads/affiliate/promo/storefront enablement |
| `AffiliateFlagHandler` | affiliate-program eligibility |
| `PromoFlagHandler` | promotion-feature gating |

### 17.4 Country-specific config

`Country` enum drives per-region behavior:

| Country | Risk Filter | Relevancy Ranking | Currency | Locale | House Advertiser ID |
|---|---|---|---|---|---|
| US | ✅ | ✅ | USD | en-US | 98780 |
| AU | ✅ | ✅ | AUD | en-AU | 98784 |
| UK | ✅ | ❌ | GBP | en-GB | 98785 |
| NZ | ❌ | ❌ | NZD | en-NZ | 98786 |
| CA | ❌ | ❌ | CAD | en-CA | 120702 (en-CA) / 120703 (fr-CA) |

---

## 18. Multi-Region Behavior

- **Country-level:** `ads-service` runs as a single deployment (us-east-1 today) but serves multiple countries via the `country` parameter on requests and `Country` enum logic.
- **NIFT region split:** US vs GB queues, endpoints, credentials, cron schedules — and per-region distributed locks/heartbeats.
- **Privacy Vault region split:** NA / EU / ANZ queues, separate listener bindings.
- **House advertiser fallback:** distinct ID per region; `Locale` dual-mapping for Canada (en-CA + fr-CA) handled in `ads.house-advertiser-id-to-locale`.
- **Timezone handling (NIFT eligibility):** US payment validation spans UTC-4 to UTC-8; GB uses UTC+1.

---

## 19. Build, Test, Deploy

### 19.1 Build

```
./gradlew ktfmtFormat           # always run before commit (Google style)
./gradlew clean build           # full build w/ tests
./gradlew integrationTest       # integration tests (TestContainers/WireMock)
./gradlew jib                   # build container image to ECR
./gradlew bootRun               # local run
```

Pipeline order (per `.buildkite/`): `ktfmtCheck → generateProtos (Wire) → compile → test → integrationTest → jacocoTestReport → JIB push`.

### 19.2 Test stack

- **Unit:** JUnit 5 + Mockk + Kotest assertions (`src/test/kotlin/`).
- **Integration:** TestContainers (Postgres), WireMock (HTTP mocking), REST Assured (`src/integTest/kotlin/`). Custom `integTest` source set wired in `build.gradle.kts`.
- **Coverage:** JaCoCo (currently `minimum=0.00`, i.e. report-only).
- Static analysis: Detekt is **currently disabled** (Kotlin version compatibility issue noted in `build.gradle.kts`).

### 19.3 Container & deploy

- **JIB** builds an image on top of `361053881171.dkr.ecr.ap-southeast-2.amazonaws.com/base/jre-17-wolfi:16` and pushes to `…/k8s/marketing/ads-service`.
- **JVM flags:** `-XX:MaxRAMPercentage=85.0 -XX:InitialRAMPercentage=35 -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp -XX:ActiveProcessorCount=2 -XX:+StartAttachListener -Dlog4j2.formatMsgNoLookups=true`.
- **K8s manifests:** `infra/k8s/app/envs/{beta,omega}-usea1-v1.yaml` for app; `infra/k8s/database/envs/...` for DB.
- **Branch tagging:** `main` builds tagged `latest`; others tagged with version only.

---

## 20. Corner Cases & Known Pitfalls

This section captures behaviors that are **non-obvious from reading the code path** and would surprise a new joiner.

### Decision pipeline

1. **`CATEGORIES` placement** is a *meta-placement* — the request says `CATEGORIES`, the response keys are dynamic `CATEGORY_{id}`. Clients must handle multiple keys appearing per single placement requested.
2. **Pinned slots take priority over algorithmic ranking.** A campaign with `Placement.slot=1` will display in slot 1 regardless of priority/auction; if multiple campaigns pin the same slot, last-write or specific tie-break logic applies (verify in `AdsSelector`).
3. **Hero ad promotion** — if a `MARQUEE_M` or `MARQUEE_L` exists in a position's candidate set, it's promoted to the front. This *can override* what auction would otherwise pick.
4. **Lower priority weight = higher priority.** `SPONSORSHIP=100` beats `HOUSE=300`. Inverted from what people often assume.
5. **Default 5 ads per placement.** Hard-coded — if a client only renders 3, it's silently discarding 2.
6. **Auction defaults** — `relevancy=500`, `eCPC=0.01` if either is missing. A consumer with no relevancy data is *not* excluded from auction; they just get the average.
7. **Risk filter is silently disabled in NZ/CA** even if a campaign relies on it being filtered out.
8. **AdsRiskStoreFilter and NIFT eligibility** are entangled — NIFT campaigns short-circuit through the risk filter regardless of `country.riskFilterEnabled`.

### Stats / accounting

9. **Aggregated stats are eventually consistent** — Redis is the source of truth for the past 0–10 min; Postgres for everything older. Ad-hoc admin queries against Postgres will lag by up to 10 min.
10. **DataDog metric tags** are normalized — two advertisers `Foo Inc.` and `foo_inc` will collide. Always cross-tag with `advertiser_id`.
11. **GMV column was retroactively added** (V18). Stats rows from before V18 may have null GMV; aggregation code defaults to 0.
12. **Stats refresh emits cumulative deltas, not absolute totals** — duplicate triggers will overcount. The job is single-instance by design.

### Privacy Vault

13. **PV listener disabled by default** (`SQS_LISTENER_ENABLED=false`) — local profile won't process PV requests.
14. **PV deletion is best-effort across regions** — there are 3 separate queues (NA/EU/ANZ); a misrouted message will not be retried in another region.
15. **Audit log is fire-and-forget** — if `PrivacyVaultApiClient.sendAuditLog()` fails, the deletion has already happened. Recovery requires manual replay.

### NIFT worker

16. **Region detection from filename** — `*GB.csv` → GB, otherwise → US. Misnamed files silently route to the wrong region.
17. **SQS message always deleted** in payment processing, even on failure. This avoids poison-message storms but loses messages on bugs. Use `/ads/v1/backfill/payment` to replay from S3.
18. **Heartbeat stale window (60s) > heartbeat interval (10s)** by 6×. A briefly-paused process can time itself out.
19. **Daily-success Redis key** is timezone-dependent on the worker host (UTC). Crossing midnight during a run can yield "yesterday" success markings.
20. **Eligibility CSV split: ≤ 50,000 rows × ≤ 100 files** — exceeding 5M eligible users will silently truncate. Monitor file-count metric.
21. **NIFT auth tokens cached for 60 min** — clock skew between pods can cause "expired token" 401s near refresh; on auth circuit-breaker open, *all* eligibility calls fail, not just for one user.

### Campaign / advertiser

22. **`v2/admin/campaigns` create is multipart**, not JSON — creative is uploaded as multipart file. Existing JSON-only clients must be updated.
23. **Status enum mismatch:** DB stores `ACTIVE/INACTIVE` only; API surface synthesizes `DRAFT/PAUSED/COMPLETED/SCHEDULED` based on dates and status. Don't assume DB and API enums match.
24. **Per-advertiser cache invalidation on update is partial** — a manual `/ads/v1/admin/advertiser/refresh` call may be needed if downstream cache (Caffeine `AdvertiserLocalCache`) appears stale.
25. **Active advertiser lookup by store** uses `(storeId, publisher)` — same store under different publishers (`AP` vs. `SHOP`) returns different advertisers.

### Jira sync

26. **Idempotency relies on `JiraCampaignMapping`** — manual deletes of mapping rows can cause duplicate campaigns on the next poll.
27. **Jira poll runs every minute** — heavy queries can pressure Jira API; `JiraClient` has no explicit rate limit configured.
28. **Errors are surfaced as Jira labels** on the issue, not as service errors — search `ads-service-error` label in Jira to triage.

### General

29. **Detekt is disabled in `build.gradle.kts`** — static-analysis violations won't fail CI right now.
30. **JaCoCo coverage threshold = 0.00** — coverage is reported but not enforced. PRs that drop coverage won't be blocked.
31. **`@SpringBootApplication` excludes `SecretsManagerAutoConfiguration`** — pulling secrets must be done manually via SDK; don't expect `@Value` from secret store.
32. **`open-in-view: false`** — lazy JPA associations *will* fail in controllers. Always fetch what's needed inside the service.
33. **Schema is `ads`, not `public`** — direct SQL queries must qualify table names (`ads.advertiser`, not `advertiser`).

---

## 21. Open Questions / Future Work

These are observed gaps worth discussing — not in-flight commitments.

1. **AuthZ on admin endpoints** — verify whether identity is enforced in-process or at the ingress; document the trust boundary.
2. **Move Detekt + JaCoCo enforcement back on** — both currently disabled.
3. **Replace `event_log` write-then-emit pattern with transactional outbox** — current implementation can drop messages between commit and emit.
4. **Stats emitter single-instance assumption** — relies on jobs being singleton-deployed; consider explicit leader-election if scaled.
5. **Audience targeting evaluation** — currently per-request to Audience Service; per-consumer caching with bounded staleness would cut latency.
6. **Cap enforcement** — `CapType` exists but actual budget exhaustion logic is a single-job tick (10 min). Could overspend by a small margin between flushes.
7. **Multi-region active/active** — today the deploy is us-east-1 only; cross-region resilience is a future direction.
8. **Per-campaign quality scoring** — auction uses raw `relevancy × eCPC`. CTR-based pacing/throttling could improve revenue.

---

## Appendix A — Quick file reference

| Concern | Source |
|---|---|
| Decision orchestrator | [`service/ads/decision/AdsDecisionService.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/AdsDecisionService.kt) |
| Filters | [`service/ads/decision/filter/`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/filter/) |
| Ranking | [`service/ads/decision/ranking/`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/ranking/) |
| Selector | [`service/ads/decision/AdsSelector.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/decision/AdsSelector.kt) |
| Customer context | [`service/ads/helper/CustomerRetriever.kt`](../src/main/kotlin/com/afterpay/ads/service/ads/helper/CustomerRetriever.kt) |
| Campaigns repo | [`data/repository/ads/`](../src/main/kotlin/com/afterpay/ads/data/repository/ads/) |
| Country/Priority/AdType enums | [`model/Country.kt`](../src/main/kotlin/com/afterpay/ads/model/Country.kt), [`model/ads/Priority.kt`](../src/main/kotlin/com/afterpay/ads/model/ads/Priority.kt), [`model/ads/AdType.kt`](../src/main/kotlin/com/afterpay/ads/model/ads/AdType.kt) |
| Migrations | [`src/main/resources/db/migration/`](../src/main/resources/db/migration/) |
| Build config | [`build.gradle.kts`](../build.gradle.kts) |
| Infra (k8s) | [`infra/k8s/app/envs/`](../infra/k8s/app/envs/), [`infra/k8s/database/envs/`](../infra/k8s/database/envs/) |
| Knowledge base (narrative) | [`knowledge-base/`](../knowledge-base/) |
| AI memory bank | [`ai-memory/`](../ai-memory/) |

## Appendix B — Slack / on-call

- Alerts: `#ad-tech-alerts`, `#block-martech-server-oncall`
- Working group / support: `#ap-ad-tech-wg`
- NIFT: `#ext-ap-nift`, `#nift-uk`
