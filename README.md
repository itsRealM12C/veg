---
name: veg-extractor
description: Reverse-engineer and inspect VEGAS Pro (Sony/MAGIX) .veg project files — Wave64-style GUID chunk container, recursive chunk walker, UTF-16LE string/path extraction, and heuristic track/event timeline, entirely client-side. Trigger whenever the user has a .veg file or asks to open/inspect/extract a VEGAS project. Tick-to-seconds conversion, audio-vs-video track type, and the playback-rate field are unresolved or unproven below — preserve those caveats rather than presenting them as settled.
---

# .veg — VEGAS Pro Project Format (reverse-engineering notes)

**Scope:** `.veg` files from VEGAS Pro. No public spec exists. Everything here comes from walking one real file byte-by-byte (`KERNKRAFT400VEGREPLACE.veg`, 117,144 bytes, saved by VEGAS Pro **17.0** per paths inside it). GUIDs and offsets may differ in other versions. **Confirmed** = checked against those bytes; **inferred/unproven** = flagged.

Separate from the `.vep` skill on purpose: `.vep` is plain XML; `.veg` is a binary GUID-chunk container.

---

## 1. Container: Wave64-style, not classic RIFF

Commonly described as "RIFF-based". Real bytes say otherwise — it uses **Wave64 (W64)** framing:

```
0x00  16B  "riff" GUID   726966662e91cf11a5d628db04c10000
0x10   8B  size (u64 LE) — 117144, equals the file size exactly (size INCLUDES the header)
0x18  16B  format GUID   ef29c4464a90d211872200c04f8edb8a
0x28  ...  chunks
```

Every chunk: `16-byte GUID + 8-byte u64 LE size` where **size includes the 24-byte header**. Payload is padded to an **8-byte boundary** before the next chunk (`next = off + size + (8 - size % 8) % 8`).

A chunk with the `"list"` GUID `6c6973742f91cf11a5d628db04c10000` behaves like RIFF `LIST`: its payload **starts with a 16-byte "list type" GUID**, then nested chunks follow. So child chunks begin at `listOffset + 24 + 16`.

Top level of the sample: first a non-list chunk `5a2d8fb20f23d21186af00c04f8edb8a` (size 3728), then nine `list` chunks of sizes 104, 1320, 5416, **102064**, 128, 3584, 40, 192, 528. The 102,064-byte list holds the tracks.

## 2. Timeline structure (confirmed in sample)

All GUIDs below are the 32-hex-char full values.

| Meaning (inferred name) | GUID |
|---|---|
| Track list (list type) | `48076c4d1623d21186b000c04f8edb8a` |
| Events container (list type) | `49076c4d1623d21186b000c04f8edb8a` |
| One Event (list type) | `4a076c4d1623d21186b000c04f8edb8a` |
| Event header (leaf, 208 B) | `6a208d162123d21186b000c04f8edb8a` |
| Event fade (leaf, 128 B) | `6d208d162123d21186b000c04f8edb8a` |
| Take list (list type, 192 B) | `ce3b2bfc8579d211871100c04f8edb8a` |

Hierarchy: big list → **Track list** (×2) → small properties chunk + **Events container** → many **Event** lists (568 B each on track 1, 480 B on track 2) → header + fade + take-list.

Sample result: **2 tracks × 96 events**.

**Pitfall that already bit once:** track lists are nested *inside* the big project list, not at top level. A finder that only scans the top-level chunk array returns 0 tracks. Walk the whole tree recursively.

## 3. Event fields

- Header chunk payload `+16`: **int64 LE** → start position in ticks. **Confirmed real**: monotonically increasing across events, in clean ratio steps (deltas ≈ 3336665/6, 1668333, 1334668 ≈ 0.8×, 6673333 ≈ 4×) — a musical grid.
- Header payload `+40`: float64 ≈ **0.7999997602** on every sampled event. Constant; meaning **unconfirmed** (do not call it clip length).
- Take-list payload `+32` (payload begins after the 16-byte type GUID): float64 = **1.0** on every sampled event. **Inferred** to be a playback-rate field because of position and value, but it **never varied** in the sample, so it is *unproven*. Don't claim per-clip pitch/speed support as verified; a file with stretched clips is needed to prove or kill it.
- Each of the two tracks has the same 96 start ticks (last = 227,226,998).

## 4. Unresolved

- **Ticks → seconds.** The commonly cited 705,600,000 ticks/s gives ~2.4 ms steps for the base unit here, which is not a plausible beat grid. Treat the constant as unverified; show timeline spacing as **relative**, never as absolute time.
- **Audio vs video track.** No GUID found that marks track media type. Sample structure is audio-shaped (a song project). UI should default to Audio with a manual switch, not guess.
- Meaning of header `+40`, the fade chunk contents, the 12 other top-level chunks, take→media linkage.

## 5. Strings and paths

VEGAS stores absolute paths as **plaintext UTF-16LE**. No obfuscation. Sample contains `M:\Projektek\Sony Vegas projektek\KERNKRAFT400VEGREPLACE.veg`, `C:\Users\GIGABYTE\...\VEGAS Pro\17.0\`, and referenced media (`.mp4`, `.wmv`). Paths leak the Windows username → privacy note when sharing.

- Scan for UTF-16LE runs on **2-byte stride**. Odd-offset scan found no accented strings in the sample (strings are even-aligned).
- **Allowed code units: Basic Latin, Latin-1 Supplement (A0–FF), Latin Extended-A/B (100–24F).** An ASCII-only filter truncates `Fütyi`, `Petőfi`. A wide-open "any printable" filter produces CJK/Hangul mojibake from binary noise (e.g. `楲晦鄮ᇏ횥`). Trade-off: real CJK/Cyrillic paths would be missed.
- `füastbement` in `Petőfi sándor füastbement terv.wmv` is **literally stored** (`U+00FC` then `U+0061`). Not a decode bug. Verify by dumping code units before "fixing" a string.

## 6. Thumbnails

**None.** Searched raw bytes and base64, ASCII and UTF-16LE, for PNG/JPEG signatures (`89504E47`, `FFD8FF`, `iVBORw0KGgo`, `/9j/`): zero hits. Unlike `.vep`, VEGAS embeds no preview image in the sample project.

## 7. Placeholders in the reference extractor

Media is only referenced by path, never embedded. Reference build uses red/green/blue test frames for video clips and a synthesized 1000 Hz tone for audio clips, and a compressed-time "Play Timeline" with a speed slider (pitch tied to speed). The slider is a UI control, not file data.

## 8. Verification habits that paid off

- Run the *real page functions* against the real file in jsdom (stub `requestAnimationFrame` and `AudioContext`) instead of re-implementing the logic in a test; that caught the track-nesting bug and proved the 50% speed → 500 Hz math.
- When "it still sounds the same", check for a stale cached copy of the HTML before touching code; the code path was correct.
