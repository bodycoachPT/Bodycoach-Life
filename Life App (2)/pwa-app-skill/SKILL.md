---
name: pwa-google-drive-app
description: >
  Build a zero-cost progressive web app (PWA) with Google Drive as the database
  and a separate coach/admin dashboard. Use this skill whenever building a
  mobile-first web app that needs user data persistence, multi-user sync,
  push from admin to client, and free hosting — especially fitness trackers,
  habit apps, client portals, or any "trainer + client" style product.
  Trigger on phrases like: "build me an app", "PWA", "no backend", "free stack",
  "Google Drive sync", "coach and client app", "single file app".
---

# PWA + Google Drive App — Full Build Pattern

Everything learned from building Bodycoach LIFE: a fitness tracker PWA with
Google Drive sync, a coach dashboard, and a client app — zero cost, zero backend.

---

## 1. Architecture Overview

```
index.html          ← Client app (users)
coach.html          ← Admin/coach dashboard
Google Drive Folder ← Database (one JSON file per user)
GitHub Pages        ← Free hosting
```

**Why this works:**
- Google Drive API is free up to 1 billion requests/day
- GitHub Pages is free, no bandwidth limits
- Google OAuth (GIS library) handles auth at no cost
- Under ~50 users → stay in Google "Testing" mode forever, no verification needed
- Single-file HTML means no build pipeline, no npm, no servers

---

## 2. File Structure Pattern

### Client app (`index.html`)
Plain single-file HTML. All CSS, JS, and HTML in one file. No bundler needed.

### Coach app (`coach.html`)
Uses a **bundler template** pattern to embed the full app as a JSON string:

```html
<script type="__bundler/template">
  "<!DOCTYPE html>..."  ← entire app HTML as JSON-encoded string
</script>
<script>
  document.addEventListener('DOMContentLoaded', async function() {
    const tmpl = document.querySelector('script[type="__bundler/template"]');
    const html = JSON.parse(tmpl.textContent);
    document.open(); document.write(html); document.close();
  });
</script>
```

**CRITICAL when editing coach.html programmatically:**
```python
import json, re

with open('coach.html') as f:
    raw = f.read()

m = re.search(r'<script type="__bundler/template">(.*?)</script>', raw, re.DOTALL)
app_html = json.loads(m.group(1).strip())

# --- make edits to app_html string here ---

new_tmpl = json.dumps(app_html).replace('</', '<\\/')  # ALWAYS escape </
raw = raw[:m.start(1)] + new_tmpl + raw[m.end(1):]

with open('coach.html', 'w') as f:
    f.write(raw)
```

**Never forget** `.replace('</', '<\\/')` — missing this breaks the HTML parser.

---

## 3. Google Drive Setup

### Credentials (store securely, never expose in logs)
```javascript
var GD_CLIENT_ID = 'YOUR_CLIENT_ID.apps.googleusercontent.com';
var GD_FOLDER_ID = 'YOUR_SHARED_FOLDER_ID';
```

### OAuth scope — CRITICAL DISTINCTION
```javascript
// Client app — MUST be 'drive' not 'drive.file'
var GD_SCOPE = 'https://www.googleapis.com/auth/drive openid email profile';

// Coach app — same
var GD_SCOPE = 'https://www.googleapis.com/auth/drive openid email profile';
```

**Why `drive` not `drive.file`:**
`drive.file` only allows access to files the app itself created. If the coach
creates an assignments file, the client cannot read it with `drive.file`. This
is the most common silent failure in this architecture.

### File naming convention
```javascript
// Per-user data file
'bcl_' + email.replace(/[^a-z0-9]/gi,'_').toLowerCase() + '_data.json'

// Coach → client assignments
'bcl_' + email.replace(/[^a-z0-9]/gi,'_').toLowerCase() + '_assignments.json'
```

### Google Cloud setup checklist
1. Create project → Enable Drive API
2. OAuth consent screen → External → Testing mode (stays permanent for <100 users)
3. Create OAuth client ID → Web application
4. Add authorised JavaScript origins: `https://yourdomain.github.io`
5. Add each client's Gmail as a Test User
6. Share the Drive folder with each client as Editor

---

## 4. Auth Pattern

### GIS (Google Identity Services) initialisation
```javascript
var GD_CLIENT_ID = '...';
var GD_FOLDER_ID = '...';
var GD_SCOPE = 'https://www.googleapis.com/auth/drive openid email profile';

var GD = {
  configured: GD_CLIENT_ID.indexOf('PASTE') === -1 && GD_FOLDER_ID.indexOf('PASTE') === -1,
  user: null, token: null, tokenExpiry: 0,
  fileId: null, asgFileId: null,
  syncing: false, lastSync: null
};

var _gdTokenClient = null;

function gdInit() {
  if (!GD.configured) { checkOnboarding(); return; }
  var saved = localStorage.getItem('bcl_gd_user');
  GD.user = saved ? JSON.parse(saved) : null;

  if (GD.user) {
    if (_isIOSStandalone()) {
      // iOS PWA silent auth hangs — show manual sign-in screen instead
      gdShowAuthScreen();
      return;
    }
    // Try silent token refresh, fallback to auth screen after 6s
    var _done = false;
    var _timer = setTimeout(function() {
      if (!_done) { _done = true; gdShowAuthScreen(); }
    }, 6000);
    _gdRequestToken(true, function(ok) {
      if (_done) return;
      _done = true; clearTimeout(_timer);
      if (ok) gdAfterTokenReady(); else gdShowAuthScreen();
    });
  } else {
    gdShowAuthScreen();
  }
}

function _isIOSStandalone() {
  return /iphone|ipad|ipod/i.test(navigator.userAgent) &&
    (window.navigator.standalone === true ||
     window.matchMedia('(display-mode: standalone)').matches);
}

function gdSignIn() {
  _gdTokenClient = null; // reset before each attempt
  var btn = document.getElementById('gd-signin-btn');
  if (btn) { btn.disabled = true; btn.textContent = 'Signing in…'; }
  _gdRequestToken(false, function(ok) {
    if (btn) { btn.disabled = false; btn.textContent = 'Continue with Google'; }
    if (ok) gdAfterTokenReady(); else { /* show error */ }
  });
}

function _gdRequestToken(silent, cb) {
  function attempt() {
    if (!_gdTokenClient) {
      _gdTokenClient = google.accounts.oauth2.initTokenClient({
        client_id: GD_CLIENT_ID, scope: GD_SCOPE,
        prompt: silent ? '' : 'select_account',
        callback: function(resp) {
          if (resp.error || !resp.access_token) { cb(false); return; }
          GD.token = resp.access_token;
          GD.tokenExpiry = Date.now() + (resp.expires_in - 60) * 1000;
          cb(true);
        }
      });
    }
    _gdTokenClient.requestAccessToken({ prompt: silent ? '' : 'select_account' });
  }
  if (typeof google !== 'undefined' && google.accounts) attempt();
  else setTimeout(function() { _gdRequestToken(silent, cb); }, 300);
}
```

### Authenticated fetch helper
```javascript
async function _gdFetch(url, opts) {
  opts = opts || {};
  opts.headers = Object.assign({ Authorization: 'Bearer ' + GD.token }, opts.headers || {});
  var resp = await fetch(url, opts);
  if (resp.status === 401) {
    // Token expired — refresh and retry once
    await new Promise(function(res) {
      _gdRequestToken(true, function() { res(); });
    });
    if (GD.token) opts.headers.Authorization = 'Bearer ' + GD.token;
    resp = await fetch(url, opts);
  }
  return resp;
}
```

---

## 5. Data Sync Pattern

### Save user data to Drive
```javascript
// Debounce — don't hammer Drive on every keystroke
var _syncTimer = null;
function gdScheduleSync() {
  clearTimeout(_syncTimer);
  _syncTimer = setTimeout(gdSync, 3000);
}

async function gdSync() {
  if (!GD.token || GD.syncing) return;
  GD.syncing = true;
  try {
    var payload = JSON.stringify({ email: GD.user.email, data: getAllLocalData() });
    if (GD.fileId) {
      await _gdFetch(
        'https://www.googleapis.com/upload/drive/v3/files/' + GD.fileId + '?uploadType=media',
        { method: 'PATCH', headers: { 'Content-Type': 'application/json' }, body: payload }
      );
    } else {
      await gdFindOrCreateFile();
    }
  } finally { GD.syncing = false; }
}

async function gdFindOrCreateFile() {
  var name = 'bcl_' + GD.user.email.replace(/[^a-z0-9]/gi,'_').toLowerCase() + '_data.json';
  var q = "name='" + name + "' and '" + GD_FOLDER_ID + "' in parents and trashed=false";
  var r = await _gdFetch('https://www.googleapis.com/drive/v3/files?q=' + encodeURIComponent(q) + '&fields=files(id)');
  var d = await r.json();
  if (d.files && d.files.length) {
    GD.fileId = d.files[0].id;
  } else {
    // Create with multipart upload
    var boundary = 'bcl_b_' + Date.now();
    var body = '...' // user data JSON
    var mp = '--' + boundary + '\r\nContent-Type: application/json\r\n\r\n'
      + JSON.stringify({ name: name, parents: [GD_FOLDER_ID], mimeType: 'application/json' })
      + '\r\n--' + boundary + '\r\nContent-Type: application/json\r\n\r\n'
      + body + '\r\n--' + boundary + '--';
    var cr = await _gdFetch('https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart', {
      method: 'POST',
      headers: { 'Content-Type': 'multipart/related; boundary=' + boundary },
      body: mp
    });
    var created = await cr.json();
    GD.fileId = created.id;
  }
}
```

**CRITICAL:** Always use **multipart upload** when creating a new file with content.
The two-step approach (POST metadata first, then PATCH content) is unreliable —
the first call can succeed but content upload silently fails.

---

## 6. Coach → Client Push Pattern

### Shared push helper (put in coach app)
```javascript
async function _pushAsgToClient(client, asg) {
  var safe    = client.email.replace(/[^a-z0-9]/gi, '_').toLowerCase();
  var asgName = 'bcl_' + safe + '_assignments.json';
  var body    = JSON.stringify(asg);

  // Self-heal: find existing file ID if not cached
  if (!client.asgFileId) {
    try {
      var q  = "name='" + asgName + "' and '" + GD_FOLDER_ID + "' in parents and trashed=false";
      var fr = await _fetch('https://www.googleapis.com/drive/v3/files?q=' + encodeURIComponent(q) + '&fields=files(id)');
      if (fr.ok) { var fd = await fr.json(); if (fd.files && fd.files.length) client.asgFileId = fd.files[0].id; }
    } catch(e) {}
  }

  var res;
  if (client.asgFileId) {
    res = await _fetch('https://www.googleapis.com/upload/drive/v3/files/' + client.asgFileId + '?uploadType=media',
      { method: 'PATCH', headers: { 'Content-Type': 'application/json' }, body: body });
    if (!res.ok) client.asgFileId = null; // stale — fall through
  }

  if (!client.asgFileId) {
    var boundary = 'bcl_b_' + Date.now();
    var mp = '--' + boundary + '\r\nContent-Type: application/json\r\n\r\n'
      + JSON.stringify({ name: asgName, parents: [GD_FOLDER_ID], mimeType: 'application/json' })
      + '\r\n--' + boundary + '\r\nContent-Type: application/json\r\n\r\n'
      + body + '\r\n--' + boundary + '--';
    res = await _fetch('https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart', {
      method: 'POST',
      headers: { 'Content-Type': 'multipart/related; boundary=' + boundary },
      body: mp
    });
    if (res.ok) { var created = await res.clone().json(); client.asgFileId = created.id; }
  }

  return res && res.ok;
}

async function _readClientAsg(client) {
  if (!client.asgFileId) return null;
  try {
    var r = await _fetch('https://www.googleapis.com/drive/v3/files/' + client.asgFileId + '?alt=media');
    if (r.ok) return await r.json();
  } catch(e) {}
  return null;
}
```

### Client-side: poll for assignments
```javascript
// Poll every 5 mins + check immediately on app open
function gdScheduleAssignmentPoll() {
  clearInterval(GD.asgPollTimer);
  GD.asgPollTimer = setInterval(function() {
    if (GD.token) gdCheckAssignments();
  }, 5 * 60 * 1000);
}

async function gdCheckAssignments() {
  if (!GD.token || !GD.user) return;
  var safe    = GD.user.email.replace(/[^a-z0-9]/gi,'_').toLowerCase();
  var asgName = 'bcl_' + safe + '_assignments.json';
  var q       = "name='" + asgName + "' and '" + GD_FOLDER_ID + "' in parents and trashed=false";
  var r       = await _gdFetch('https://www.googleapis.com/drive/v3/files?q='
                  + encodeURIComponent(q) + '&fields=files(id,modifiedTime)');
  if (!r.ok) return;
  var d = await r.json();
  if (!d.files || !d.files.length) return;
  GD.asgFileId = d.files[0].id;

  // Skip if already applied this version
  var lastChecked = localStorage.getItem('bcl_asg_checked');
  if (lastChecked && lastChecked >= d.files[0].modifiedTime) return;

  var ar = await _gdFetch('https://www.googleapis.com/drive/v3/files/' + GD.asgFileId + '?alt=media');
  if (!ar.ok) return;
  var asg; try { asg = await ar.json(); } catch(e) { return; }

  // Apply targets to profile
  if (asg.targets) {
    var prof = {}; try { prof = JSON.parse(localStorage.getItem('bcl_profile') || '{}'); } catch(e) {}
    Object.assign(prof, asg.targets);
    localStorage.setItem('bcl_profile', JSON.stringify(prof));
    loadState(); // reload targets into app state
  }

  // Merge routines
  if (asg.routines && Array.isArray(asg.routines)) {
    var existing = []; try { existing = JSON.parse(localStorage.getItem('bcl_routines_v1') || '[]'); } catch(e) {}
    asg.routines.forEach(function(rt) {
      var i = existing.findIndex(function(e) { return e.id === rt.id; });
      if (i >= 0) existing[i] = rt; else existing.push(rt);
    });
    localStorage.setItem('bcl_routines_v1', JSON.stringify(existing));
  }

  // Store message in log
  if (asg.message) {
    var log = getCoachMessageLog();
    if (!log.length || log[0].text !== asg.message) {
      log.unshift({ text: asg.message, at: new Date().toISOString(), read: false });
      localStorage.setItem('bcl_coach_messages', JSON.stringify(log.slice(0, 50)));
    }
  }

  localStorage.setItem('bcl_asg_checked', d.files[0].modifiedTime);
  try { renderToday(); } catch(e) {}
}
```

---

## 7. State Management Pattern

### Single state object, stored in localStorage
```javascript
var S = { data: {}, goals: [], tab: 'exercise', page: 'today' };

function loadState() {
  try { S.data  = JSON.parse(localStorage.getItem('bcl_v2') || '{}'); } catch(e) {}
  try { S.goals = JSON.parse(localStorage.getItem('bcl_goals_v2') || '[]'); } catch(e) {}
  if (!S.data) S.data = seedTracking();

  // Apply coach-pushed targets from profile
  try {
    var prof = JSON.parse(localStorage.getItem('bcl_profile') || '{}');
    CORE_METRICS.forEach(function(m) {
      var saved = prof['tgt_' + m.id];
      if (saved && saved > 0) m.target = saved;
    });
  } catch(e) {}
}

function saveState() {
  try {
    localStorage.setItem('bcl_v2', JSON.stringify(S.data));
    localStorage.setItem('bcl_goals_v2', JSON.stringify(S.goals));
  } catch(e) {}
  gdScheduleSync(); // debounced Drive sync
}

// Generic get/set for habit data
// Data shape: S.data[catKey][id][dayIndex] = value
function gv(catKey, id, day) {
  try { return S.data[catKey][id][day]; } catch(e) { return undefined; }
}
function sv(catKey, id, day, val) {
  if (!S.data[catKey]) S.data[catKey] = {};
  if (!S.data[catKey][id]) S.data[catKey][id] = {};
  S.data[catKey][id][day] = val;
}

// Day index = days since epoch (timezone-safe)
var TDY = Math.floor((Date.now() - new Date().getTimezoneOffset() * 60000) / 86400000);
```

---

## 8. Mobile PWA Patterns

### Bottom navigation
```html
<div id="bottom-nav" style="
  position:fixed; bottom:0; left:50%; transform:translateX(-50%);
  width:100%; max-width:430px;
  padding-bottom:env(safe-area-inset-bottom);
  background:var(--surface); border-top:1px solid var(--border);
  display:flex; z-index:100;
">
  <!-- nav buttons -->
</div>
```

### Scroll fix — bottom nav blocks last content item
```javascript
function fixScrollPadding() {
  var nav = document.getElementById('bottom-nav');
  if (!nav) return;
  var h = nav.getBoundingClientRect().height;
  if (h < 20) h = 70;
  var pad = (h + 24) + 'px';
  ['today-content','progress-content','goals-content','me-content'].forEach(function(id) {
    var el = document.getElementById(id);
    if (el) el.style.paddingBottom = pad;
  });
}
window.addEventListener('load', function() { setTimeout(fixScrollPadding, 100); });
window.addEventListener('resize', fixScrollPadding);
```

**Also required in CSS:**
```css
#main { overflow: hidden; height: 100dvh; }
/* Each content div handles its own scroll: */
#today-content { overflow-y: auto; }
```

### Bottom sheet (log panel, modal)
```html
<div id="sheet-bg" style="display:none;position:fixed;inset:0;z-index:190;background:rgba(10,15,30,.5);backdrop-filter:blur(6px)">
  <div id="sheet" onclick="event.stopPropagation()"
    style="position:absolute;bottom:0;left:50%;transform:translateX(-50%);
           width:100%;max-width:430px;background:var(--surface);
           border-radius:24px 24px 0 0;padding:16px 20px calc(var(--sab)+16px);
           max-height:88dvh;overflow-y:auto;">
    <div id="sheet-inner"></div>
  </div>
</div>
```

```javascript
function openSheet() {
  var bg = document.getElementById('sheet-bg');
  var sheet = document.getElementById('sheet');
  bg.style.display = 'block';
  sheet.style.transition = 'none';
  sheet.style.transform = 'translateX(-50%) translateY(100%)';
  requestAnimationFrame(function() {
    requestAnimationFrame(function() {
      sheet.style.transition = 'transform .35s cubic-bezier(.16,1,.3,1)';
      sheet.style.transform = 'translateX(-50%) translateY(0)';
    });
  });
}
function closeSheet() {
  var sheet = document.getElementById('sheet');
  sheet.style.transition = 'transform .3s cubic-bezier(.16,1,.3,1)';
  sheet.style.transform = 'translateX(-50%) translateY(100%)';
  setTimeout(function() {
    document.getElementById('sheet-bg').style.display = 'none';
  }, 300);
}
```

### Dropdown inside table — position:fixed workaround
Dropdowns inside `<table>` cells get clipped by `overflow:hidden`. Fix:
append to `<body>` with `position:fixed` and use `getBoundingClientRect`.
```javascript
var drop = document.createElement('div');
drop.style.cssText = 'display:none;position:fixed;background:var(--surface);'
  + 'border:1px solid var(--border);border-radius:8px;max-height:220px;'
  + 'overflow-y:auto;z-index:99999;box-shadow:0 4px 16px rgba(0,0,0,.25);';
document.body.appendChild(drop);

// Position on open:
var rect = inputEl.getBoundingClientRect();
drop.style.top  = (rect.bottom + 4) + 'px';
drop.style.left = rect.left + 'px';
drop.style.width = rect.width + 'px';
drop.style.display = 'block';
```

---

## 9. CSS Design System

### CSS variables (light + dark mode)
```css
:root {
  --bg:          #F9F8F7;
  --surface:     #FFFFFF;
  --border:      #EBEBEB;
  --faint:       #F4F3F2;
  --text:        #1A1A20;
  --muted:       #9CA3AF;
  --light:       #6B7280;
  --brand-blue:  #0A84FF;
  --green:       #22C55E;
  --amber:       #F59E0B;
  --red:         #DC2626;
  --red-bg:      #FEF2F2;
  --red-bd:      #FECACA;
  --shadow-sm:   0 1px 3px rgba(0,0,0,.06);
  --sab:         env(safe-area-inset-bottom);
}
[data-theme="dark"] {
  --bg:      #0F1117;
  --surface: #1C1F2E;
  --border:  #2A2D3E;
  --faint:   #252836;
  --text:    #F1F5F9;
  --muted:   #64748B;
  --light:   #94A3B8;
}
```

### PWA meta tags (essential)
```html
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="theme-color" content="#F9F8F7">
<link rel="manifest" href="manifest.json">  <!-- or inline with blob URL -->
```

### PWA manifest (inline via JS)
```javascript
var manifest = { name:'App Name', short_name:'App', start_url:'/',
  display:'standalone', background_color:'#F9F8F7', theme_color:'#F9F8F7',
  icons:[{src:'data:image/svg+xml,...', sizes:'192x192', type:'image/svg+xml'}] };
var blob = new Blob([JSON.stringify(manifest)], {type:'application/manifest+json'});
document.querySelector('link[rel="manifest"]').href = URL.createObjectURL(blob);
```

---

## 10. Exercise Type System

When building any workout/exercise tracking app, define measurement types:

| Type | Input | Use case |
|------|-------|----------|
| `weight_reps` | sets × reps @ kg | Barbell/dumbbell exercises |
| `bodyweight_reps` | sets × reps | Push-ups, pull-ups, dips |
| `distance_time` | km + mins | Running, cycling, rowing |
| `time` | seconds/mins | Planks, holds, spin class |
| `reps` | count only | Standalone rep count |

```javascript
var EX_TYPE = {
  'Core': 'time',          // planks etc default to timed
  'Cardio': 'distance_time',
  'Chest': 'weight_reps',
  'Back': 'weight_reps',
  // ... per category defaults
};

var EX_TYPE_OVERRIDE = {
  // Override specific exercises that differ from category default
  'Plank': 'time',
  'Push Up': 'bodyweight_reps',
  'Pull Up': 'bodyweight_reps',
  'Jump Rope': 'time',
  'Burpees': 'bodyweight_reps',
  // etc.
};

function getExType(name, category) {
  return EX_TYPE_OVERRIDE[name] || EX_TYPE[category] || 'weight_reps';
}
```

---

## 11. JS Syntax Safety (Python editing)

When writing JS inside Python strings, escaping is a common source of bugs.

### Single quotes inside onclick attributes
```python
# WRONG — breaks JS string
"onclick=\"logItem('hub-num-inp')\""

# CORRECT — use escaped single quotes
"onclick=\"logItem(\\'hub-num-inp\\')\""

# In Python raw strings (r"""..."""), \' is literal backslash+quote
# which becomes \' in JS (valid escaped single quote)
```

### Always syntax-check after edits
```python
import re, subprocess, tempfile, os

with open('index.html') as f:
    raw = f.read()

scripts = re.findall(r'<script(?![^>]*src)[^>]*>(.*?)</script>', raw, re.DOTALL)
sc = max(scripts, key=len)

with tempfile.NamedTemporaryFile(mode='w', suffix='.js', delete=False) as f:
    f.write(sc); fname = f.name

result = subprocess.run(['node', '--check', fname], capture_output=True, text=True)
os.unlink(fname)
print(result.stderr or "SYNTAX OK")
```

### Apostrophes in JS string literals
```javascript
// WRONG
'Check your phone's Health app.'
// CORRECT
'Check your phone\'s Health app.'
```

---

## 12. Common Bugs Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| Push from coach never arrives | Client scope is `drive.file` | Change to `drive` — all clients re-auth once |
| Log button does nothing | Runtime error in render function (missing helper fn) | Check for deleted functions; wrap in try/catch |
| Bottom items hidden behind nav | `#main { overflow-y: auto }` creates dual scroll | Set `#main { overflow: hidden }`, use JS padding fix |
| iOS login loops forever | `requestAccessToken({prompt:''})` hangs in standalone mode | Detect `navigator.standalone`, show manual sign-in |
| Dropdown invisible in table | `position:absolute` clipped by table overflow | Append to body with `position:fixed` |
| Coach push creates duplicate files | Two-step create (POST then PATCH) | Use single multipart upload |
| Targets pushed but UI unchanged | `renderToday()` only called inside `if (message)` block | Move render call outside the condition |
| Exercise type shows wrong input | `reps` type falls through to `time` handler | Use `bodyweight_reps` for bodyweight exercises |
| `node --check` passes but app breaks | Runtime error (not syntax error) | Add try/catch in render functions, log to console |
| `coach.html` changes lost | Edited outer HTML not the bundler template | Always edit via `json.loads` / `json.dumps` pattern |

---

## 13. Deployment: GitHub Pages

Completely free. No credit card. No limits for <50 users.

1. Create GitHub account → New repository (Public)
2. Upload `index.html` + `coach.html` via web UI: **Add file → Upload files**
3. **Settings → Pages → Branch: main, / (root) → Save**
4. URL: `https://username.github.io/repo-name`
5. Add this URL to Google Cloud → Credentials → Authorised JavaScript origins
6. Future updates: click file on GitHub → pencil icon → paste new content → Commit

**After scope change (`drive.file` → `drive`):**
Every existing user must sign out and back in once to grant the new permission.

---

## 14. Coach Dashboard Pattern

### Loading clients from Drive
```javascript
async function loadClients() {
  var q = "'" + GD_FOLDER_ID + "' in parents and trashed=false and name contains 'bcl_'";
  var d = await _fetchJSON('https://www.googleapis.com/drive/v3/files?q='
    + encodeURIComponent(q) + '&fields=files(id,name,modifiedTime)&pageSize=200');
  var files = d.files || [];

  var dataFiles = files.filter(function(f) { return f.name.endsWith('_data.json'); });
  var asgFiles  = files.filter(function(f) { return f.name.endsWith('_assignments.json'); });

  var clients = [];
  for (var i = 0; i < dataFiles.length; i++) {
    var file = dataFiles[i];
    var r    = await _fetch('https://www.googleapis.com/drive/v3/files/' + file.id + '?alt=media');
    var payload = await r.json();
    var email   = payload.email;
    var profile = JSON.parse(payload.data && payload.data.bcl_profile || '{}');
    var asgF    = asgFiles.find(function(af) {
      return af.name.includes(email.replace(/[^a-z0-9]/gi,'_').toLowerCase());
    });
    clients.push({
      email: email, name: profile.name || email,
      fileId: file.id, asgFileId: asgF ? asgF.id : null,
      lastSync: file.modifiedTime, payload: payload
    });
  }
  return clients;
}
```

### Push targets to client
```javascript
async function pushTargets(client, targets, message) {
  var asg = (await _readClientAsg(client)) || {}; // preserve existing routines
  asg.pushedAt = new Date().toISOString();
  asg.targets  = targets;
  if (message) asg.message = message;
  var ok = await _pushAsgToClient(client, asg);
  return ok;
}
```

---

## 15. Onboarding Flow Pattern

```javascript
var OB_STEPS = ['welcome','profile','targets','schedule','done'];
var OB_PROFILE = {};

function checkOnboarding() {
  var done = localStorage.getItem('bcl_onboarded');
  if (done) { initApp(); return; }
  showOnboardingStep('welcome');
}

function finishOnboarding() {
  localStorage.setItem('bcl_profile', JSON.stringify(OB_PROFILE));
  localStorage.setItem('bcl_onboarded', '1');
  initApp();
  if (GD.token) gdSync(); // first sync after onboarding
}
```

---

## 16. Session History Pattern

Store workout sessions as array in user data:
```javascript
// Save a completed session
function saveSession(session) {
  // session = { id, date, routineId, routineName, duration, exercises: [...] }
  var sessions = getSessions();
  sessions.unshift(session); // newest first
  if (sessions.length > 200) sessions = sessions.slice(0, 200);
  localStorage.setItem('bcl_sessions_v1', JSON.stringify(sessions));
  sv('exercise', 'workout', TDY, true); // mark workout done for today
  saveState();
}

function getSessions() {
  try { return JSON.parse(localStorage.getItem('bcl_sessions_v1') || '[]'); } catch(e) { return []; }
}
```

Coach reads sessions from `payload.data.bcl_sessions_v1` after loading client data file.

---

## 17. Build Sequence for a New App

1. **Scaffold** — single `index.html` with CSS vars, PWA meta, bottom nav
2. **Auth** — GIS init, sign-in button, `gdAfterTokenReady`, Drive file create
3. **State** — `loadState`, `saveState`, `gv`, `sv`, `TDY`
4. **Onboarding** — multi-step wizard, profile to localStorage
5. **Core screens** — Today/dashboard, Log sheet, Progress, Me/profile
6. **Sync** — debounced `gdSync`, `gdFindOrCreateFile`
7. **Coach app** — `coach.html` with bundler template, `loadClients`, push helpers
8. **Assignment poll** — `gdCheckAssignments`, 5-min interval + on-open check
9. **Mobile polish** — scroll padding, safe area insets, bottom sheet animations
10. **Deploy** — GitHub Pages, update Google OAuth origins, test on iOS Safari

---

## 18. Infrastructure Constraints

Always respect these for this class of app:
- **Cost:** Free only. No paid APIs, no paid databases, no paid servers.
- **Scale:** <50 users. Do not over-engineer.
- **Architecture:** Single-file as much as possible. Avoid build steps.
- **Storage:** Google Drive API first. No paid database.
- **Native:** PWA only. No App Store submissions.
- **Auth:** Google Identity Services (GIS). Free, handles token refresh.
