# Strava .fit File Uploader — Design Doc

**Date:** 2026-03-07

## Summary

A single `index.html` file hosted on GitHub Pages that allows users to authenticate with Strava and batch-upload `.fit` files. No backend, no build step, no dependencies. Works on any platform with a browser (Mac, iOS, Android).

## Architecture

Single self-contained HTML file with inline CSS and JavaScript. Hosted on GitHub Pages (free, provides HTTPS domain required for OAuth). All state stored in `localStorage`.

## Authentication

- Strava OAuth 2.0 authorization code flow with PKCE
- Scopes: `activity:write`
- Redirect URI: GitHub Pages URL (e.g. `https://username.github.io/strava-helper/`)
- Tokens (access + refresh + expiry) stored in `localStorage`
- Auto-refresh on expiry before each upload
- On refresh failure: clear tokens, prompt reconnect

## UI States

**1. Not connected**
- "Connect to Strava" button
- Brief description

**2. Connected**
- Connected status + "Disconnect" link
- Drag-and-drop zone for `.fit` files (also clickable)
- Selected file list
- "Upload" button

**3. Uploading**
- Per-file status: pending / uploading / processing / done / error
- Overall counter (e.g. "3 of 7 uploaded")
- Inline error messages per file
- "Upload more" button when complete

## Data Flow

### OAuth
1. Click "Connect" → redirect to `https://www.strava.com/oauth/authorize` with `client_id`, `redirect_uri`, `scope`, PKCE `code_challenge`
2. Strava redirects back with `?code=...`
3. Exchange code for tokens via `POST https://www.strava.com/oauth/token`
4. Save tokens to `localStorage`, remove `code` from URL via `history.replaceState`

### Upload (per file, sequential)
1. `POST /api/v3/uploads` — `multipart/form-data` with file and `data_type=fit`
2. Receive `upload_id`
3. Poll `GET /api/v3/uploads/{upload_id}` every 2 seconds
4. On `"Your activity is ready."` → mark done, show activity link
5. On `error` field set → mark failed, show error inline

## Error Handling

| Error | Display |
|---|---|
| Duplicate activity | "Already uploaded" + link to existing activity |
| Invalid .fit file | "Invalid file" |
| Network failure | "Network error, try again" |
| Token refresh failure | "Session expired, reconnect" + clear tokens |
| Rate limit (429) | "Too many requests, wait a moment" |

## Deployment

1. Create GitHub repo, push `index.html`
2. Enable GitHub Pages (Settings → Pages → Deploy from branch)
3. Register a Strava API application at https://www.strava.com/settings/api
4. Set redirect URI to GitHub Pages URL
5. Add `client_id` and `client_secret` to the HTML file

## Out of Scope

- Activity metadata editing (name, type, description)
- Parallel uploads
- Activity history / listing
- Any backend or server component
