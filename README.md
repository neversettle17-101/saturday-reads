# saturday-reads

Frontend for the Saturday Reads Post Studio. A single HTML file — no build step, no framework, no npm. Users upload session photos, generate an AI caption, review it, and publish directly to Instagram.

---

## Project structure

```
saturday-reads/
└── index.html    # The entire frontend: HTML + CSS + JavaScript in one file
```

Everything is self-contained. There are no dependencies to install and nothing to compile.

---

## Setup

### 1. Deploy the backend first

Follow the setup in `saturday-reads-api/README.md` and get your deployed Vercel URL, e.g.:
```
https://saturday-reads-api.vercel.app
```

### 2. Set the backend URL in index.html

Open `index.html` and find this line near the top of the `<script>` section:

```js
const PROXY_URL = 'https://saturday-reads-api.vercel.app';
```

Replace the value with your own deployed backend URL.

### 3. Push to GitHub and enable Pages

```bash
git remote add origin https://github.com/<your-username>/saturday-reads.git
git add index.html
git commit -m "initial commit"
git push -u origin main
```

Then in your GitHub repo: **Settings → Pages → Source → Deploy from a branch → `main` / `(root)` → Save**

Your site will be live at:
```
https://<your-username>.github.io/saturday-reads/
```

Deployments take 1–2 minutes; track progress under the repo's **Actions** tab.

### 4. Lock down CORS on the backend

In Vercel Dashboard → `saturday-reads-api` → Settings → Environment Variables, set:

```
ALLOWED_ORIGIN = https://<your-username>.github.io
```

This restricts the backend to only accept requests from your GitHub Pages domain. Without this it defaults to `*` (any origin).

---

## User flow

```
1. Upload photos  →  drag-and-drop or file picker (up to 10 images)
2. Fill in details →  session date, books being read, optional notes
3. Generate caption →  AI reads the first photo + session details, writes caption
4. Review & edit  →  contenteditable caption box, regenerate if needed
5. Post           →  photos uploaded to Cloudinary, then published to Instagram
```

The user never sees or configures any API credentials.

---

## Code walkthrough

### HTML structure

The page is divided into cards, each representing a step:

```
<header>              — title and step indicator (1 Photos → 2 Details → 3 Caption → 4 Post)
<div class="card">    — photo upload (drop zone + thumbnail grid)
<div class="card">    — session details (date, books list, notes)
<button #genBtn>      — "Generate caption" trigger
<div #progressWrap>   — progress bar (hidden until active)
<div #outputArea>     — caption display + action buttons (hidden until caption ready)
<div #statusMsg>      — success / error messages
```

### State

Two module-level variables hold all runtime state:

```js
let photos = [];    // Array of { file: File, url: string } objects
                    // Slots are set to null when a photo is removed (index stability)
let caption = '';   // The currently generated caption string
```

`photos` uses null slots rather than splicing so that thumbnail DOM elements stay in sync with array indices — `rmPhoto(idx, el)` sets `photos[idx] = null` instead of shifting the array.

### Photo handling

```js
function addPhotos(files) {
  const room = 10 - photos.length;           // enforce max 10
  files.slice(0, room).forEach(file => {
    const entry = { file, url: URL.createObjectURL(file) };  // blob URL for preview
    photos.push(entry);
    // ... append thumbnail DOM element
  });
}
```

`URL.createObjectURL` creates an in-memory URL pointing to the file in the browser's memory — no upload happens here. The actual `File` object is kept in `photos[i].file` for later upload.

### Step 1: Generate caption (`run()`)

```
run()
 ├── validate: at least one photo
 ├── convert up to 5 photos to base64 strings
 ├── POST /api/generate-caption  { images: [...], context: { books, date, notes } }
 └── display result in #captionBox (contenteditable)
```

The function only generates the caption — it does **not** upload or post. This gives the user a chance to review.

```js
// File → base64 helper (strips the "data:image/jpeg;base64," prefix)
function toBase64(file) {
  return new Promise((res, rej) => {
    const r = new FileReader();
    r.onload = () => res(r.result.split(',')[1]);
    r.onerror = rej;
    r.readAsDataURL(file);
  });
}
```

`FileReader.readAsDataURL` gives you `data:image/jpeg;base64,<data>`. Splitting on `,` and taking index `[1]` strips the prefix, leaving only the raw base64 string that the backend expects.

### Step 2: Post to Instagram (`postToIG()`)

```
postToIG()
 ├── read caption from #captionBox (user may have edited it)
 ├── validate: photos exist, caption not empty
 ├── for each photo:
 │    ├── POST /api/cloudinary-sign  →  { signature, timestamp, cloudName, apiKey }
 │    └── POST cloudinary.com/.../upload  (direct, with signature)
 │         →  returns secure_url
 └── POST /api/publish-to-instagram  { imageUrl: secure_url, caption }
      →  { postId }
```

The Cloudinary upload goes **directly from the browser to Cloudinary** — it does not pass through Vercel. This avoids Vercel's 4.5 MB function body limit and makes uploads faster.

### Cloudinary signed upload explained

A "signed" upload means the request includes a cryptographic proof that your backend authorised it:

```js
// Browser asks backend to sign
const { signature, timestamp, cloudName, apiKey } = await fetch('/api/cloudinary-sign')

// Browser uploads directly to Cloudinary with that signature
const fd = new FormData()
fd.append('file', file)
fd.append('api_key', apiKey)         // identifies the account
fd.append('timestamp', timestamp)    // prevents replay attacks
fd.append('signature', signature)    // SHA256(params + API_SECRET) — computed server-side
```

Cloudinary recomputes the hash on its end. If it matches, the upload is accepted. The `API_SECRET` never leaves the backend.

### Caption box: `contenteditable`

```html
<div class="caption-area" id="captionBox" contenteditable="false"></div>
```

Setting `contenteditable="true"` turns any element into an inline editor — no `<textarea>` needed. The edit toggle function flips it:

```js
function toggleEdit() {
  const box = document.getElementById('captionBox')
  const isEditing = box.contentEditable === 'true'
  box.contentEditable = isEditing ? 'false' : 'true'
  if (!isEditing) box.focus()
}
```

When `postToIG()` reads the caption, it uses `.textContent.trim()` which gives the plain text regardless of any inline formatting the browser may have added during editing.

### Progress bar and step indicator

`setProgress(pct, label)` and `updateSteps(n)` are pure DOM update functions — no state involved. They're called throughout `run()` and `postToIG()` to keep the UI in sync with async operations:

```js
setProgress(10, 'Generating caption…')   // 10% filled
// ... await API call ...
setProgress(100, 'Caption ready.')
setTimeout(hideProgress, 2000)           // auto-hide after 2s
```

### CORS

The frontend is hosted on a different domain than the backend. Every `fetch()` call to `PROXY_URL` is a cross-origin request. The backend sets these headers on every response:

```js
res.setHeader('Access-Control-Allow-Origin', process.env.ALLOWED_ORIGIN || '*')
res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS')
res.setHeader('Access-Control-Allow-Headers', 'Content-Type')
```

Browsers also send a **preflight** `OPTIONS` request before any `POST` with a JSON body, to check if the server allows it. Each handler returns `200` immediately for `OPTIONS` requests:

```js
if (req.method === 'OPTIONS') return res.status(200).end()
```

Without this, every API call would fail in the browser before it even tried.

---

## Data flow diagram

```
Browser                          Vercel Backend                 External APIs
  │                                    │                              │
  │── POST /api/generate-caption ──────▶│                              │
  │   { images: [base64], context }    │── POST api.groq.com ────────▶│
  │                                    │◀─ { caption }                │
  │◀─ { caption } ─────────────────────│                              │
  │                                    │                              │
  │ (user reviews / edits caption)     │                              │
  │                                    │                              │
  │── POST /api/cloudinary-sign ───────▶│                              │
  │◀─ { signature, timestamp, ... } ───│                              │
  │                                    │                              │
  │── POST cloudinary.com/upload ─────────────────────────────────────▶│
  │   (direct — no Vercel) ←──────────────────────── { secure_url } ──│
  │                                    │                              │
  │── POST /api/publish-to-instagram ──▶│                              │
  │   { imageUrl, caption }            │── POST graph.instagram.com ─▶│
  │                                    │   create media container      │
  │                                    │── POST graph.instagram.com ─▶│
  │                                    │   publish container           │
  │◀─ { postId } ──────────────────────│◀─ { id: postId } ────────────│
```
