# google-photos-dedupe

A [Claude Code](https://claude.com/claude-code) **skill** that finds and removes duplicate photos from your Google Photos library, by driving your own logged-in browser session and calling Google Photos' internal API directly. No extension to install, no API keys, no OAuth, and nothing leaves your machine.

> ⚠️ This uses Google's **undocumented internal web API** and runs in your own authenticated session. It can break if Google changes things, and it deletes from your real library. Everything it removes goes to the **Bin (recoverable for 60 days)** — it never permanently deletes. Use at your own risk. Not affiliated with or endorsed by Google.

## Why this exists

Most "Google Photos duplicate remover" tools are flaky browser extensions, and Google **removed delete access from the official Photos API in 2025**, so you can't script deletions the clean way anymore. This skill takes the only reliable no-install route: it runs JavaScript inside your already-logged-in `photos.google.com` tab and calls the same internal API the web app uses.

It also bakes in the things that are easy to get wrong:

- **There are usually no exact byte-duplicates** — Google blocks identical re-uploads at upload time. So it targets re-uploads that share the exact capture-time + dimensions (the "same photo uploaded twice from two devices" case).
- **It excludes undated photos**, which otherwise collapse into one giant false "group" by camera resolution.
- **It spot-checks before trusting itself** and warns that old photos (second-only timestamps) can false-positive, restoring the risky era automatically.
- **It never empties the Bin.** That irreversible step stays yours.

## What it does / doesn't catch

| Catches | Doesn't catch |
| --- | --- |
| The same photo uploaded multiple times (same capture-time + dimensions) | Visual near-dupes: resizes, screenshots of photos, messenger re-encodes, light edits |

The visual near-dupes need a perceptual/ML image-similarity tool (e.g. a local-ML deduper extension). This skill deliberately sticks to the high-confidence, metadata-identifiable duplicates so it can run safely and unattended.

## Install

Copy the skill folder into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/google-photos-dedupe
curl -fsSL https://raw.githubusercontent.com/PepoBCN/google-photos-dedupe/main/SKILL.md \
  -o ~/.claude/skills/google-photos-dedupe/SKILL.md
```

Or just clone and copy `SKILL.md` into `~/.claude/skills/google-photos-dedupe/`.

## Use

In Claude Code (with a browser-automation integration that can run JS in a page, e.g. the Chrome connection):

```
/photos-dedupe
```

or just say *"dedupe my Google Photos"*. Claude opens your Photos tab, reads the library, groups the duplicates, shows you a sample to eyeball, moves the confirmed dupes to the Bin, and reports exact counts. You empty the Bin yourself once you're happy (or let it auto-purge in 60 days).

## How it works (technical)

It calls three internal `batchexecute` RPCs from the page context:

- `lcxiM` — page through the library timeline
- `XwAOJf` — move to Bin (`[null,1,keys,3]`) / restore (`[null,3,keys,2]`)
- `zy0IHe` — list the Bin

Full field mappings, the request-builder, the resumable chunked enumeration, the grouping logic and the safety rails are documented in [`SKILL.md`](./SKILL.md).

## Safety

- Deletions go to the **Bin, reversible for 60 days**.
- The skill **never permanently deletes / empties the Bin** — that's left to you.
- It **spot-checks** a visual sample and restores likely false positives before finishing.
- It reports exactly what it binned, restored and skipped.

## License

MIT — see [`LICENSE`](./LICENSE).
