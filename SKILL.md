---
name: google-photos-dedupe
description: Find and remove duplicate photos from your Google Photos library by driving your logged-in browser session and calling Google's internal API directly (no extension, no official API, no data leaves your machine). Trigger when the user says "dedupe my google photos", "/photos-dedupe", "clean my google photos duplicates", or similar.
---

# Google Photos de-duplication

Removes duplicate photos from a Google Photos library by talking to Google's **internal `batchexecute` API** from inside the user's logged-in `photos.google.com` tab (via a browser-automation tool that can run JavaScript in the page, e.g. Claude's Chrome integration).

## Why it works this way (read first)
- **The official Google Photos API can't delete** — Google removed write/delete access in 2025. An OAuth token won't help you delete.
- **Browser extensions can't be installed by automation** (`chrome://extensions` needs in-page clicks plus a native file dialog). So extension-based dedupers require a manual install.
- **The working path: run JS in the logged-in Photos page.** `fetch` to the internal API is same-origin and carries the user's auth cookies. This is the whole trick.
- **Exact byte-duplicates basically don't exist** — Google blocks identical re-uploads at upload time. So "duplicates" are either (a) re-uploads sharing capture-time + dimensions (catchable from metadata, this skill), or (b) visual near-dupes: resizes, screenshots, messenger re-encodes (need image analysis, NOT covered here).

## Setup
1. Open `https://photos.google.com/` in the controlled browser and confirm signed in (`window.WIZ_global_data.SNlM0e` is present).
2. All calls go through one helper that reads tokens from `window.WIZ_global_data` and POSTs to `batchexecute`.

## The internal API (rpcids)
- List library timeline: **`lcxiM`**, payload `[pageId, timestamp(null), pageSize(500), null, 1, sourceCode]` — sourceCode `1`=library, `2`=archive, `3`=both. Returns `data[0]`=items, `data[1]`=nextPageId.
- Library item shape: `mediaKey=item[0]`, `thumbUrl=item[1][0]`, `w=item[1][1]`, `h=item[1][2]`, `takenTs(ms)=item[2]`, **`dedupKey=item[3]`** (trash/restore use this).
- Move to Bin: **`XwAOJf`**, payload `[null, 1, dedupKeyArray, 3]`. Reversible.
- Restore from Bin: **`XwAOJf`**, payload `[null, 3, dedupKeyArray, 2]`.
- List Bin: **`zy0IHe`**, payload `[pageId]`. Same item shape as library.

## Helper (run once per page load)
```js
window.__api = async (rpcid, rd) => {
  const G = window.WIZ_global_data;
  const wrapped = [[[rpcid, JSON.stringify(rd), null, 'generic']]];
  const body = `f.req=${encodeURIComponent(JSON.stringify(wrapped))}&at=${encodeURIComponent(G.SNlM0e)}&`;
  const params = new URLSearchParams({rpcids:rpcid,'source-path':location.pathname,'f.sid':G.FdrFJe,bl:G.cfb2h,pageId:'none',rt:'c'});
  const r = await fetch(`https://photos.google.com${G.eptZe}data/batchexecute?${params}`,
    {method:'POST',credentials:'include',headers:{'content-type':'application/x-www-form-urlencoded;charset=UTF-8'},body});
  const txt = await r.text();
  const line = txt.split('\n').find(l => l.includes('wrb.fr'));
  if (!line) throw new Error('no envelope');
  return JSON.parse(JSON.parse(line)[0][2]);
};
```

## Procedure

### 1. Enumerate the library — RESUMABLE, chunked
Renderer eval calls time out (~45s) and libraries are large (10k–30k+), so page in chunks across calls. Keep state on `window`; use a `running` guard (a timed-out loop keeps running in the renderer and double-counts — if numbers look wrong, **reload the tab** to kill it and restart clean).
```js
window.__gp = {items:[], pageId:null, done:false, pages:0, running:false}; // init once
await (async () => {
  const gp = window.__gp;
  if (gp.running) return {busy:true}; if (gp.done) return {done:true,total:gp.items.length};
  gp.running = true;
  try { let n=0; while(n<14){
    const d = await window.__api('lcxiM',[gp.pageId,null,500,null,1,1]);
    for (const it of (d?.[0]||[])) gp.items.push({d:it?.[3], t:it?.[2], w:it?.[1]?.[1], h:it?.[1]?.[2]});
    gp.pageId = d?.[1]; gp.pages++; n++; if(!gp.pageId){gp.done=true;break;}
  }} finally { gp.running=false; }
  return {done:gp.done, collected:gp.items.length, hasMore:!!gp.pageId};
})()
```

### 2. Group into high-confidence duplicates
Group by `takenTs(ms) | w | h`. A group of 2–4 with a **valid** timestamp is almost certainly the same photo uploaded more than once. Keep one per group; collect the rest's `dedupKey`s.
- **EXCLUDE placeholder timestamps** `t <= 946684800000` (year 2000). Undated photos collapse to `-62169897600000` and would falsely group hundreds of distinct photos by camera resolution. A single huge "group" (e.g. hundreds) is the tell.
- **SKIP groups of 5+** — flag for manual review, don't auto-delete big clusters.

### 3. Spot-check BEFORE trusting it (critical)
The metadata heuristic has a real false-positive rate: **old photos** (pre-~2011) often timestamp only to the second, so two different burst shots at the same resolution collide.
- Render ~12 sample pairs (kept thumb vs candidate thumb) into a fixed overlay and **screenshot to eyeball**. Build the DOM with `document.createElement` (Photos enforces Trusted Types — `innerHTML` throws). Size thumbs `url+'=w200-h200'`, `objectFit:'contain'`.
- **Do NOT rely on in-page canvas/`getImageData` perceptual hashing** — cross-origin Google thumbnails don't read back reliably (returns identical pixels for different images). Trust the screenshot, not auto-hashing.
- If false positives appear, **restore the risky era** (e.g. pre-2011): `XwAOJf [null,3,keys,2]`.

### 4. Move duplicates to the Bin, in batches
```js
for (let i=0;i<keys.length;i+=200) await window.__api('XwAOJf',[null,1,keys.slice(i,i+200),3]);
```
Verify via `zy0IHe` count (the Bin UI lazy-loads and can look empty even when populated — count via API). Navigating wipes `window` state, not the server-side deletions.

## HARD RULES
- **Everything goes to the Bin (recoverable 60 days). NEVER empty the Bin / permanently delete** — that's irreversible; leave it to the user (Bin → "Empty bin") or the automatic 60-day purge.
- **Spot-check and restore false positives before declaring done.** Don't claim "clean" off metadata alone.
- **Report exact counts**: total library, binned, restored, skipped (big/undated groups), and that nothing is permanently gone.
- This only catches metadata-identifiable duplicates. Visual near-dupes (resizes/screenshots/re-encodes) need an ML image-similarity tool — flag that as the remaining gap.
