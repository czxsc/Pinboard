# Pinterest Moodboard — Project Brief

A Milanote-style visual whiteboard that pulls images directly from your Pinterest boards, so you can arrange, label, group, and connect them on an infinite canvas instead of manually saving and re-uploading images.

---

## 1. Goals

Two goals drive every technical decision here:

1. **Build real skills** in backend development, cloud, APIs, and deployment.
2. **Support a Pinterest SWE application** by deliberately mirroring Pinterest's real tech stack where it's appropriate — and being able to explain *why* each choice was made.

The second goal is why the stack below leans Python / React / MySQL / AWS rather than whatever would be fastest. The point is defensible, interview-ready decisions.

---

## 2. Product concept

A standalone web app (not a browser extension) with an infinite canvas.

**Core MVP features:**
- Infinite canvas with pan and zoom
- Drag-and-drop image placement
- Text labels
- Grouping of items
- Connector lines between items
- Save / load a board (persists between sessions)
- Import images directly from your Pinterest boards

**Deliberately later / optional:**
- A browser-extension "clip this image" companion
- Multi-user sharing
- Export to image / PDF
- More varied whiteboard features

---

## 3. Architecture overview

When someone uses the app, the flow is:

- **Browser (React canvas)** sends requests to →
- **Python backend** (running in a Docker container, managed by an orchestrator on AWS), which talks to →
  - **MySQL** — the board's *arrangement* and account data
  - **S3** — the actual image files
  - **Pinterest API** — external, to fetch boards and pins

The database holds "your stuff" (layout, labels, connectors, references). Pinterest holds the original pins. S3 optionally holds your own copies of the images.

---

## 4. Tech stack (Pinterest-flavored)

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React + TypeScript, with **tldraw** for the canvas | tldraw provides infinite canvas, drag/drop, connectors, grouping, undo/redo out of the box; Pinterest is a React shop |
| Backend language | Python — **Django REST Framework** or **FastAPI** *(decision open)* | Python is Pinterest's primary language; DRF mirrors their framework, FastAPI is the lighter modern option |
| Database | **MySQL** on AWS **RDS** | Deliberately matches Pinterest's core datastore |
| Image storage | AWS **S3** | Exactly what Pinterest does for images |
| Containers | **Docker** | Standard everywhere |
| Orchestration | Start on **ECS/Fargate**, graduate to **EKS (Kubernetes)** | EKS matches Pinterest; ECS is a gentler starting point |
| CI/CD | **GitHub Actions** → build image → push to **ECR** → deploy | Mirrors the concept of Pinterest's internal pipeline |
| Infra as code | **Terraform** | Reproducible, reviewable infrastructure |
| Async jobs | **Celery / Redis** (for board-import image processing) | Right-sized stand-in for Pinterest's Kafka/Flink |
| Observability | **OpenTelemetry** + Prometheus/Grafana *(later)* | Matches Pinterest's OTEL usage |
| Service mesh | **Envoy** *(optional, advanced)* | Matches Pinterest; overkill for one service, worth knowing |
| Auth | A **managed auth provider** (e.g. Supabase Auth, Clerk, Auth0, or Sign in with Google) *(decision open)* | Avoids owning password security |

**Guiding principle — don't cargo-cult the logos.** A single-user moodboard has no real need for Kafka, Flink, or a service mesh. Use right-sized tools, and treat the scale-ups (CDC pipelines, streaming, mesh) as things you can *explain you deliberately didn't build*. That judgment is itself the interview signal.

---

## 5. Key decisions made

- **Standalone web app**, not an extension.
- **Database stores the arrangement, not the images.** Positions, sizes, text labels, connector lines, groups, which Pinterest pin each item references, and user accounts. None of this exists on Pinterest.
- **Images: hotlink first, self-host later.**
  - v1: store the Pinterest image URL and load images straight from Pinterest (simple, no S3, but URLs can break if pins change or go private).
  - Later: download copies into your own S3 bucket so boards are self-contained (more robust; this is what Pinterest itself does).
- **Single-user first.** Staying single-user keeps the Pinterest app in trial mode indefinitely and avoids their app review. Multi-user is a distinct, later phase.
- **Deployment-first development.** Set up the full deploy pipeline while there's almost no code, so each tool is met one at a time.

---

## 6. Pinterest API integration notes

- **API v5**, OAuth 2.0. Read-only access to the *authenticated user's own* boards and pins.
- **Scopes needed:** `boards:read`, `pins:read` (least privilege — no write/ads).
- **Business account** required to register the app at developers.pinterest.com.
- **Trial vs Standard:** new apps are in trial mode — only the app owner's account can connect. Letting *other people* connect requires submitting for Standard access (a video-demo review).
- **Tokens:** refresh tokens are long-lived and refreshable; store per user, encrypted.
- **Caveat to verify:** check Pinterest's developer terms on storing/caching their data before relying on self-hosting images.

**OAuth connection flow (per user):**
1. User clicks "Connect Pinterest" on your site.
2. Redirect to Pinterest's login/consent page (on Pinterest's domain).
3. User logs in and approves.
4. Pinterest redirects back to your registered redirect URI with a short-lived **authorization code**.
5. Your backend exchanges that code + your **app secret** for an **access token** (server-to-server).
6. Store the token per user; use it to fetch their boards. The token never travels through the browser.

---

## 7. Security requirements (for the multi-user phase)

Priority order:

1. **Protect tokens & secrets.** Encrypt tokens at rest; keep them server-side only (never sent to the browser). Keep the app secret in environment variables / a secrets manager — never committed to Git.
2. **Use a managed auth provider** rather than building password handling yourself.
3. **HTTPS everywhere** in production.
4. **OAuth protections:** use the `state` parameter (prevents callback CSRF); request minimal read-only scopes.
5. **Web-app basics:** parameterized queries / ORM (prevents SQL injection); rely on React escaping (prevents XSS); rate-limit endpoints; don't leak stack traces.
6. **Respect user data:** let users disconnect Pinterest and delete their account/data; mind GDPR/CCPA if you have EU/UK/California users; collect the minimum needed.
7. **Infra hygiene:** keep the S3 bucket and database private; least-privilege IAM roles; keep dependencies updated.

---

## 8. Development roadmap

1. **Walking skeleton to production** — trivial app deployed end-to-end through the full pipeline (this is the first prototype; see below).
2. **Canvas MVP** — React + tldraw: drag/drop, text labels, grouping, connectors, pan/zoom.
3. **Persistence + S3** — save canvas state to MySQL; wire up S3.
4. **Pinterest OAuth + board browsing** — auth flow, list boards and pins in a side panel.
5. **The bridge + async ingestion** — import selected pins onto the canvas; background worker for image downloads.
6. **Advanced infra** — migrate ECS→EKS, add observability, optionally Envoy. Also the Pinterest Standard-access review if going multi-user.

---

## 9. Initial data model sketch

Rough starting tables (refine as you go):

- **users** — id, email, created_at (or delegated to the auth provider)
- **canvases** — id, user_id, name, updated_at
- **canvas_items** — id, canvas_id, type (`image` | `text` | `connector` | `group`), x, y, width, height, content/label, and for images: the Pinterest pin reference + image URL (or S3 key once self-hosting)
- **pinterest_connections** — id, user_id, access_token (encrypted), refresh_token (encrypted), expires_at

---

## 10. First prototype — start here

The first prototype is the **walking skeleton**: prove the whole pipeline works end-to-end with almost no features. Suggested steps:

- [ ] **Step 0 (local warm-up):** a Python backend with a single `GET /health` endpoint returning JSON, running inside Docker on your laptop.
- [ ] Add a minimal React + TypeScript frontend that calls `/health` and displays the result.
- [ ] Add a MySQL database locally (via Docker Compose alongside the backend); have the backend confirm it can connect.
- [ ] Put it all in a GitHub repo with a GitHub Actions workflow that builds the Docker image and runs one trivial test on push.
- [ ] Write Terraform to create the AWS pieces: an RDS MySQL instance, an S3 bucket (placeholder), and an ECS/Fargate service.
- [ ] Deploy the skeleton to AWS and confirm: deployed frontend → deployed backend → database, all reachable over HTTPS.

**Definition of done:** you can push a code change and watch it flow automatically to a live URL, where the frontend successfully calls the backend, which successfully reaches the database. No canvas, no Pinterest — just the pipeline.

Only once that's green do you move to Phase 2 (the canvas).

---

## 11. Open decisions to make

- Backend framework: **Django REST Framework** vs **FastAPI**
- Orchestration start point: **ECS/Fargate first** vs straight to **EKS**
- Managed auth provider: Supabase Auth / Clerk / Auth0 / Sign in with Google
- Verify Pinterest developer terms on caching/storing image data
