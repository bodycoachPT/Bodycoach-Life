# Bodycoach LIFE — Full Setup Guide

Complete, step-by-step instructions to go from these files to a live app with working Google Drive sync. Estimated time: ~30 minutes.

---

## Quick answers before you start

**Will there be a cap on users?**
No practical cap for your use case. Google limits unverified apps ("Testing" mode) to **100 manually-approved users**. Since you have under 50 clients, you stay in Testing mode permanently — no Google verification process, no paperwork, no fees. Just add each client's Gmail to the approved list as they join.

**What does this cost?**
Zero. Google Drive API is free up to 1 billion requests/day (you'll use a few thousand). Netlify free tier covers 100GB bandwidth and unlimited deploys. Nothing to pay.

---

## What you'll end up with

- `https://your-site.netlify.app` — client app (you share this link with clients)
- `https://your-site.netlify.app/coach.html` — your coach dashboard (keep this private)
- Each client's data lives in a Google Drive folder you own, as a JSON file per client
- Data syncs automatically every time a client logs something

---

## Phase 1 — Google Cloud Setup

### Step 1 — Create a Google Cloud Project

1. Go to [https://console.cloud.google.com](https://console.cloud.google.com)
2. Sign in with your Google account (the one you'll use as coach)
3. At the top, click the project dropdown → **New Project**
4. Name: `Bodycoach LIFE` → **Create**
5. Wait ~10 seconds, then click the dropdown again and select your new project

---

### Step 2 — Enable Google Drive API

1. In the left sidebar: **APIs & Services → Library**
2. Search: `Google Drive API`
3. Click it → **Enable**

---

### Step 3 — Configure the OAuth Consent Screen

This is the screen your clients see when signing in for the first time.

1. Left sidebar: **APIs & Services → OAuth consent screen**
2. User type: **External** → **Create**
3. Fill in the form:
   - **App name**: `Bodycoach LIFE`
   - **User support email**: your Gmail
   - **Developer contact email**: your Gmail
   - Leave everything else blank
4. Click **Save and Continue**
5. On the **Scopes** screen — click **Save and Continue** (no changes needed)
6. On the **Test users** screen — click **Save and Continue** (you'll add users later)
7. Click **Back to Dashboard**

> Your app is now in "Testing" mode. This is intentional and permanent for your use case. Only the Google accounts you explicitly approve can sign in.

---

### Step 4 — Create OAuth Credentials

1. Left sidebar: **APIs & Services → Credentials**
2. Click **+ Create Credentials → OAuth client ID**
3. **Application type**: Web application
4. **Name**: `Bodycoach LIFE Web`
5. Under **Authorised JavaScript origins** — leave blank for now (you'll add your Netlify URL after deploying)
6. Click **Create**
7. A popup appears with your Client ID — it looks like:
   `123456789-abcdefghijk.apps.googleusercontent.com`
8. **Copy the Client ID and paste it somewhere safe** (Notes, email to yourself, etc.)

---

### Step 5 — Create the Shared Drive Folder

This is where all client data is stored. You own it; clients write to it via their OAuth token.

1. Go to [https://drive.google.com](https://drive.google.com)
2. Click **+ New → New folder**
3. Name it: `BCL Client Data` → **Create**
4. Double-click the folder to open it
5. Look at the URL in your browser — it will look like:
   `https://drive.google.com/drive/folders/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OjuAQHlQ`
6. **Copy the long string after `/folders/`** — this is your Folder ID. Save it with your Client ID.

**Now share the folder** so clients can write to it:
1. Right-click the `BCL Client Data` folder → **Share**
2. In the "Add people" box, add each client's Gmail address as you onboard them
3. Permission level: **Editor**
4. Uncheck "Notify people" (optional — up to you)
5. Click **Share**

> You'll come back here each time you add a new client. Just share the folder with their Gmail before you send them the app link.

---

## Phase 2 — Deploy to Netlify (First Deploy)

You need to deploy first to get your URL, then come back to Google Cloud to register it.

### Step 6 — First Deploy

1. Go to [https://app.netlify.com/drop](https://app.netlify.com/drop)
2. Sign up / log in (free account)
3. Open your project folder (the one with `index.html` and `coach.html`)
4. Drag **both files** onto the Netlify drop zone
5. Netlify will deploy and give you a random URL like `https://jolly-koala-abc123.netlify.app`

### Step 7 — Set a Custom Site Name (important — do this now)

The random name will change on every redeploy unless you lock it.

1. In your Netlify dashboard, click **Site configuration** (or **Site settings**)
2. Under **Site details**, click **Change site name**
3. Choose something like `bodycoach-michael` or `bcl-clients`
4. Your URL becomes: `https://bodycoach-michael.netlify.app`
5. **Copy your final URL** — you'll need it next

---

## Phase 3 — Wire Up Google OAuth

### Step 8 — Add Your Netlify URL to Google Cloud

1. Go back to [Google Cloud Console](https://console.cloud.google.com) → **APIs & Services → Credentials**
2. Click your **Bodycoach LIFE Web** OAuth client ID
3. Under **Authorised JavaScript origins**, click **+ Add URI**
4. Add your Netlify URL **exactly** (no trailing slash):
   `https://bodycoach-michael.netlify.app`
5. Also add `http://localhost` for testing locally (optional but useful)
6. Click **Save**

> Changes can take 5 minutes to propagate. Don't panic if sign-in fails immediately.

---

## Phase 4 — Configure the App Files

### Step 9 — Paste Your Credentials

Open `index.html` in a text editor (TextEdit, VS Code, Notepad — anything).

Search for (Cmd+F / Ctrl+F):
```
PASTE_YOUR_GOOGLE_CLIENT_ID_HERE
```

You'll find two lines that look like this:
```javascript
var GD_CLIENT_ID = 'PASTE_YOUR_GOOGLE_CLIENT_ID_HERE';
var GD_FOLDER_ID = 'PASTE_YOUR_DRIVE_FOLDER_ID_HERE';
```

Replace both placeholders with your actual values:
```javascript
var GD_CLIENT_ID = '123456789-abcdefghijk.apps.googleusercontent.com';
var GD_FOLDER_ID = '1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OjuAQHlQ';
```

**Save the file.**

Now do exactly the same thing in `coach.html` — same two lines, same values.

**Save coach.html.**

---

## Phase 5 — Final Deploy

### Step 10 — Redeploy with Credentials

1. Go back to [https://app.netlify.com](https://app.netlify.com)
2. Open your site
3. Go to **Deploys** tab
4. Drag both `index.html` and `coach.html` onto the deploy drop zone (or drag the whole folder)
5. Wait ~15 seconds for the deploy to finish

Your app is now live.

---

## Phase 6 — Add Client Access

### Step 11 — Approve Each Client's Google Account

Before a client can sign in, their Gmail must be approved in two places.

**In Google Cloud:**
1. **APIs & Services → OAuth consent screen**
2. Scroll to **Test users** → **+ Add users**
3. Enter the client's Gmail address
4. Click **Save**

**In Google Drive:**
1. Right-click `BCL Client Data` folder → **Share**
2. Add the client's Gmail → **Editor** → Share

Do both steps every time you add a new client. Each takes 30 seconds.

---

## How It Works Day-to-Day

**Client flow:**
1. They open your Netlify URL on their phone
2. First time only: they tap "Continue with Google" and sign in
3. First time only: they go through a short setup (name, weight, targets)
4. Every time they log something, it syncs automatically to Drive (3-second delay)
5. Their data loads back from Drive on any device they use

**Coach flow:**
1. You open `https://your-site.netlify.app/coach.html`
2. Sign in with your Google account
3. All clients who have synced appear in the list
4. Click a client → see their week summary, set targets, push a workout routine
5. Targets/routines are written to Drive → the client picks them up on their next app open

---

## Sending the App to Clients

When a client is ready to start, send them:
1. The app URL: `https://your-site.netlify.app`
2. A short message: *"Download this as an app on your phone — tap the Share button (iOS) or the three-dot menu (Android), then 'Add to Home Screen'"*

That's it. No App Store, no download, works on any phone.

---

## Updating the App

When you make changes to `index.html` or `coach.html`, redeploy by dragging the files to Netlify again. Client data in Drive is unaffected — it persists across all deploys.

---

## Troubleshooting

**"Sign in failed" / popup closes with no result**
- Your Netlify URL isn't added to Authorised JavaScript Origins, or it was added with a typo (trailing slash, wrong protocol). Double-check in Google Cloud.
- OAuth changes take up to 5 minutes. Wait and retry.

**Client not appearing in coach dashboard**
- They need to have signed in and synced at least once
- Check they're added as a Test User in OAuth consent screen
- Check their Gmail is a Folder Editor in Drive

**"Error loading clients" in coach.html**
- The `GD_FOLDER_ID` in `coach.html` doesn't match the actual folder ID
- Your Google account (signed into coach.html) doesn't have access to the folder

**Data not syncing**
- Open browser DevTools (F12) → Console tab — any red errors will say exactly what's wrong
- Most common cause: OAuth token expired. Refresh the page; it re-authenticates automatically.
- If a red popup appears saying "blocked", the client needs to allow popups for your domain (once only)

**"Access blocked: This app's request is invalid"**
- The Netlify URL is missing from Authorised JavaScript Origins
- Or the `GD_CLIENT_ID` in the file doesn't match what's in Google Cloud

**Clients on iOS: "You can't sign in because this browser is not supported"**
- They're opening the app from the iOS Safari in-app browser (e.g. from a link in Instagram/WhatsApp)
- Solution: open the URL in Safari directly, then add to home screen

---

## Summary of Values to Keep Safe

| What | Where to find it | Goes in |
|------|-----------------|---------|
| Client ID | Google Cloud → Credentials → your OAuth client | `GD_CLIENT_ID` in both files |
| Folder ID | Google Drive URL when folder is open | `GD_FOLDER_ID` in both files |
| Netlify URL | Your Netlify dashboard | OAuth Authorised Origins + what you share with clients |
