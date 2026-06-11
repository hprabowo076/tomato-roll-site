# Tomato Roll

Tomato Roll is a content-driven website built with [Payload CMS](https://payloadcms.com) and [Next.js](https://nextjs.org). It provides a full-featured admin panel, layout builder, publication workflow, and a production-ready frontend — all backed by PostgreSQL.

## Tech Stack

- **CMS**: Payload 3.85
- **Framework**: Next.js 16 (App Router)
- **Database**: PostgreSQL (via `@payloadcms/db-postgres`)
- **Runtime**: Node.js 22+
- **Package Manager**: pnpm >=11
- **Styling**: TailwindCSS 4 + shadcn/ui
- **Rich Text**: Lexical editor
- **Language**: TypeScript

## Features

- **Layout Builder** — Build unique page layouts using pre-configured blocks (Hero, Content, Media, Call To Action, Archive, Form)
- **Draft & Live Preview** — Preview unpublished content before it goes live
- **On-demand Revalidation** — Automatic cache invalidation when content changes
- **SEO** — Full SEO control from the admin panel via the Payload SEO Plugin
- **Search** — Server-side search powered by the Payload Search Plugin
- **Redirects** — Manage URL redirects from the admin panel
- **Forms** — Build and embed forms with the Form Builder Plugin
- **Scheduled Publishing** — Publish or unpublish content on a schedule via the Jobs queue
- **Access Control** — Role-based access for published vs. draft content
- **Dark Mode** — Built-in dark mode support

## Collections

| Collection | Description |
|---|---|
| **Pages** | Layout-builder enabled, draft-enabled pages |
| **Posts** | Blog posts with categories, authors, and layout blocks |
| **Media** | Uploads with pre-configured sizes, focal points, and resizing |
| **Categories** | Nested taxonomy for grouping posts |
| **Users** | Auth-enabled admin and content editors |
| **Forms** | Custom form builder |
| **Form Submissions** | Collected form data |
| **Redirects** | URL redirect management |
| **Search** | Indexed search documents |

## Globals

| Global | Description |
|---|---|
| **Header** | Navigation links and header configuration |
| **Footer** | Footer navigation links |

## Local Development

### Prerequisites

- Node.js 22+
- pnpm >=11
- Docker (for PostgreSQL)

### 1. Clone & Install

```bash
git clone https://github.com/hprabowo076/tomato-roll-site.git
cd tomato-roll-site
cp .env.example .env
pnpm install
```

### 2. Start PostgreSQL

```bash
docker compose -f docker-compose.dev.yml up -d postgres
```

Wait for the healthcheck to pass:

```bash
docker compose -f docker-compose.dev.yml ps
```

### 3. Run the Dev Server

```bash
pnpm dev
```

Open `http://localhost:3000` for the website, or `http://localhost:3000/admin` for the admin panel.

### Docker Dev (Alternative)

Run everything together with the dev compose file:

```bash
docker compose -f docker-compose.dev.yml up
```

This starts PostgreSQL and the Payload app with hot-reload via `pnpm dev`.

## Database

This project uses PostgreSQL via the `@payloadcms/db-postgres` adapter.

- **Development**: Schema changes are applied automatically (`push: true`).
- **Production**: Use migrations to apply schema changes safely.

### Migrations

Create a migration:

```bash
pnpm payload migrate:create
```

Run migrations (required before starting in production):

```bash
pnpm payload migrate
```

## Production Deployment

The production stack runs three containers: **app** (Next.js standalone), **postgres**, and **cloudflared** (Cloudflare Tunnel). Migrations run automatically at startup via `prodMigrations` in the database adapter config — no manual migration step needed.

### 1. Configure Environment

```bash
cp .env.example .env
```

Edit `.env` and replace all placeholder values:

| Variable | How to generate |
|---|---|
| `PAYLOAD_SECRET` | `openssl rand -hex 16` |
| `CRON_SECRET` | `openssl rand -hex 16` |
| `PREVIEW_SECRET` | `openssl rand -hex 16` |
| `POSTGRES_PASSWORD` | Set a strong password |
| `NEXT_PUBLIC_SERVER_URL` | Your Cloudflare Tunnel public URL (e.g. `https://tomatoroll.cloud`) |
| `CLOUDFLARED_TOKEN` | From Cloudflare Zero Trust dashboard (see below) |
| `BUILD_DATABASE_URL` | Connection string to reach PostgreSQL from the Docker builder (e.g. `postgres://postgres:password@host.docker.internal:5432/tomato-roll`) |

### 2. Set Up Cloudflare Tunnel

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) > Networks > Tunnels
2. Create a new tunnel, name it (e.g. `tomato-roll`)
3. Note the tunnel token — this goes in `CLOUDFLARED_TOKEN`
4. Configure the tunnel's public hostname to point to `http://app:3000`

### 3. Build & Deploy

The build requires a running PostgreSQL instance to generate static pages. Set `BUILD_DATABASE_URL` to point to your PostgreSQL from the Docker builder (use `host.docker.internal` on Docker Desktop):

```bash
export BUILD_DATABASE_URL="postgres://postgres:yourpassword@host.docker.internal:5432/tomato-roll"
docker compose up -d --build
```

This builds the app image (using Docker Build Secrets for `PAYLOAD_SECRET`), starts PostgreSQL, runs the app on port 3000, and connects the Cloudflare Tunnel. Migrations are applied automatically when the app starts. The app is accessible at your tunnel domain.

> **Build secrets**: `PAYLOAD_SECRET` is passed as a Docker Build Secret (not ARG/ENV) to avoid leaking into the image layers. `DATABASE_URL` is passed as a build ARG since it is only a temporary connection string used during build, not persisted in the runner image.

### 4. Create First Admin User

Visit `https://your-domain.com/admin` and create the first admin user through the Payload admin panel.

### Cloudflared UDP Buffer Warning

Cloudflared may log a warning about UDP receive buffer size:

```
failed to sufficiently increase receive buffer size (was: 208 kiB, wanted: 7168 kiB, got: 416 kiB)
```

This is non-critical but affects QUIC tunnel performance. On Linux hosts, fix it at the OS level:

```bash
sudo sysctl -w net.core.rmem_max=7500000
```

On Docker Desktop (macOS/Windows), this cannot be set per-container and can be safely ignored.

### Managing the Stack

```bash
# View logs
docker compose logs -f app
docker compose logs -f cloudflared

# Restart
docker compose restart

# Stop
docker compose down

# Stop and remove data
docker compose down -v
```

## Environment Variables

| Variable | Description | Required |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string (used at runtime) | Yes |
| `PAYLOAD_SECRET` | Secret used to sign JWT tokens (injected via Docker Build Secret) | Yes |
| `NEXT_PUBLIC_SERVER_URL` | Public URL of the app (no trailing slash) | Yes |
| `CRON_SECRET` | Secret to authenticate cron job requests | Yes |
| `PREVIEW_SECRET` | Secret to validate draft preview requests | Yes |
| `BUILD_DATABASE_URL` | PostgreSQL connection string used during `docker build` | Build only |
| `CLOUDFLARED_TOKEN` | Cloudflare Tunnel token | Prod only |
| `POSTGRES_PASSWORD` | PostgreSQL password (used by compose) | Prod only |

## Project Structure

```
src/
├── Header/          # Header global config
├── Footer/          # Footer global config
├── access/          # Access control functions
├── app/             # Next.js App Router pages
├── blocks/          # Layout builder blocks
├── collections/     # Payload collections
├── components/      # Shared React components
├── endpoints/       # Custom API endpoints & seed data
├── fields/          # Reusable field configs
├── heros/           # Hero block variants
├── hooks/           # Collection/global hooks
├── plugins/         # Payload plugin configs
├── providers/       # React context providers
├── search/          # Search functionality
└── utilities/       # Helper functions
```

## License

MIT
