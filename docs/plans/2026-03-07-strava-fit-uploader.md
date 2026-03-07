# Strava .fit File Uploader Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a single `index.html` file that lets users authenticate with Strava via OAuth and batch-upload `.fit` files, hosted on GitHub Pages.

**Architecture:** Single self-contained HTML file with inline CSS and JS, no build step, no dependencies. Strava OAuth 2.0 authorization code flow with PKCE, tokens in `localStorage`. Files uploaded sequentially with polling to confirm each activity is created.

**Tech Stack:** Vanilla HTML, CSS, JavaScript (ES2020). Strava API v3. GitHub Pages for hosting.

---

## Important Notes Before Starting

- This is a **single file** project — everything lives in `index.html`
- No npm, no bundler, no framework
- Testing is done manually in the browser (open the file, interact with UI)
- You will need a Strava API app registered at https://www.strava.com/settings/api — do this before Task 3
- The `client_secret` is embedded in the HTML — this is acceptable for a personal tool but means the file should not be shared publicly with your real credentials

---

### Task 1: HTML skeleton and CSS

**Files:**
- Create: `index.html`

**Step 1: Create the HTML file with three UI states**

Create `index.html` with this exact content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Strava .fit Uploader</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: #f5f5f5;
      color: #333;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 1rem;
    }

    .card {
      background: white;
      border-radius: 12px;
      padding: 2rem;
      width: 100%;
      max-width: 540px;
      box-shadow: 0 2px 12px rgba(0,0,0,0.1);
    }

    h1 { font-size: 1.4rem; margin-bottom: 0.5rem; }
    p.subtitle { color: #666; font-size: 0.9rem; margin-bottom: 1.5rem; }

    /* Connect state */
    #state-connect { display: none; }
    #state-connect.active { display: block; }

    .btn-strava {
      display: inline-flex;
      align-items: center;
      gap: 0.5rem;
      background: #fc4c02;
      color: white;
      border: none;
      border-radius: 6px;
      padding: 0.75rem 1.5rem;
      font-size: 1rem;
      font-weight: 600;
      cursor: pointer;
      text-decoration: none;
      transition: background 0.2s;
    }
    .btn-strava:hover { background: #e04300; }

    /* Upload state */
    #state-upload { display: none; }
    #state-upload.active { display: block; }

    .header-bar {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 1.5rem;
      font-size: 0.9rem;
    }
    .connected-label { color: #28a745; font-weight: 600; }
    .btn-link {
      background: none;
      border: none;
      color: #666;
      cursor: pointer;
      font-size: 0.85rem;
      text-decoration: underline;
    }
    .btn-link:hover { color: #333; }

    .drop-zone {
      border: 2px dashed #ccc;
      border-radius: 8px;
      padding: 2.5rem 1rem;
      text-align: center;
      cursor: pointer;
      transition: border-color 0.2s, background 0.2s;
      margin-bottom: 1rem;
    }
    .drop-zone:hover, .drop-zone.drag-over {
      border-color: #fc4c02;
      background: #fff5f0;
    }
    .drop-zone p { color: #888; font-size: 0.9rem; margin-top: 0.4rem; }
    .drop-icon { font-size: 2rem; }

    #file-input { display: none; }

    #file-list { margin-bottom: 1rem; }
    .file-item {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 0.5rem 0;
      border-bottom: 1px solid #f0f0f0;
      font-size: 0.875rem;
    }
    .file-name { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
    .file-status { margin-left: 1rem; font-size: 0.8rem; white-space: nowrap; }
    .status-pending { color: #888; }
    .status-uploading { color: #007bff; }
    .status-processing { color: #fd7e14; }
    .status-done { color: #28a745; }
    .status-error { color: #dc3545; }
    .status-duplicate { color: #6f42c1; }

    .btn-primary {
      display: block;
      width: 100%;
      background: #fc4c02;
      color: white;
      border: none;
      border-radius: 6px;
      padding: 0.75rem;
      font-size: 1rem;
      font-weight: 600;
      cursor: pointer;
      transition: background 0.2s;
    }
    .btn-primary:hover:not(:disabled) { background: #e04300; }
    .btn-primary:disabled { background: #ccc; cursor: not-allowed; }

    .progress-summary {
      text-align: center;
      font-size: 0.85rem;
      color: #666;
      margin-bottom: 0.75rem;
    }
  </style>
</head>
<body>
  <div class="card">
    <!-- State: not connected -->
    <div id="state-connect">
      <h1>Strava .fit Uploader</h1>
      <p class="subtitle">Upload .fit files from any device directly to your Strava account.</p>
      <button class="btn-strava" id="btn-connect">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
          <path d="M15.387 17.944l-2.089-4.116h-3.065L15.387 24l5.15-10.172h-3.066m-7.008-5.599l2.836 5.598h4.172L10.463 0l-7 13.828h4.169"/>
        </svg>
        Connect to Strava
      </button>
    </div>

    <!-- State: connected / uploading -->
    <div id="state-upload">
      <div class="header-bar">
        <span class="connected-label">Connected to Strava</span>
        <button class="btn-link" id="btn-disconnect">Disconnect</button>
      </div>

      <div class="drop-zone" id="drop-zone">
        <div class="drop-icon">📂</div>
        <strong>Drop .fit files here</strong>
        <p>or click to select files</p>
      </div>
      <input type="file" id="file-input" accept=".fit" multiple>

      <div id="file-list"></div>
      <p class="progress-summary" id="progress-summary"></p>
      <button class="btn-primary" id="btn-upload" disabled>Upload</button>
    </div>
  </div>

  <script>
    // Config — fill these in before deploying
    const CLIENT_ID = 'YOUR_CLIENT_ID';
    const CLIENT_SECRET = 'YOUR_CLIENT_SECRET';
    const REDIRECT_URI = window.location.origin + window.location.pathname;

    // --- State ---
    let files = []; // Array of { file, status, message, activityId }

    // --- Auth helpers ---
    function getToken() { return localStorage.getItem('strava_access_token'); }
    function getRefreshToken() { return localStorage.getItem('strava_refresh_token'); }
    function getExpiry() { return parseInt(localStorage.getItem('strava_token_expiry') || '0'); }
    function saveTokens(data) {
      localStorage.setItem('strava_access_token', data.access_token);
      localStorage.setItem('strava_refresh_token', data.refresh_token);
      localStorage.setItem('strava_token_expiry', data.expires_at);
    }
    function clearTokens() {
      localStorage.removeItem('strava_access_token');
      localStorage.removeItem('strava_refresh_token');
      localStorage.removeItem('strava_token_expiry');
    }
    function isTokenValid() {
      return getToken() && getExpiry() > Math.floor(Date.now() / 1000) + 60;
    }

    // --- PKCE helpers ---
    function randomBase64url(len) {
      const buf = new Uint8Array(len);
      crypto.getRandomValues(buf);
      return btoa(String.fromCharCode(...buf)).replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
    }
    async function sha256base64url(plain) {
      const enc = new TextEncoder().encode(plain);
      const hash = await crypto.subtle.digest('SHA-256', enc);
      return btoa(String.fromCharCode(...new Uint8Array(hash))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
    }

    // --- UI ---
    function showState(name) {
      document.getElementById('state-connect').classList.toggle('active', name === 'connect');
      document.getElementById('state-upload').classList.toggle('active', name === 'upload');
    }

    function renderFiles() {
      const list = document.getElementById('file-list');
      list.innerHTML = files.map((f, i) => {
        let statusHtml = '';
        if (f.status === 'pending')     statusHtml = '<span class="file-status status-pending">Pending</span>';
        if (f.status === 'uploading')   statusHtml = '<span class="file-status status-uploading">Uploading...</span>';
        if (f.status === 'processing')  statusHtml = '<span class="file-status status-processing">Processing...</span>';
        if (f.status === 'done')        statusHtml = `<span class="file-status status-done"><a href="https://www.strava.com/activities/${f.activityId}" target="_blank">Done</a></span>`;
        if (f.status === 'duplicate')   statusHtml = `<span class="file-status status-duplicate"><a href="https://www.strava.com/activities/${f.activityId}" target="_blank">Already uploaded</a></span>`;
        if (f.status === 'error')       statusHtml = `<span class="file-status status-error">${f.message || 'Error'}</span>`;
        return `<div class="file-item"><span class="file-name" title="${f.file.name}">${f.file.name}</span>${statusHtml}</div>`;
      }).join('');

      const done = files.filter(f => f.status === 'done' || f.status === 'duplicate').length;
      const errored = files.filter(f => f.status === 'error').length;
      const summary = document.getElementById('progress-summary');
      if (files.length === 0) {
        summary.textContent = '';
      } else {
        summary.textContent = `${done} of ${files.length} uploaded${errored ? `, ${errored} failed` : ''}`;
      }
    }

    function updateUploadButton() {
      const btn = document.getElementById('btn-upload');
      const allDone = files.length > 0 && files.every(f => ['done','duplicate','error'].includes(f.status));
      if (allDone) {
        btn.textContent = 'Upload more';
        btn.disabled = false;
        btn.onclick = resetForMore;
      } else {
        btn.textContent = 'Upload';
        btn.disabled = files.length === 0;
        btn.onclick = startUpload;
      }
    }

    function resetForMore() {
      files = [];
      document.getElementById('file-input').value = '';
      renderFiles();
      updateUploadButton();
    }

    // --- OAuth ---
    async function connectStrava() {
      const verifier = randomBase64url(32);
      const challenge = await sha256base64url(verifier);
      sessionStorage.setItem('pkce_verifier', verifier);
      const params = new URLSearchParams({
        client_id: CLIENT_ID,
        redirect_uri: REDIRECT_URI,
        response_type: 'code',
        approval_prompt: 'auto',
        scope: 'activity:write',
        code_challenge: challenge,
        code_challenge_method: 'S256',
      });
      window.location.href = 'https://www.strava.com/oauth/authorize?' + params;
    }

    async function handleOAuthCallback(code) {
      const verifier = sessionStorage.getItem('pkce_verifier');
      sessionStorage.removeItem('pkce_verifier');
      history.replaceState(null, '', window.location.pathname);
      const res = await fetch('https://www.strava.com/oauth/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          client_id: CLIENT_ID,
          client_secret: CLIENT_SECRET,
          code,
          grant_type: 'authorization_code',
          code_verifier: verifier,
        }),
      });
      if (!res.ok) throw new Error('Token exchange failed');
      saveTokens(await res.json());
    }

    async function refreshToken() {
      const res = await fetch('https://www.strava.com/oauth/token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          client_id: CLIENT_ID,
          client_secret: CLIENT_SECRET,
          refresh_token: getRefreshToken(),
          grant_type: 'refresh_token',
        }),
      });
      if (!res.ok) throw new Error('Refresh failed');
      saveTokens(await res.json());
    }

    async function ensureValidToken() {
      if (!isTokenValid()) {
        await refreshToken();
      }
    }

    function disconnect() {
      clearTokens();
      files = [];
      renderFiles();
      updateUploadButton();
      showState('connect');
    }

    // --- Upload ---
    async function uploadFile(entry) {
      entry.status = 'uploading';
      renderFiles();

      try {
        await ensureValidToken();
      } catch {
        clearTokens();
        showState('connect');
        throw new Error('Session expired');
      }

      const form = new FormData();
      form.append('file', entry.file);
      form.append('data_type', 'fit');

      let uploadRes;
      try {
        uploadRes = await fetch('https://www.strava.com/api/v3/uploads', {
          method: 'POST',
          headers: { Authorization: 'Bearer ' + getToken() },
          body: form,
        });
      } catch {
        entry.status = 'error';
        entry.message = 'Network error, try again';
        return;
      }

      if (uploadRes.status === 429) {
        entry.status = 'error';
        entry.message = 'Too many requests, wait a moment';
        return;
      }
      if (!uploadRes.ok) {
        entry.status = 'error';
        entry.message = 'Upload failed';
        return;
      }

      const { id: uploadId } = await uploadRes.json();

      // Poll for result
      entry.status = 'processing';
      renderFiles();

      for (let attempts = 0; attempts < 30; attempts++) {
        await new Promise(r => setTimeout(r, 2000));
        let pollRes;
        try {
          pollRes = await fetch(`https://www.strava.com/api/v3/uploads/${uploadId}`, {
            headers: { Authorization: 'Bearer ' + getToken() },
          });
        } catch {
          entry.status = 'error';
          entry.message = 'Network error while checking status';
          return;
        }
        const data = await pollRes.json();
        if (data.error) {
          if (data.error.includes('duplicate')) {
            entry.status = 'duplicate';
            const match = data.error.match(/(\d+)/);
            entry.activityId = match ? match[1] : null;
          } else {
            entry.status = 'error';
            entry.message = data.error;
          }
          return;
        }
        if (data.activity_id) {
          entry.status = 'done';
          entry.activityId = data.activity_id;
          return;
        }
        // status is still "Your activity is being processed." — keep polling
      }

      entry.status = 'error';
      entry.message = 'Timed out waiting for Strava';
    }

    async function startUpload() {
      document.getElementById('btn-upload').disabled = true;
      for (const entry of files) {
        if (entry.status !== 'pending') continue;
        await uploadFile(entry);
        renderFiles();
      }
      updateUploadButton();
    }

    // --- File selection ---
    function addFiles(newFiles) {
      for (const f of newFiles) {
        if (f.name.toLowerCase().endsWith('.fit')) {
          files.push({ file: f, status: 'pending', message: '', activityId: null });
        }
      }
      renderFiles();
      updateUploadButton();
    }

    // --- Init ---
    async function init() {
      const params = new URLSearchParams(window.location.search);
      const code = params.get('code');

      if (code) {
        try {
          await handleOAuthCallback(code);
        } catch {
          alert('Failed to connect to Strava. Please try again.');
          showState('connect');
          return;
        }
      }

      if (getToken()) {
        showState('upload');
      } else {
        showState('connect');
      }

      // Event listeners
      document.getElementById('btn-connect').addEventListener('click', connectStrava);
      document.getElementById('btn-disconnect').addEventListener('click', disconnect);

      const dropZone = document.getElementById('drop-zone');
      const fileInput = document.getElementById('file-input');

      dropZone.addEventListener('click', () => fileInput.click());
      fileInput.addEventListener('change', () => addFiles(fileInput.files));

      dropZone.addEventListener('dragover', e => { e.preventDefault(); dropZone.classList.add('drag-over'); });
      dropZone.addEventListener('dragleave', () => dropZone.classList.remove('drag-over'));
      dropZone.addEventListener('drop', e => {
        e.preventDefault();
        dropZone.classList.remove('drag-over');
        addFiles(e.dataTransfer.files);
      });

      document.getElementById('btn-upload').addEventListener('click', startUpload);

      renderFiles();
      updateUploadButton();
    }

    init();
  </script>
</body>
</html>
```

**Step 2: Verify the file renders correctly**

Open `index.html` in a browser (double-click or `open index.html` on Mac).

Expected:
- Page loads with "Strava .fit Uploader" heading
- "Connect to Strava" orange button visible
- No console errors

**Step 3: Commit**

```bash
git init
git add index.html
git commit -m "feat: add strava fit uploader single-page app"
```

---

### Task 2: Register a Strava API app and fill in credentials

**This is a manual step — no code to write.**

**Step 1: Register your Strava API application**

1. Go to https://www.strava.com/settings/api (must be logged into Strava)
2. Fill in the form:
   - **Application Name:** anything (e.g. "My Fit Uploader")
   - **Category:** choose any
   - **Club:** leave blank
   - **Website:** your GitHub Pages URL (e.g. `https://yourusername.github.io/strava-helper/`) — this can be a placeholder for now, update after GitHub Pages is set up
   - **Authorization Callback Domain:** your GitHub Pages domain (e.g. `yourusername.github.io`) — just the domain, no path
3. Submit → you'll see your **Client ID** and **Client Secret**

**Step 2: Fill credentials into index.html**

Edit `index.html` and replace:
```js
const CLIENT_ID = 'YOUR_CLIENT_ID';
const CLIENT_SECRET = 'YOUR_CLIENT_SECRET';
```
with your actual values (Client ID is a number, Client Secret is a long string).

**Step 3: Commit**

```bash
git add index.html
git commit -m "config: add strava client credentials"
```

> **Security note:** This file contains your client secret. Do not share it publicly or post it in a public GitHub repo without understanding the implications. For a personal tool, this is acceptable.

---

### Task 3: Deploy to GitHub Pages

**Step 1: Create a GitHub repository**

1. Go to https://github.com/new
2. Create a new repo (e.g. `strava-helper`)
3. Make it **private** (recommended, since the file contains your client secret) or public

**Step 2: Push to GitHub**

```bash
git remote add origin https://github.com/YOUR_USERNAME/strava-helper.git
git branch -M main
git push -u origin main
```

**Step 3: Enable GitHub Pages**

1. In your repo, go to Settings → Pages
2. Under "Source", select "Deploy from a branch"
3. Branch: `main`, folder: `/ (root)`
4. Click Save
5. Wait ~1 minute → your site is live at `https://YOUR_USERNAME.github.io/strava-helper/`

**Step 4: Update the Strava app redirect URI**

1. Go back to https://www.strava.com/settings/api
2. Update "Authorization Callback Domain" to `YOUR_USERNAME.github.io`
3. Update "Website" to your full GitHub Pages URL

**Step 5: Verify deployment**

Open `https://YOUR_USERNAME.github.io/strava-helper/` in a browser.

Expected: Same page as local, "Connect to Strava" button visible.

---

### Task 4: End-to-end test

**Step 1: Test OAuth flow**

1. Open your GitHub Pages URL
2. Click "Connect to Strava"
3. Strava authorization page opens — click "Authorize"
4. You're redirected back to your page

Expected: Page shows "Connected to Strava" and the drop zone.

**Step 2: Test file upload**

1. Drag a `.fit` file onto the drop zone (or click to select)
2. File appears in the list with "Pending" status
3. Click "Upload"

Expected:
- Status changes to "Uploading..." then "Processing..." then "Done" with a link to the activity
- Clicking the link opens the activity on Strava

**Step 3: Test duplicate detection**

Upload the same `.fit` file again.

Expected: Status shows "Already uploaded" with a link (purple).

**Step 4: Test batch upload**

Select 3+ `.fit` files at once and click Upload.

Expected: Files process one by one, each getting their own status, overall counter updates.

**Step 5: Test disconnect**

Click "Disconnect".

Expected: Returns to "Connect to Strava" screen, tokens cleared from localStorage.

---

### Task 5: Update memory

**Step 1: Save project notes**

Create `/Users/li.ye/.claude/projects/-Users-li-ye-Documents-strava-helper/memory/MEMORY.md`:

```markdown
# Strava Helper Project

## What it is
Single `index.html` file — Strava .fit uploader hosted on GitHub Pages.

## Key files
- `index.html` — entire app (HTML + CSS + JS inline)
- `docs/plans/` — design and implementation docs

## Architecture
- Strava OAuth 2.0 with PKCE, tokens in localStorage
- Batch sequential upload via POST /api/v3/uploads + polling
- No build step, no dependencies

## Deployment
- Hosted on GitHub Pages
- Strava API app registered at https://www.strava.com/settings/api
- CLIENT_ID and CLIENT_SECRET embedded in index.html
```

---
