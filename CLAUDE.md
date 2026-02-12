# CLAUDE.md — 图片素材平台 (Image Asset Platform)

## Project Overview

A free image asset platform built with PHP + MySQL, designed to help other websites easily find and use image resources. The platform collects images, recognizes their content via AI, auto-generates names/descriptions/tags, and provides search and browsing capabilities.

**Core Mission**: Make it easy for other websites to find high-quality image assets (素材).

**Scale Target**: Support 1M–10M+ images with high concurrent access.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | PHP 8.2+ |
| Framework | Laravel 11 |
| Database | MySQL 8.0 |
| Frontend | Blade templates + Tailwind CSS + Alpine.js |
| Image Processing | Intervention Image (PHP) |
| Storage | Local filesystem (dev) / **Cloudflare R2** (prod) — S3-compatible, **zero egress fees** |
| CDN | **Cloudflare CDN** — free tier or Pro ($20/mo), paired with R2 for zero-cost image delivery |
| Queue | Laravel Queue (Redis/database driver) for async image processing |
| Cache | Redis (required for production scale) |
| Search | Meilisearch (recommended for millions of records) / MySQL fulltext (dev only) |
| AI Vision | TBD — pluggable adapter interface, to be decided later |

---

## Project Structure

```
img/
├── CLAUDE.md                  # This file — project guide for AI assistants
├── app/
│   ├── Console/Commands/      # Artisan CLI commands
│   │   ├── CrawlCommand.php          # php artisan img:crawl {site}
│   │   ├── CrawlUrlCommand.php       # php artisan img:crawl-url {url}
│   │   └── FlushCountersCommand.php  # Flush Redis counters to MySQL
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── ImageController.php       # Image CRUD, search, detail pages
│   │   │   ├── CategoryController.php    # Category/tag browsing
│   │   │   ├── HomeController.php        # Homepage, trending images
│   │   │   ├── UploadController.php      # Image upload handling
│   │   │   └── Api/                      # API controllers for external access
│   │   ├── Middleware/
│   │   └── Requests/                     # Form request validation
│   ├── Models/
│   │   ├── Image.php           # Core image model
│   │   ├── Tag.php             # Image tags
│   │   ├── Category.php        # Image categories
│   │   └── ImageHash.php       # Perceptual hash for deduplication
│   ├── Services/
│   │   ├── ImageService.php           # Image business logic
│   │   ├── DuplicateDetector.php      # Duplicate image detection (perceptual hash)
│   │   ├── ImageProcessor.php         # Resize, thumbnail, format conversion
│   │   ├── ColorExtractor.php         # Extract dominant colors from images
│   │   ├── StorageManager.php         # Abstraction over local/cloud storage
│   │   ├── Crawler/                   # Image crawler system
│   │   │   ├── CrawlerInterface.php   # Interface for site crawlers
│   │   │   ├── GenericCrawler.php     # Generic HTML img extractor
│   │   │   ├── CrawlerManager.php     # Manages crawlers, scheduling
│   │   │   └── Adapters/             # Site-specific crawlers
│   │   │       ├── UnsplashAdapter.php
│   │   │       └── PixabayAdapter.php
│   │   └── AiVision/
│   │       ├── VisionInterface.php    # Interface for AI recognition adapters
│   │       ├── ClaudeVisionAdapter.php # Claude Vision implementation (future)
│   │       └── NullAdapter.php        # No-op adapter for when AI is disabled
│   └── Jobs/
│       ├── ProcessImageUpload.php     # Async: generate thumbnails, extract metadata
│       ├── RecognizeImageContent.php  # Async: AI recognition, tagging
│       ├── DetectDuplicate.php        # Async: check for duplicate images
│       ├── CrawlSite.php             # Async: crawl a site/page for images
│       └── DownloadImage.php          # Async: download and process one image
├── config/
│   ├── image.php              # Image platform config (sizes, formats, limits)
│   ├── crawler.php            # Crawler config (sites, delays, user-agent)
│   └── ai-vision.php          # AI vision adapter config
├── database/
│   └── migrations/            # Database schema migrations
├── public/
│   ├── index.php              # Entry point
│   └── storage/               # Symlink to storage (local images)
├── resources/
│   ├── views/
│   │   ├── layouts/           # Base layouts
│   │   ├── home.blade.php     # Homepage — trending & recent images
│   │   ├── images/
│   │   │   ├── index.blade.php   # Image gallery / search results
│   │   │   ├── show.blade.php    # Image detail page
│   │   │   └── upload.blade.php  # Upload form
│   │   └── categories/
│   ├── css/
│   └── js/
├── routes/
│   ├── web.php                # Web routes
│   └── api.php                # API routes for external consumers
├── storage/
│   └── app/images/            # Local image storage
├── tests/
│   ├── Feature/               # Feature/integration tests
│   └── Unit/                  # Unit tests
├── composer.json
├── package.json
├── .env.example
├── .gitignore
└── vite.config.js
```

---

## Core Features & Requirements

### 1. Image Collection & Upload
- Manual upload via web form (single & batch)
- URL-based image import (paste a URL, system downloads and stores)
- **Automated crawler** (see Section 11 below) for bulk collection from other websites
- Support formats: JPEG, PNG, GIF, WebP, SVG, BMP, TIFF
- Max file size: configurable (default 20MB)
- Auto-extract EXIF metadata on upload

### 2. Duplicate Image Detection
- **Perceptual hashing** (pHash/dHash) to detect visually identical images
- Compare hash on upload; reject if duplicate already exists
- Store hashes in `image_hashes` table for fast lookup
- Allow configurable similarity threshold (exact match vs near-duplicate)

### 3. AI Image Recognition (Pluggable — TBD)
- Adapter pattern via `VisionInterface`
- Auto-generate: image title/name, description, tags/keywords
- Runs asynchronously via queue jobs
- NullAdapter available when AI is not configured
- Future adapters: Claude Vision, OpenAI Vision, BLIP/CLIP

### 4. Image Metadata & Display
- **Display**: filename, title, description, tags, categories
- **Technical metadata**: width, height, file size (human-readable), format, color space, DPI
- **Platform metadata**: upload date, view count, download count, popularity score

### 5. Image Loading Optimization
- **Thumbnails**: Auto-generate multiple sizes (small: 150px, medium: 400px, large: 800px)
- **Lazy loading**: Use `loading="lazy"` attribute + Intersection Observer
- **WebP conversion**: Auto-convert uploads to WebP for serving (keep original)
- **Progressive JPEG**: Generate progressive JPEGs for large images
- **CDN-ready**: Storage abstraction supports CDN URL prefixing
- **Blurhash/LQIP**: Generate low-quality image placeholders for loading states
- **Pagination**: Cursor-based or offset pagination, default 24 images per page
- **Responsive images**: `srcset` with multiple resolutions

### 6. Search & Discovery
- **Fulltext search**: Search by title, description, tags
- **Filter by**:
  - Category (分类): 动图, 植物, 人物, 动漫, etc.
  - Format (格式): PNG, JPG, GIF, WebP
  - Dimensions (尺寸): icon/small/medium/large/xlarge presets, or custom min/max
  - Aspect ratio (比例): landscape(横图), portrait(竖图), square(方图)
  - File size range
  - Color (dominant color filter)
  - Date range
  - Source domain (来源网站)
- **Sort by**: newest, most viewed, most downloaded, trending
- **Trending/Hot**: Based on view count in recent time window (e.g., last 7 days)
- **Related images**: Show similar images on detail page (by tags/category)
- **Format gallery**: Dedicated pages per format (e.g., `/images/gif` shows all GIFs)
- **Size gallery**: Dedicated pages per size preset (e.g., `/images/large` shows all large images)

### 7. Categories & Tags
- Hierarchical categories (parent/child)
- Free-form tags (many-to-many with images)
- Tag cloud on homepage
- Category browsing pages with image count

#### Predefined Top-Level Categories (初始分类)
| Slug | Name | Description |
|------|------|-------------|
| `dongtu` | 动图/GIF | Animated images, GIF animations |
| `zhiwu` | 植物 | Plants, flowers, trees, nature |
| `renwu` | 人物 | People, portraits, lifestyle |
| `dongman` | 动漫 | Anime, manga, cartoon illustrations |
| `dongwu` | 动物 | Animals, pets, wildlife |
| `fengjing` | 风景 | Landscapes, scenery, travel |
| `meishi` | 美食 | Food, cooking, recipes |
| `keji` | 科技 | Technology, devices, UI/UX |
| `shangwu` | 商务 | Business, office, corporate |
| `beijing` | 背景 | Backgrounds, textures, patterns |
| `tubiao` | 图标 | Icons, logos, symbols |
| `qita` | 其他 | Other / uncategorized |

- Categories are seeded via `database/seeders/CategorySeeder.php`
- Each top-level category can have sub-categories (e.g., 动漫 > 日漫, 国漫, 欧美动漫)
- New categories can be added by admin; AI recognition can suggest categories

### 8. API for External Websites
- RESTful JSON API under `/api/v1/`
- Endpoints: search, list, image detail, categories, tags
- Rate limiting (configurable)
- Optional API key authentication
- CORS support for cross-origin usage
- Response includes direct image URLs + metadata

### 9. Storage System (Cloudflare R2 — 最省方案)

**Why R2**: S3-compatible API, storage $0.015/GB, **出站流量完全免费 ($0)**. 100万张图片约 $42/月，不限流量。

- **Dev**: Local filesystem (`FILESYSTEM_DISK=local`)
- **Prod**: Cloudflare R2 (`FILESYSTEM_DISK=r2`) — uses S3-compatible driver in Laravel
- **CDN**: Cloudflare CDN automatically caches R2 objects via custom domain (zero config)
- Configured via `FILESYSTEM_DISK` in `.env`
- Images organized by hash sharding: `images/{hash[0:2]}/{hash[2:4]}/{hash}.{ext}` (supports millions of files)
- Thumbnails stored alongside: `images/{hash[0:2]}/{hash[2:4]}/{hash}_thumb_{size}.webp`
- Original files always preserved
- R2 public bucket with custom domain (e.g., `img.yourdomain.com`) for direct CDN access

#### Cost Estimate (100万张图片)
| Item | Cost |
|------|------|
| R2 Storage (~2.8TB) | ~$42/月 |
| R2 Egress (出站流量) | **$0 (免费)** |
| R2 Class A ops (写入, 10万次/月) | ~$4.50 |
| R2 Class B ops (读取, 1000万次/月) | ~$3.60 |
| Cloudflare Pro (CDN + 安全) | $20/月 |
| **合计** | **~$70/月** |

> 对比 AWS S3 中流量场景 ~$1,414/月，**省了 95%+**

### 10. Scalability & High Concurrency (百万~千万级)

The platform is designed to support 1M–10M+ images with high concurrent users.

#### Database Optimization
- **Indexing strategy**: Composite indexes on frequently queried columns (`format + created_at`, `category_id + created_at`, etc.)
- **Table partitioning**: Partition `images` table by `created_at` (RANGE partitioning by month/year) for faster queries on large datasets
- **Read/Write separation**: Use MySQL replication — writes go to primary, reads go to replica(s). Configure in Laravel via `read`/`write` database connections
- **ID strategy**: Use BIGINT UNSIGNED auto-increment for internal PKs; UUID for public-facing identifiers (avoids exposing record counts)
- **Counter caching**: `view_count` and `download_count` incremented via Redis first, flushed to MySQL periodically (every N minutes) to reduce write pressure
- **Avoid COUNT(*)**: Use cached counters in `tags.image_count` and `categories.image_count`, updated via events
- **Query optimization**: Never use `SELECT *`; always select specific columns. Use `chunk()` or cursor for batch operations. Avoid N+1 queries via eager loading

#### Caching Architecture (Redis Required)
- **Multi-layer cache**:
  - L1: Application-level (Laravel model caching for hot data)
  - L2: Redis (page cache, query result cache, image metadata cache)
  - L3: Cloudflare CDN (R2 image objects cached at edge, static assets)
- **Cache strategies**:
  - Homepage/trending: Cache for 5 minutes, warm via scheduled task
  - Image detail pages: Cache until image metadata changes (event-based invalidation)
  - Search results: Cache for 2 minutes with query-based cache keys
  - Category/tag pages: Cache for 10 minutes
  - API responses: Cache with appropriate `Cache-Control` headers
- **Cache keys**: Namespaced (`img:detail:{id}`, `img:trending:{page}`, `img:search:{hash}`)
- **Redis usage**: Sessions, cache, queue broker, rate limiting counters, view count buffer

#### High Concurrency Handling
- **Nginx**: Serve static files directly (images, CSS, JS); never go through PHP for static assets
- **PHP-FPM tuning**: Dynamic process manager, tune `pm.max_children` based on available RAM
- **OPcache**: Enable with appropriate settings (`opcache.memory_consumption=256`, `opcache.max_accelerated_files=20000`)
- **Connection pooling**: Use persistent database connections; consider ProxySQL for connection pooling at scale
- **Rate limiting**: Per-IP rate limiting on uploads and API (Laravel `ThrottleRequests` middleware)
- **Queue workers**: Multiple queue workers (Horizon recommended) for parallel async processing
- **Static asset optimization**: Vite build with minification, gzip/brotli compression in Nginx

#### Image Storage at Scale (Cloudflare R2 + CDN)
- **Directory sharding**: Hash-based key structure in R2: `images/{hash[0:2]}/{hash[2:4]}/{hash}.{ext}` (~65K prefix groups for efficient listing)
- **CDN**: Cloudflare CDN auto-caches R2 objects via custom domain — zero egress cost, global edge delivery
- **Cache-Control headers**: Set `Cache-Control: public, max-age=31536000, immutable` on image objects (content-addressed, never changes)
- **Thumbnail pre-generation**: Generate all thumbnail sizes at upload time, not on-demand
- **Storage cleanup**: Scheduled artisan command to remove orphaned R2 objects
- **R2 lifecycle rules**: Optional — auto-delete temporary upload chunks after 24h

#### Search at Scale
- **Meilisearch**: Required for production with millions of records. MySQL fulltext degrades beyond 100K rows
- **Laravel Scout**: Use Scout driver for Meilisearch integration — keeps search index in sync with database
- **Search fields**: Index `title`, `description`, `tags`, `category_name`
- **Filterable attributes**: `format`, `width`, `height`, `file_size`, `created_at`, `category_id`
- **Sortable attributes**: `created_at`, `view_count`, `download_count`, `file_size`

#### Monitoring & Performance
- **Laravel Telescope**: For development debugging (disable in production)
- **Slow query log**: Enable MySQL slow query log (threshold: 1s)
- **Application metrics**: Track response times, queue depth, cache hit rates
- **Health check endpoint**: `/api/health` — checks DB, Redis, storage, queue connectivity

### 11. Image Crawler / Spider (图片爬虫)

Automated system to crawl and collect images from other websites.

#### Architecture
- **Crawler engine**: Artisan commands + queue jobs for async crawling
- **Site adapters**: Each target website has its own adapter/spider class implementing `CrawlerInterface`
- **Politeness**: Respect `robots.txt`, configurable delay between requests, User-Agent identification
- **Scheduling**: Laravel Scheduler runs crawl tasks at configurable intervals

#### Crawler Pipeline
```
Target URL → Download HTML → Extract image URLs → Filter (size/format/quality)
    → Download image → Duplicate check (pHash) → Extract metadata
    → Generate thumbnails → Save to storage → Index in database
    → (Optional) AI recognition via queue
```

#### Key Components
| File | Purpose |
|------|---------|
| `app/Services/Crawler/CrawlerInterface.php` | Interface for all site crawlers |
| `app/Services/Crawler/GenericCrawler.php` | Generic crawler (extracts all `<img>` from any URL) |
| `app/Services/Crawler/Adapters/*.php` | Site-specific adapters (Unsplash, Pixabay, etc.) |
| `app/Jobs/CrawlSite.php` | Queue job: crawl a single site/page |
| `app/Jobs/DownloadImage.php` | Queue job: download, validate, and store one image |
| `app/Console/Commands/CrawlCommand.php` | `php artisan img:crawl {site} {--pages=10}` |
| `config/crawler.php` | Crawler config (sites, delays, limits, user-agent) |

#### Crawl Rules & Filtering
- **Min dimensions**: Skip images smaller than configurable threshold (default: 200x200 px)
- **Min file size**: Skip images under 10KB (likely icons/spacers)
- **Format filter**: Only collect JPEG, PNG, GIF, WebP (skip SVG/BMP by default)
- **Duplicate check**: pHash comparison before storing — reject if already exists
- **Source tracking**: Store `source_url` and `source_domain` for every crawled image
- **Rate limiting**: Configurable delay per domain (default: 2 seconds between requests)
- **Max concurrent**: Limit parallel download jobs per domain

#### Crawler Database Table

##### crawl_tasks
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT UNSIGNED PK | Auto-increment |
| site_name | VARCHAR(100) | Target site identifier |
| target_url | VARCHAR(2048) | URL to crawl |
| status | ENUM('pending','running','completed','failed') | Task status |
| images_found | INT UNSIGNED DEFAULT 0 | Total images discovered |
| images_saved | INT UNSIGNED DEFAULT 0 | Images actually saved (after dedup) |
| images_skipped | INT UNSIGNED DEFAULT 0 | Images skipped (duplicate/too small) |
| error_message | TEXT NULL | Error details if failed |
| started_at | TIMESTAMP NULL | When crawl started |
| completed_at | TIMESTAMP NULL | When crawl finished |
| created_at | TIMESTAMP | Task creation time |

#### Crawler Commands
```bash
# Crawl a specific site (using adapter)
php artisan img:crawl unsplash --pages=10 --category=fengjing

# Crawl a generic URL (extracts all images)
php artisan img:crawl-url "https://example.com/gallery" --min-width=400

# List crawl history
php artisan img:crawl-status

# Retry failed crawl tasks
php artisan img:crawl-retry --failed
```

### 12. Image Detail Page & Metadata Display (图片详情页)

Each image has a detail page showing all information.

#### Display Fields
- **Title** (标题): Auto-generated or manual, used as page `<title>` for SEO
- **Description** (描述): What the image contains — e.g., "一只橘色的猫咪坐在窗台上晒太阳" (AI-generated or manual)
- **Category** (分类): e.g., 动物 > 猫咪
- **Tags** (标签): e.g., 猫, 橘猫, 宠物, 窗台
- **Format** (格式): PNG, JPG, GIF, WebP — displayed with format icon/badge
- **Dimensions** (尺寸): e.g., 1920×1080, 64×64 — displayed as `宽×高`
- **Size preset label** (尺寸类型): Auto-classified:
  - 图标 (Icon): ≤128×128
  - 小图 (Small): ≤640×640
  - 中图 (Medium): ≤1280×1280
  - 大图 (Large): ≤2560×2560
  - 超大图 (Extra Large): >2560
- **File size** (文件大小): Human-readable, e.g., "2.4 MB", "156 KB"
- **Color palette** (主色调): Extract dominant colors from image (5 colors)
- **Source** (来源): Original source URL if crawled
- **Upload date** (上传时间)
- **View count** (浏览量)
- **Download count** (下载量)
- **Download button**: Direct download of original file
- **Related images**: Based on same category/tags

#### Dimension-Based Browsing & Filtering
Users can filter/browse images by dimensions:
- **Preset sizes**: 图标(≤128), 小图(≤640), 中图(≤1280), 大图(≤2560), 超大图(>2560)
- **Custom range**: Filter by min/max width and height
- **Common resolutions**: Quick filter for 1920×1080, 1080×1080, 800×600, etc.
- **Aspect ratio filter**: 横图(landscape), 竖图(portrait), 方图(square)

#### Format-Based Browsing
- Browse by format: `/images?format=png`, `/images?format=gif`
- Format badges on thumbnails in gallery view (e.g., "GIF" badge on animated images)
- GIF/animated images auto-play on hover in gallery

---

## Database Schema (Key Tables)

> **Note**: All tables use InnoDB engine. The `images` table should be partitioned by `created_at` when exceeding 1M rows.

### images
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT UNSIGNED PK | Auto-increment |
| uuid | CHAR(36) UNIQUE | Public identifier |
| title | VARCHAR(255) | Auto-generated or manual title |
| description | TEXT NULL | AI-generated or manual description |
| original_filename | VARCHAR(255) | Original upload filename |
| storage_path | VARCHAR(500) | Path in storage disk |
| mime_type | VARCHAR(50) | e.g., image/jpeg |
| format | VARCHAR(10) | e.g., jpg, png, webp |
| file_size | BIGINT UNSIGNED | Size in bytes |
| width | INT UNSIGNED | Pixel width |
| height | INT UNSIGNED | Pixel height |
| blurhash | VARCHAR(100) NULL | Blurhash placeholder string |
| category_id | BIGINT UNSIGNED NULL FK | Primary category |
| source_url | VARCHAR(2048) NULL | Original source URL if crawled/imported |
| source_domain | VARCHAR(255) NULL | Domain of source (e.g., unsplash.com) |
| size_label | ENUM('icon','small','medium','large','xlarge') | Auto-classified by dimensions |
| aspect_ratio | ENUM('landscape','portrait','square') | Auto-classified w/h ratio |
| dominant_colors | JSON NULL | Array of hex color strings (top 5) |
| is_animated | BOOLEAN DEFAULT FALSE | True for GIF/animated WebP |
| view_count | INT UNSIGNED DEFAULT 0 | Total views |
| download_count | INT UNSIGNED DEFAULT 0 | Total downloads |
| ai_processed | BOOLEAN DEFAULT FALSE | Whether AI recognition has run |
| created_at | TIMESTAMP | Upload time |
| updated_at | TIMESTAMP | Last update time |

**Key indexes on `images`**:
- `idx_images_category_created` (category_id, created_at)
- `idx_images_format_created` (format, created_at)
- `idx_images_created_at` (created_at)
- `idx_images_view_count` (view_count DESC)
- `idx_images_size_label` (size_label, created_at)
- `idx_images_aspect_ratio` (aspect_ratio, created_at)
- `idx_images_uuid` (uuid) — UNIQUE
- `idx_images_source_domain` (source_domain) — for tracking crawl sources
- FULLTEXT index on (title, description) — dev environment only

### image_hashes
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT UNSIGNED PK | Auto-increment |
| image_id | BIGINT UNSIGNED FK | Reference to images |
| phash | CHAR(64) | Perceptual hash |
| dhash | CHAR(64) | Difference hash |

**Key indexes on `image_hashes`**:
- `idx_hashes_phash` (phash) — for duplicate detection lookup
- `idx_hashes_image_id` (image_id) — FK lookup

### tags
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT UNSIGNED PK | Auto-increment |
| name | VARCHAR(100) UNIQUE | Tag name |
| slug | VARCHAR(100) UNIQUE | URL-friendly name |
| image_count | INT UNSIGNED DEFAULT 0 | Cached count |

### image_tag (pivot)
| Column | Type |
|--------|------|
| image_id | BIGINT UNSIGNED FK |
| tag_id | BIGINT UNSIGNED FK |

### categories
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT UNSIGNED PK | Auto-increment |
| parent_id | BIGINT UNSIGNED NULL FK | Self-referencing for hierarchy |
| name | VARCHAR(100) | Category name |
| slug | VARCHAR(100) UNIQUE | URL-friendly name |
| description | TEXT NULL | Category description |
| cover_image_id | BIGINT UNSIGNED NULL FK | Representative image for category |
| image_count | INT UNSIGNED DEFAULT 0 | Cached count |
| sort_order | INT UNSIGNED DEFAULT 0 | Display order |

### crawl_tasks
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT UNSIGNED PK | Auto-increment |
| site_name | VARCHAR(100) | Target site identifier |
| target_url | VARCHAR(2048) | URL to crawl |
| status | ENUM('pending','running','completed','failed') | Task status |
| images_found | INT UNSIGNED DEFAULT 0 | Total images discovered |
| images_saved | INT UNSIGNED DEFAULT 0 | Saved after dedup |
| images_skipped | INT UNSIGNED DEFAULT 0 | Skipped (dup/too small) |
| error_message | TEXT NULL | Error if failed |
| started_at | TIMESTAMP NULL | |
| completed_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |

**Key indexes on `crawl_tasks`**:
- `idx_crawl_status` (status, created_at)
- `idx_crawl_site` (site_name, created_at)

---

## Development Workflow

### Prerequisites
- PHP 8.2+ with extensions: gd, exif, fileinfo, mbstring, pdo_mysql, redis
- Composer 2.x
- MySQL 8.0+
- Node.js 18+ & npm (for frontend assets)
- Redis 7+ (required for cache, queue, sessions, counters)
- Meilisearch (required for production search at scale)

### Setup
```bash
composer install
cp .env.example .env
php artisan key:generate
# Edit .env with database credentials
php artisan migrate
php artisan storage:link
npm install && npm run dev
php artisan serve
```

### Common Commands
```bash
# Development server
php artisan serve

# Run migrations
php artisan migrate

# Run queue worker (for async image processing)
php artisan queue:work

# Clear caches
php artisan cache:clear
php artisan config:clear
php artisan view:clear

# Run tests
php artisan test
# or
./vendor/bin/phpunit

# Code style check (if using Pint)
./vendor/bin/pint --test

# Code style fix
./vendor/bin/pint

# Build frontend assets
npm run build
npm run dev   # development with HMR
```

### Testing
- **Unit tests**: `tests/Unit/` — test services, models in isolation
- **Feature tests**: `tests/Feature/` — test HTTP endpoints, full request lifecycle
- Run with `php artisan test` or `./vendor/bin/phpunit`
- Test database should use a separate MySQL database or SQLite in-memory

---

## Code Conventions

### PHP
- Follow **PSR-12** coding standard
- Use **Laravel Pint** for code formatting
- Type hints on all method parameters and return types
- Use Form Request classes for validation (not inline validation)
- Business logic in Service classes, not in Controllers
- Controllers should be thin — delegate to services
- Use Eloquent relationships, avoid raw SQL unless necessary for performance
- Queue heavy operations (image processing, AI calls)
- Never use `SELECT *` — always specify columns, especially on the `images` table
- Use `chunk()` or `cursor()` for batch operations on large datasets
- Always eager load relationships to prevent N+1 queries (`with()`)
- Use Redis for counters (views, downloads) — flush to DB via scheduled command
- Add database indexes for any new query patterns

### Naming
- **Models**: singular PascalCase (`Image`, `Tag`, `Category`)
- **Controllers**: PascalCase with `Controller` suffix (`ImageController`)
- **Services**: PascalCase with `Service/Detector/Processor` suffix
- **Migrations**: Laravel default snake_case timestamp format
- **Routes**: kebab-case URLs (`/images/search`, `/api/v1/images`)
- **Database**: snake_case table and column names
- **Config keys**: snake_case with dot notation (`image.thumbnail.sizes`)

### Frontend
- Blade templates with Tailwind CSS utility classes
- Alpine.js for interactive components (modals, dropdowns, lazy load triggers)
- Responsive design — mobile-first approach
- Semantic HTML with proper alt text on images
- All user-facing text in Chinese (zh-CN) as primary language

### Git
- Commit messages in English
- Conventional format: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`
- Keep commits atomic — one logical change per commit
- Branch naming: `feature/xxx`, `fix/xxx`, `refactor/xxx`

---

## Key Design Decisions

1. **Free platform**: No payment, no user accounts required for browsing/downloading. Admin panel for content management only.
2. **Duplicate rejection**: Images that match existing entries (by perceptual hash) are rejected at upload time with a message linking to the existing image.
3. **Async processing**: All heavy operations (thumbnail generation, AI recognition, hash computation) run in background queue jobs to keep uploads fast.
4. **Storage: Cloudflare R2**: Production uses R2 (S3-compatible, zero egress fees). Dev uses local filesystem. Switch via `FILESYSTEM_DISK` env variable.
5. **AI as plugin**: The AI vision system is behind an interface. The platform works fully without AI configured — titles/descriptions can be set manually.
6. **SEO-friendly**: Clean URLs, proper meta tags, structured data (schema.org ImageObject), sitemap generation.
7. **API-first mindset**: While the web UI is the primary interface, all data is also available via REST API for programmatic access by other websites.

---

## Security Considerations

- Validate all uploaded files by MIME type AND file content (not just extension)
- Sanitize filenames; never use original filenames in storage paths (use hashes/UUIDs)
- Rate limit uploads and API endpoints
- CSRF protection on all web forms (Laravel built-in)
- XSS prevention via Blade's `{{ }}` auto-escaping
- SQL injection prevention via Eloquent parameterized queries
- Image files served from `storage/` symlink, never from `app/` directory
- Max upload size enforced at PHP, web server, and application levels

---

## Environment Variables (Key Ones)

```env
APP_NAME="图片素材平台"
APP_ENV=local
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=img_platform
DB_USERNAME=root
DB_PASSWORD=

FILESYSTEM_DISK=local          # 'local' (dev) or 'r2' (prod)

# Cloudflare R2 (S3-compatible, zero egress fees)
CLOUDFLARE_R2_ACCESS_KEY_ID=
CLOUDFLARE_R2_SECRET_ACCESS_KEY=
CLOUDFLARE_R2_BUCKET=img-platform
CLOUDFLARE_R2_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
CLOUDFLARE_R2_URL=https://img.yourdomain.com   # Custom domain via Cloudflare CDN

QUEUE_CONNECTION=redis          # 'sync' (dev), 'redis' (prod)
CACHE_STORE=redis              # 'file' (dev), 'redis' (prod)

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=null

MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=

AI_VISION_DRIVER=null          # 'null', 'claude', 'openai' (TBD)
AI_VISION_API_KEY=

IMAGE_MAX_UPLOAD_SIZE=20480    # KB
IMAGE_THUMBNAIL_SIZES=150,400,800
IMAGE_DUPLICATE_THRESHOLD=5    # Hamming distance threshold for pHash

# Crawler settings
CRAWLER_USER_AGENT="ImgPlatformBot/1.0"
CRAWLER_DELAY_MS=2000          # Delay between requests per domain
CRAWLER_MAX_CONCURRENT=3       # Max parallel downloads per domain
CRAWLER_MIN_WIDTH=200          # Min image width to collect
CRAWLER_MIN_HEIGHT=200         # Min image height to collect
CRAWLER_MIN_FILESIZE=10240     # Min file size in bytes (10KB)
```

---

## Notes for AI Assistants

- This is a **Chinese-language** image asset platform. UI text should be in Simplified Chinese (zh-CN).
- The project uses **Laravel 11** conventions. Refer to Laravel 11 docs, not older versions.
- When adding new features, always consider the async queue pattern for heavy operations.
- The AI vision system is not yet implemented — use the `NullAdapter` as default.
- Image deduplication is a core feature — never skip hash checking on upload.
- Always generate thumbnails when processing new images.
- Test with various image formats (JPEG, PNG, GIF, WebP) to ensure compatibility.
- Keep the API and web UI in sync — any data accessible on the web should also be available via API.

### Scalability Reminders
- **ALWAYS** think about performance when writing queries — this database will have millions of rows.
- Never use `COUNT(*)` on large tables without caching the result.
- Never use `OFFSET` for deep pagination — use cursor-based pagination (`where id > :last_id`) on list endpoints.
- Always add appropriate indexes when introducing new query patterns.
- Use Redis to buffer write-heavy operations (view counts, download counts).
- Any new list/search endpoint must be paginated with a reasonable default (24 items).
- Image file paths must use hash-based directory sharding, never flat directories.
- All heavy processing (thumbnails, AI, hashing) must go through the queue — never block HTTP requests.
- Cache aggressively but invalidate correctly — use model events for cache invalidation.
- When adding new features, consider the impact on the `images` table query performance.
