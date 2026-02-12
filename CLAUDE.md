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
| Storage | Local filesystem (dev) / Cloud OSS/S3 (prod), configurable via Laravel Filesystem |
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
│   ├── Console/Commands/      # Artisan CLI commands (image import, cleanup, etc.)
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
│   │   ├── StorageManager.php         # Abstraction over local/cloud storage
│   │   └── AiVision/
│   │       ├── VisionInterface.php    # Interface for AI recognition adapters
│   │       ├── ClaudeVisionAdapter.php # Claude Vision implementation (future)
│   │       └── NullAdapter.php        # No-op adapter for when AI is disabled
│   └── Jobs/
│       ├── ProcessImageUpload.php     # Async: generate thumbnails, extract metadata
│       ├── RecognizeImageContent.php  # Async: AI recognition, tagging
│       └── DetectDuplicate.php        # Async: check for duplicate images
├── config/
│   ├── image.php              # Image platform config (sizes, formats, limits)
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
- **Filter by**: category, format, size range, color, date range
- **Sort by**: newest, most viewed, most downloaded, trending
- **Trending/Hot**: Based on view count in recent time window (e.g., last 7 days)
- **Related images**: Show similar images on detail page (by tags/category)

### 7. Categories & Tags
- Hierarchical categories (parent/child)
- Free-form tags (many-to-many with images)
- Tag cloud on homepage
- Category browsing pages

### 8. API for External Websites
- RESTful JSON API under `/api/v1/`
- Endpoints: search, list, image detail, categories, tags
- Rate limiting (configurable)
- Optional API key authentication
- CORS support for cross-origin usage
- Response includes direct image URLs + metadata

### 9. Storage System
- **Dual mode**: local filesystem (dev/small scale) and cloud storage (S3/OSS for production)
- Configured via `FILESYSTEM_DISK` in `.env`
- Images organized by hash sharding: `images/{hash[0:2]}/{hash[2:4]}/{hash}.{ext}` (supports millions of files)
- Thumbnails stored alongside: `images/{hash[0:2]}/{hash[2:4]}/{hash}_thumb_{size}.webp`
- Original files always preserved

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
  - L3: CDN (static assets, image files)
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

#### Image Storage at Scale
- **Directory sharding**: Don't put millions of files in one directory. Use hash-based sharding: `images/{hash[0:2]}/{hash[2:4]}/{hash}.{ext}` (creates ~65K subdirectories)
- **CDN**: Production must use CDN (CloudFlare, AWS CloudFront, or Alibaba CDN) for image delivery
- **Thumbnail pre-generation**: Generate all thumbnail sizes at upload time, not on-demand
- **Storage cleanup**: Scheduled artisan command to remove orphaned files

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
| source_url | VARCHAR(2048) NULL | Original source URL if imported |
| view_count | INT UNSIGNED DEFAULT 0 | Total views |
| download_count | INT UNSIGNED DEFAULT 0 | Total downloads |
| ai_processed | BOOLEAN DEFAULT FALSE | Whether AI recognition has run |
| created_at | TIMESTAMP | Upload time |
| updated_at | TIMESTAMP | Last update time |

**Key indexes on `images`**:
- `idx_images_format_created` (format, created_at)
- `idx_images_created_at` (created_at)
- `idx_images_view_count` (view_count DESC)
- `idx_images_uuid` (uuid) — UNIQUE
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
| image_count | INT UNSIGNED DEFAULT 0 | Cached count |

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
4. **Storage abstraction**: Use Laravel's Filesystem abstraction. Swap between local and S3/OSS by changing one `.env` variable.
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

FILESYSTEM_DISK=local          # 'local' or 's3'
AWS_ACCESS_KEY_ID=             # For S3/OSS storage
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=
AWS_BUCKET=

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
