# Seagrass Sri Lanka — Backend (Django REST API)

A Django REST API powering the Seagrass Sri Lanka platform. It provides authentication (JWT), content features (blogs, news, research articles, gallery), media uploads, an e‑commerce flow (products, cart, orders, payments), and admin operations.

---

## Tech Stack

- Django 5.2 (Django REST Framework)
- Simple JWT (stateless auth)
- PostgreSQL (psycopg2‑binary)
- CORS Headers, WhiteNoise (static files in production)
- Google API (Calendar integration)
- Gunicorn (production WSGI)

## Repo Structure

```
backend/            Django project (settings, URLs, WSGI/ASGI)
users/              Custom user model + auth (JWT)
blogs/              Blogs, comments, likes
products/           Products, images, cart, payments
order/              Orders + order items
news/               News + Google Calendar integration
research/           Research article links
core/               Gallery images + emailing stub
media/              Generic media assets
admin_actions/      Admin utilities (cross‑app)
images/             Media storage root (served at /images/ in DEBUG)
requirements.txt    Python dependencies
Procfile            Gunicorn process file (see note below)
```

## Requirements

- Python 3.10+
- PostgreSQL 13+
- PowerShell (Windows) or any shell
- Git

## Quickstart (Windows PowerShell)

1) Create and activate a virtual environment

```powershell
python -m venv venv
.\venv\Scripts\Activate
```

2) Install dependencies

```powershell
pip install -r requirements.txt
```

3) Configure PostgreSQL

- Create a DB and user (update names/passwords as you prefer):

```sql
-- In psql as a superuser (e.g. postgres)
CREATE DATABASE seagrass_db;
CREATE USER seagrass_admin WITH PASSWORD 'REPLACE_ME';
GRANT ALL PRIVILEGES ON DATABASE seagrass_db TO seagrass_admin;
```

- Optionally restore from the included backup (may conflict with migrations — prefer running migrations on a fresh DB):

```powershell
pg_restore -U seagrass_admin -d seagrass_db -v "D:\\path\\to\\seagrass_db.backup"
```

4) Environment variables (recommended)

Configure these before production. The current code uses hardcoded defaults — change `backend/settings.py` to read from environment variables.

```
DJANGO_SECRET_KEY=your-secret
DJANGO_DEBUG=True
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DB_NAME=seagrass_db
DB_USER=seagrass_admin
DB_PASSWORD=REPLACE_ME
DB_HOST=localhost
DB_PORT=5432
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://127.0.0.1:3000,http://localhost:5173,http://127.0.0.1:5173
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://localhost:8000/api/auth/google/callback/
PAYHERE_MERCHANT_ID=...
PAYHERE_MERCHANT_SECRET=...
```

5) Apply migrations and create a superuser

```powershell
python manage.py migrate
python manage.py createsuperuser
```

6) Run the development server

```powershell
python manage.py runserver
```

- Base URL: `http://localhost:8000/`
- Media files (DEBUG=True): served from `/images/` (root folder `images/`)

## Settings Overview

Key defaults in `backend/settings.py`:

- `DEBUG=False` (set `True` for local dev)
- `ALLOWED_HOSTS=['*']` (tighten in production)
- `AUTH_USER_MODEL='users.Users'`
- `REST_FRAMEWORK` defaults to `IsAuthenticated`; specific endpoints allow anonymous access
- `SIMPLE_JWT` configured with 60‑min access tokens and 7‑day refresh
- `MEDIA_URL='/images/'` and `MEDIA_ROOT=BASE_DIR / 'images'`
- `STATIC_URL='static/'` with `STATIC_ROOT=BASE_DIR / 'staticfiles'`

## Apps and Data Model

- `users` — Custom user model (`Users`) using email as username; fields include `user_id`, `fname`, `lname`, `image`, `is_staff`. JWT authentication.
- `blogs` — `Blog`, `Comments`, `Likes` with basic like/comment features and optional image per blog.
- `products` — `Product` (with `ProductImage`), `Cart`/`CartItem`, and `Payment` records. E‑commerce primitives and PayHere payment flow.
- `order` — `Order` and `OrderItem` linked to `Payment` and `Product`.
- `news` — `News` with publication flags and image; also exposes Google Calendar endpoints.
- `research` — `Research_articles` storing external links with metadata.
- `core` — `Gallery_images` CRUD by admins; public listing and detail; `emailing` stub endpoint.
- `media` — Generic `Media` entity tied to users/admins.
- `admin_actions` — `Admin` model used to map staff users to admin operations across apps.

## Authentication

- JWT via `djangorestframework-simplejwt`
- Global default permission: `IsAuthenticated`
- Obtain tokens:
  - `POST /api/auth/token/` — returns `access` and `refresh`
  - `POST /api/auth/token/refresh/`
  - `POST /api/auth/token/blacklist/` — logout (send `refresh_token`)
- User flows:
  - `POST /api/auth/register/`
  - `POST /api/auth/login/`
  - `GET  /api/auth/profile/`
  - `PUT|PATCH /api/auth/profile/update/<user_id>/`
  - `POST /api/auth/profile/image/upload/`
  - `DELETE /api/auth/profile/image/delete/`
  - `POST /api/auth/profile/change_password/<user_id>/`

Use `Authorization: Bearer <access>` for protected endpoints.

### Sample: Register and Login

```bash
curl -X POST http://localhost:8000/api/auth/register/ \
  -H "Content-Type: application/json" \
  -d '{"fname":"Ada","lname":"Lovelace","email":"ada@example.com","password":"Passw0rd!"}'

curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"email":"ada@example.com","password":"Passw0rd!"}'
```

## API Overview

Base prefixes are configured in `backend/urls.py`.

- Users (`/api/auth/`)
  - Register/Login/Profile endpoints (see above)
  - Admin: list/update/delete/toggle active; create admin user
- Blogs (`/api/blogs/`)
  - Public: `GET /` list, `GET /<blog_id>/` detail
  - Create/Modify: `POST /post/`, `PUT|PATCH /<blog_id>/update/`, `DELETE /<blog_id>/delete/`
  - Interactions: `POST /<blog_id>/like/`, `POST /create/comment/`, `POST /comments/<comment_id>/reply/`, `DELETE /user/delete_comment/<comment_id>/`
  - Filters: `GET /user/<user_id>/`, `GET /blogs/user/<user_id>/`, `GET /user/blog_comments/<user_id>/`
  - Admin: `GET /admin/adminposts/`, `GET /admin/userposts/`, `POST /admin/post/`, `GET /admin/<blog_id>/`, `PUT|PATCH /admin/<blog_id>/update/`, `DELETE /admin/<blog_id>/delete/`
- Products (`/api/products/`)
  - Public: `GET /list/`, `GET /view_products/<product_id>/`
  - Cart: `POST /cart/add/`, `GET /cart_items/`, `PUT /cart/update_item_count/<product_id>/`, `DELETE /cart/remove_cart_item/<product_id>/`
  - Checkout: `POST /buy-products/` (pre‑checkout), `POST /payment/process/`, `POST /payment/create_payment/`, `POST /payment/payment_notify` (PayHere callback), `POST /payment/save_payment/`
  - Admin: `GET /admin/list/`, `POST /admin/add/`, `PUT|PATCH /admin/<product_id>/update/`, `DELETE /admin/<product_id>/delete/`
- Orders (`/api/order/`)
  - User: `GET /my-orders/`
  - Admin: `GET /admin/list/`, `PUT|PATCH /admin/<order_id>/update/`
- News (`/api/news/`)
  - Public: `GET /` list, `GET /<news_id>/` detail
  - Admin: `POST /admin/add/`, `GET /admin/list/`, `PUT|PATCH /admin/<news_id>/update/`, `DELETE /admin/<news_id>/delete/`
  - Google Calendar: `POST /api/auth/google/login/`, `GET /api/auth/google/callback/`, `GET /api/calendar/events/` (note: paths include `api/` twice under news; keep as‑is or normalize in code)
- Research (`/api/research/`)
  - Public: `GET /list/`, `GET /search/`
  - Admin: `POST /admin/add/`, `GET /admin/list/`, `PUT|PATCH /admin/<research_id>/update/`, `DELETE /admin/<research_id>/delete/`
- Core (`/api/core/`)
  - Public Gallery: `GET /gallery/`, `GET /gallery/<image_id>/`
  - Admin Gallery: `POST /admin/gallery/upload/`, `GET /admin/gallery/my-images/`, `PUT|PATCH /admin/gallery/<image_id>/update/`, `DELETE /admin/gallery/<image_id>/delete/`
  - Misc: `POST /emailing/`
- Media (`/api/media/`)
  - `POST /upload/`, `GET /gallery/`, `GET|POST /gallery/manage/`
- Admin Actions (`/api/admin/`)
  - `POST /products/add/`, `POST /news/add/`, `POST /research/add/`, `POST /gallery/manage/`, `DELETE /delete/<model_name>/<id_value>/`, `POST /create_admin_user/`

### Sample: Product list and add to cart

```bash
curl http://localhost:8000/api/products/list/

curl -X POST http://localhost:8000/api/products/cart/add/ \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"product_id":"<ID>","count":1}'
```

## File Uploads

- Upload size limit: 5 MB (`FILE_UPLOAD_MAX_MEMORY_SIZE`)
- Allowed image extensions: `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`
- Media served at `/images/` during development (`MEDIA_ROOT=images/`)

## CORS

- `CORS_ALLOW_ALL_ORIGINS=True` (broad in development)
- Explicit allowed origins include `http://localhost:3000`, `http://localhost:5173`, etc.

## Payments (PayHere)

- `products.views.create_payment` prepares a signed payload for PayHere checkout (sandbox enabled)
- `products.views.payment_notify` validates PayHere callbacks via MD5 hash
- `products.views.save_payment` creates `Payment`, `Order`, and `OrderItem` records and clears the cart
- Replace hardcoded merchant credentials with environment variables before going live

## Deployment Notes

- Static files: collect using `python manage.py collectstatic` and serve via WhiteNoise or your web server
- WSGI: The included `Procfile` uses `Seagrass_SriLanka_Backend.wsgi:application` but the actual module is `backend.wsgi:application`. Use:

```bash
gunicorn backend.wsgi:application
```

- Set `DEBUG=False` and configure `ALLOWED_HOSTS`
- Configure database via environment variables
- Ensure media files are served by your platform (e.g., S3, Azure Blob, or web server)

## Testing

Basic test execution (if tests are implemented):

```powershell
python manage.py test
```

## Troubleshooting

- 401/403 errors: Most endpoints require JWT. Obtain an access token and pass `Authorization: Bearer <token>`.
- Admin actions forbidden: Ensure your user has `is_staff=True`. Some flows also create/remove the corresponding `admin_actions.Admin` entry.
- Media 404 in production: Configure a proper media server or storage. Django does not serve media with `DEBUG=False`.
- JWT invalid signature: Make sure the same `SECRET_KEY` is used everywhere issuing/verifying tokens.
- Migration conflicts after DB restore: Prefer running migrations on a fresh DB instead of restoring the backup.

## License

This project includes a `LICENSE` file. See its contents for details.

---

### Security Checklist (Before Production)

- Move all secrets (SECRET_KEY, DB creds, Google OAuth, payment secrets) to environment variables
- Tighten `ALLOWED_HOSTS` and CORS origins
- Rotate all credentials currently checked into the repository
- Consider enabling HTTPS and secure cookies
- Add logging/monitoring and error reporting
