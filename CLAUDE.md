# SH-MDN-BW — Music Data Analysis

## Project layout
All binary audio lives under this folder now (previously the raw file sat one level up); none of it is committed to git — see `.gitignore`.

| Path | Contents | Size | In git? |
|---|---|---|---|
| `raw-source/2025-10-26_0303_raw_recording.wav` | Original unedited recorder output (renamed from `261025_0303.WAV` — see Source file below) | 1.4 GB | No |
| `tracks/` | 276 speech-free slices, Roman-numeral filenames (`I.wav` … `CCLXXVI.wav`) | 992 MB | No |
| `archive/SH-MDN-BW_speech-free-tracks.zip` | Zipped copy of `tracks/` + docs, uploaded to Drive (link below) | 792 MB | No |
| `CLAUDE.md`, `tracklist.txt`, `.gitignore` | Documentation | — | Yes |

## Audio files
The 276 extracted WAV tracks and the raw source recording are **not** stored in the GitHub repo — GitHub isn't suited to hosting binary audio at this scale. The zip of tracks + docs was uploaded manually to Google Drive:

**https://drive.google.com/file/d/1ZkTLlsgdJd4thhfVXe_8gx6T4GBNU_D0/view?usp=drive_link**

The GitHub repo contains only the documentation and tracklist.

## Source file
- **File:** `raw-source/2025-10-26_0303_raw_recording.wav` (renamed from the recorder's original auto-generated name `261025_0303.WAV` for clarity; moved into this project folder — 1.4 GB)
- **Original filename pattern:** `261025_0303` was a date/time stamp — `26-10-25` (26 Oct 2025) at `03:03`, consistent with a field/live recorder auto-naming convention (e.g. Zoom, Tascam, Sound Devices).

## Technical properties
| Property | Value |
|---|---|
| Format | WAV / PCM, 16-bit signed int |
| Channels | 2 (stereo) |
| Sample rate | 44,100 Hz (CD quality) |
| Bit rate | 1,411.2 kbps (uncompressed) |
| Duration | 8,602.6 sec ≈ **2 h 23 m 23 s** |
| File size | 1,517,509,376 bytes (~1.41 GB) |

## Level / loudness analysis (ffmpeg)
- **Mean volume:** −29.4 dB
- **Max (sample) peak:** 0.0 dBFS
- **True peak:** **+1.5 dBTP** — exceeds 0 dBFS, confirming **actual clipping** (inter-sample peaks overshoot full scale, not just touching it)
- **Integrated loudness:** **−26.5 LUFS** (very quiet by streaming standards — Spotify/YouTube normalize to roughly −14 LUFS, so this sits about 12 LU below typical released music)
- **Loudness range (LRA):** 16.6 LU — wide dynamic range, consistent with an unmixed/unmastered live or ambient recording rather than a compressed, radio-ready track
- **Silence detection (threshold −40 dB, min 5 s):** **zero silent gaps found** across the entire 2h23m file — the recording is continuous with no long pauses, consistent with an unedited single take (e.g. a full live set, rehearsal, or ambient/room recording) rather than multiple distinct songs separated by gaps.

## Interpretation
- This is almost certainly a **single continuous raw recording**, not a mixed/mastered music file — the combination of a timestamp-style filename, 2h23m unbroken runtime, 16-bit/44.1kHz PCM (typical recorder default), low integrated loudness (−26.5 LUFS), and wide LRA (16.6 LU) all point to an unprocessed live/field capture rather than a studio or streaming track.
- **Clipping is present** (+1.5 dBTP true peak) — at least one loud moment overshoots 0 dBFS. If this matters for playback/mastering, it should be addressed (true-peak limiting) before any public release.
- No internal track markers or silence breaks were detected, so if this is meant to contain multiple songs/sets, they are **not separated by silence** — splitting would require manual timestamps or transient/onset detection.

## Suggested next steps
- If this is a live set, consider normalizing loudness (e.g. to −16 LUFS) and trimming any clipped peak for a listenable export.

---

## Speech-free music extraction

Ran voice activity detection (VAD) across the full file to isolate every stretch of audio with no detected speech, and exported each as its own lossless file — goal was to preserve as much of the music as possible while guaranteeing no speech leaks into any track.

**Method:**
1. **VAD pass** — `whisper-cpp`'s `whisper-vad-speech-segments` (Silero VAD model, `ggml-silero-v5.1.2.bin`) scanned the full 2h23m file and found **1,014 raw speech segments**, totaling ~33.4 min of detected speech out of 143.4 min total (~23% of the file).
2. **Safety padding + merge** — each raw speech segment was padded by 0.4s on both sides (to avoid clipping word onsets/tails) and overlapping/adjacent padded segments were merged into 536 contiguous "speech blocks."
3. **Complement extraction** — every gap between speech blocks of **≥ 1.0 second** was treated as a music-only candidate. This produced **276 segments**, totaling **~97.9 minutes (68.3% of the file)** preserved as speech-free audio.
4. **Lossless export** — each segment was cut from the original with `ffmpeg -c copy` (no re-encoding — sample-accurate, zero quality loss, same 44.1kHz/16-bit PCM as source) into `tracks/`, named purely with **Roman numerals** in chronological order: `I.wav`, `II.wav`, `III.wav`, … `CCLXXVI.wav`.

**Results:**
- `tracks/` — 276 WAV files, ~992 MB total.
- `tracklist.txt` — full list of all 276 tracks with start/end timestamps (relative to the original recording) and duration.
- Segment lengths vary widely (1.0s up to ~20.8 min) since real pauses-in-speech vs. genuine long music-only stretches differ enormously — this is expected and intentional per the "preserve everything, however small" goal.

**Caveats:**
- VAD can occasionally miss very quiet or heavily music-masked speech, or false-positive on vocal-heavy music (singing) as "speech" — this was not manually verified track-by-track.
- The 0.4s padding is a safety margin, not a guarantee; a few tracks may have a faint partial word at the very edge if speech onset was more gradual than the model's padding assumed.
- Tiny sub-5-second tracks (171 of the 276) are real but very short — they're mostly brief instrumental/musical gaps between spoken phrases, not standalone "songs."

---
*Analysis generated from direct inspection with `afinfo` and `ffmpeg` (`volumedetect`, `silencedetect`, `loudnorm`), plus VAD-based speech detection via `whisper-cpp`. No metadata tags (artist/title/album) were present in the WAV header.*

---

## Dev log (2026-11-02)

**11:23** — Working directory inspected. Contents: a single file, `261025_0303.WAV` (1,517,509,376 bytes / ~1.41 GB). No other project files, no git repo.

**11:23–11:35** — Ran format/level analysis on the WAV with tools already on the machine:
- `afinfo 261025_0303.WAV` → 2 ch, 44100 Hz, Int16 PCM, duration 8602.646349 sec.
- `ffmpeg -af volumedetect` → mean_volume -29.4 dB, max_volume 0.0 dB.
- `ffmpeg -af silencedetect=noise=-40dB:d=5` → **no output** — zero silence gaps ≥5s anywhere in the file, i.e. the recording is one continuous unbroken take.
- `ffmpeg -af loudnorm=print_format=summary` → launched as a background task (full two-pass loudness scan of a 2h23m file isn't realtime even at ~5-19x speed); returned ~29 min later: Integrated −26.5 LUFS, True Peak +1.5 dBTP (confirms clipping), LRA 16.6 LU.

**11:57** — Created `SH-MDN-BW/` folder per instruction.

**~11:58** — Wrote first version of `CLAUDE.md` with the format/level findings and interpretation (single continuous raw/field recording, not a mastered track, true-peak clipping present).

**12:00** — User asked for a tracklist of music-only slices (no speech) and exported files per slice, prioritizing preserving as much of the recording as possible.
- Checked for speech-detection tooling: no `whisper`/`whisper-cpp` CLI on `PATH`, no relevant Python packages (`torch`, `pyannote`, `silero`, etc.) installed.
- Found `whisper-cpp` 1.9.1 already installed via Homebrew (`brew list whisper-cpp`), including a purpose-built binary: `whisper-vad-speech-segments` (Silero VAD, no transcription needed — faster and sufficient for speech/non-speech classification).
- That binary requires a VAD model file not bundled by the brew formula. Downloaded `ggml-silero-v5.1.2.bin` (~885 KB) from `https://huggingface.co/ggml-org/whisper-vad/resolve/main/ggml-silero-v5.1.2.bin` into a temporary `SH-MDN-BW/models/` folder (removed after use — see cleanup below).

**12:01–~12:03** — Ran `whisper-vad-speech-segments` across the full WAV (backgrounded + monitored to completion). Raw result: **1,014 speech segments**, timestamps in centiseconds. Parsed with a small Python script (`re` + manual unit conversion, centiseconds confirmed by cross-checking the final segment end (860179 cs ≈ 8601.79s) against the known 8602.6s file duration).

**12:03** — Gap-distribution analysis on the raw 1,015 gaps between/around speech segments: median 0.86s, p75 1.99s, p90 6.11s, p99 69.82s. This confirmed most gaps are natural micro-pauses between words/phrases, not real musical breaks — informed the merge-threshold decision below.

**Design decision** (made directly, not asked of user, since it's an implementation detail): to avoid producing thousands of near-zero-length fragments from normal speech pauses, raw VAD segments were padded ±0.4s (safety margin against clipping word onsets/tails) and merged if overlapping/touching, collapsing 1,014 raw segments into **536 speech blocks**. Only gaps between blocks of **≥1.0s** were kept as music candidates — this produced exactly **276 segments**, ~97.9 min (68.3% of the file).

**Asked user, got answers:**
- Output format → **lossless WAV** (not FLAC/MP3).
- Naming scheme → **pure Roman numerals** (`I.wav`, `II.wav`, … — no `track_` prefix, after an initial follow-up correction from the user).

**12:04** — Extracted all 276 segments with `ffmpeg -ss <start> -i <src> -t <dur> -c copy` (stream copy, no re-encode → sample-accurate, zero quality loss) into `SH-MDN-BW/tracks/`. All 276 reported `OK`. Verified on disk: 276 files, 992 MB total.

**12:04** — Generated `tracklist.txt`: one row per track with Roman-numeral name, start/end timestamp (`HH:MM:SS.ss`, relative to the original recording), and duration.

**12:05** — Cleanup: removed intermediate working files no longer needed — `vad_raw_output.txt` (raw VAD stdout dump), `speech_segments_raw.json`, `music_segments.json` (intermediate Python data), `extract_log.txt` (per-track ffmpeg status log), and the temporary `models/` folder (VAD model binary, not needed after extraction finished). Final `CLAUDE.md` rewritten to document the extraction methodology and caveats.

**Caveats carried forward (unverified):** VAD may miss speech buried under loud music, or misclassify singing as speech; the 276 tracks have not been manually spot-checked for leakage at segment edges.

**12:08** — User asked to clean up project file naming ("raw files etc, figure it out"). Reviewed the full tree and made two changes:
- `261025_0303.WAV` → `SH-MDN-BW/raw-source/2025-10-26_0303_raw_recording.wav` — moved the raw file from the parent directory into the project folder and gave it a self-describing name (was the recorder's cryptic auto-generated name; date/time meaning was previously only documented in prose, not encoded in a readable filename).
- `SH-MDN-BW-tracks.zip` (sitting loose in the parent directory) → `SH-MDN-BW/archive/SH-MDN-BW_speech-free-tracks.zip` — consolidated into the project folder under a clearly-labeled `archive/` subfolder.
- Updated `.gitignore` to also exclude `raw-source/` and `archive/` (both large binaries, same reasoning as `tracks/`).
- Left `tracks/I.wav` … `CCLXXVI.wav` untouched — that pure-Roman-numeral scheme was an explicit user decision earlier in the project, not an oversight.
- Did not rename the enclosing `Desktop/untitled folder` directory itself — it's the active working directory for this session, and renaming it out from under an in-progress session risks breaking path references; flagged to user as a separate decision rather than done silently.
