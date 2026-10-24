# ScamShield AI

ScamShield AI is a small full-stack starter with an Express API and a React app powered by Vite.

## Requirements

- Node.js 20 or newer
- npm

## Install

From the project root, install all dependencies:

```sh
npm install
npm install --prefix server
npm install --prefix client
```

Copy `server/.env.example` to `server/.env` and configure your environment. Message checks use Anthropic when `ANTHROPIC_API_KEY` is set; URL checks are string-only and never open or fetch submitted URLs.

## Run

Start the API and client together:

```sh
npm run dev
```

Open the Vite URL shown in the terminal (normally http://localhost:5173). The page requests `/api/health` through Vite's development proxy and displays the server status.

The API listens on http://localhost:3001. Its health endpoint is http://localhost:3001/api/health.

## Check API

- `POST /api/check` accepts `{ "type": "url" | "message", "content": "..." }` (up to 5,000 characters) and returns the analysis with a check ID.
- `POST /api/check/image` accepts multipart form data with an `image` field and optional `uiLanguage`. PNG, JPEG, and WebP files are limited to 5 MB and 4,000 pixels per side. Images are re-encoded without metadata before Claude vision analysis; images are never saved, and only the first 100 characters of extracted text can be retained in history. The response includes `source: "image"` and a short `extractedPreview`, not the full extracted text.
- `GET /api/history?limit=20` returns recent checks. The limit can be from 1 to 100.
- `GET /api/checks/:id` returns a saved check.

Checks are stored in SQLite in `server/scamshield.sqlite` by default. Only the first 100 characters and a sanitized result summary are stored; complete submitted messages and URLs, images, extracted image text, and free-form AI text are never written to the database. Set `HISTORY_ENABLED=false` to stop saving checks and purge existing history on server startup. `DELETE /api/history` removes all saved checks. `POST /api/check` is limited to 20 requests per minute per IP; image checks are limited to 5 per minute per IP.

Configure `CORS_ORIGIN` to a comma-separated allow-list of exact browser-app origins (for local development, `http://localhost:5173`). Requests from other origins are not granted CORS access. Set `REPORT_SCAM_URL` to an official HTTPS scam-reporting resource; the app displays the link only when configured. Request bodies are not logged.

Run server tests with `npm test --prefix server`.

## Evaluate scam detection

The synthetic labeled dataset lives in `server/test-data/`, including five screenshot fixtures with expected extracted text. Run `npm run evaluate --prefix server` to evaluate URL checks, deterministic message rules, and screenshot text fixtures without making Anthropic API calls. The evaluator prints accuracy, precision, recall, and any misclassified samples; it exits with a failure status when accuracy is below 85%.

## Deployment

### Render or Railway: separate server and static client

Build and run the API from the repository root using `server/Dockerfile`. Configure these server environment variables in the hosting dashboard:

- `PORT`: supplied by the platform.
- `CORS_ORIGIN`: the exact deployed static-client origin, such as `https://scamshield.example`.
- `DATABASE_PATH`: a path on a mounted persistent disk/volume, such as `/var/data/scamshield.sqlite` on Render or `/data/scamshield.sqlite` on Railway.
- `HISTORY_ENABLED`: `true` to retain 100-character previews, or `false` to purge and disable history.
- `REPORT_SCAM_URL`: optional official HTTPS reporting page.
- `ANTHROPIC_API_KEY` and `ANTHROPIC_MODEL`: configure these if using Claude message analysis.

Attach a persistent disk/volume at the parent directory of `DATABASE_PATH`; otherwise SQLite history may be lost when the container is replaced. On Render, create a Docker web service with the repository root as the build context and `server/Dockerfile` as its Dockerfile path. On Railway, deploy the repository root and select `server/Dockerfile`; attach a volume and set `DATABASE_PATH` to a path inside its mount.

For a Render Static Site, use `npm install --prefix client --no-audit --no-fund && npm run build --prefix client` as the build command and `client/dist` as the publish directory. Set `VITE_API_BASE_URL` to the API's public origin (for example, `https://scamshield-api.example`) in the static site's build environment. Add a rewrite from `/*` to `/index.html` with status `200` so direct links to history pages work. On Railway, deploy the client with `client/Dockerfile`; set its build-time `VITE_API_BASE_URL` argument to the same API origin. The client container uses Nginx history fallback.

For local containers, build from the repository root:

```sh
docker build -f server/Dockerfile -t scamshield-server .
docker build -f client/Dockerfile --build-arg VITE_API_BASE_URL=http://localhost:3001 -t scamshield-client .
```

The client image listens on port 80 and the API image listens on the platform-provided `PORT` (3001 by default).

GitHub Actions runs the server tests, synthetic evaluation, and static client build on each push and pull request.

## Chrome extension

Load the unpacked extension from `extension/` in `chrome://extensions` with Developer mode enabled. In the popup, enter the ScamShield API origin; the extension requests host access only to that origin when a check is submitted. The default is `http://localhost:3001` for local development; remote API origins must use HTTPS.

The extension uses the `activeTab` and `contextMenus` permissions and optional host access for the configured API origin. It reads only the active tab URL, never page content. Its context menu opens a link check for the selected URL. To allow the extension to call the API, add its generated Chrome extension ID to the server's CORS allow-list, for example:

```text
CORS_ORIGIN=http://localhost:5173,chrome-extension://abcdefghijklmnopabcdefghijklmnop
```

For a published extension, use its stable extension ID. Restart the server after changing `CORS_ORIGIN`.