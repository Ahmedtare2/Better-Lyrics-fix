# Better Lyrics

An Android app that turns **a song + its lyrics into word-by-word synced TTML**, entirely on the phone. No server, no account, no upload.

It works for **Arabic** and **English** (and songs that mix both), and the output opens in the Better Lyrics Composer / Apple Music-style TTML players.

> The app is built by GitHub Actions. You do not need Android Studio: run the workflow, download the APK from **Releases**, install it.

---

## What it does

1. **Separates the vocals** from the song with HTDemucs (ONNX).
2. **Aligns every word** to the vocal stem with wav2vec2 CTC models (ONNX): one for Arabic, one for English.
3. **Writes word-level TTML**, including background vocals such as `(Hey, girl)`, with per-word timing, singer (agent) tags and a built-in alignment log.

Lyrics can come from an LRC file, plain text, LRCLIB, or an existing TTML. Line timestamps from an LRC are used as hints, not as truth.

### How the alignment gets its accuracy

| Stage | What it does |
|---|---|
| Song-wide CTC check | One forced alignment of *all* lines over the whole vocal stem, softly tied to the LRC stamps (±30 s search band). Finds a constant offset in the stamps and moves lines whose stamps are clearly wrong. |
| Per-line alignment | Each line is aligned inside its own window; repairs fix swallowed lines, late lines, crushed words and line-to-line transitions. |
| Arbitration | Every line is compared with the song-wide result; the one that fits the audio better wins, with sanity guards against jumps that nothing else supports. |
| Background words | Trailing background blocks are placed behind the main line they echo. |
| Onset snapping | Word boundaries are snapped to vocal onsets and checked for overlaps. |

Every decision is written into an XML comment at the top of the exported `.ttml` (`Better Lyrics … alignment log`). If a line looks wrong, that log says what the aligner did and why.

Measured on three benchmark songs against hand-refined references (offline replay of the song-wide stage with the shipped models): 86–98 % of lines within 0.4 s. Real results on a phone vary with the stem quality and the lyrics. See [`docs/alignment-accuracy-changes.md`](docs/alignment-accuracy-changes.md) for the per-round numbers and known limits.

---

## Get the APK

1. Open the **Actions** tab → **Build Better Lyrics APK** → **Run workflow**.
2. Wait for the run to finish (the first run downloads the models, so it takes longer).
3. Open the **Releases** page and download the newest `BetterLyrics-*.apk`.

Options on the **Run workflow** form:

| Option | Meaning |
|---|---|
| `model_zip_url` | Optional HTTPS link to a models ZIP. Anything missing is downloaded automatically. |
| `arabic_fp32` | Keep the Arabic CTC model in full FP32. Usually slightly better timing, about 280 MB larger, needs more RAM. |
| `run_checks` | Run unit tests and lint first. Failures never block the APK. |

The workflow also runs on every push that changes code (not on `*.md` or `docs/` changes). The latest 3 builds are kept as Releases.

> The repository can hold the project either unpacked or as a single `Better-Lyrics-GitHub-Ready-*.zip` at the top level. The workflow unpacks the zip automatically.

### Install

- The APK is large (about 400 MB, the models are inside). Make sure you have at least **1.5 GB free**.
- Compare the downloaded size with the one in the release notes. If it differs, the download is incomplete: download again (use a browser, not a download manager).
- If Android says *Invalid package* after an update, uninstall the old app first.
- Debug builds are signed with a fixed key (`app/debug-shared.keystore`), so new builds install over old ones.
- Requires **Android 8.0 (API 26)** or newer. A phone with 6 GB+ RAM is recommended.

---

## Use it

1. Open the app and pick the **audio** file.
2. Load the **lyrics** (LRC with timestamps gives the best result).
3. Start the alignment. A foreground notification shows progress; a 4-minute song takes a few minutes on a phone.
4. Check the **Preview** tab, fix anything in the **Composer** tab, then export the **TTML**.

Tips for better results:

- Use an LRC whose text matches what is actually sung (repeats, ad-libs and `(background)` parts in brackets).
- One main singer per song works best. Songs with several singers skip the song-wide stamp check.
- If a line is off, open the alignment log at the top of the exported TTML first.

---

## Repository layout

```text
.github/workflows/
  build-and-import.yml     # the build: toolchain, models, APK, GitHub Release
  import-zip.yml           # optional: import model assets from a ZIP into the repo
BetterLyricsAndroid/       # (or the same files at the repo root / inside the zip)
  app/src/main/java/org/betterlyrics/android/
    alignment/             # CTC forced alignment, song-wide check, benchmark
    dsp/                   # audio decoding, STFT, HTDemucs separation
    domain/ttml/           # TTML model, reader and writer
    data/lyrics/           # LRC / text parsers, LRCLIB client
    service/               # foreground alignment service, model manager
    ui/                    # Jetpack Compose UI
    export/                # audio / TTML export
  app/src/test/            # unit tests (TTML, DSP parity, song-wide CTC)
  docs/                    # accuracy notes and benchmark description
```

---

## Models

All models run on-device with ONNX Runtime. The workflow downloads pinned revisions and verifies their SHA-256:

| Purpose | Model |
|---|---|
| Vocal separation | [`StemSplitio/htdemucs-onnx`](https://huggingface.co/StemSplitio/htdemucs-onnx) |
| Arabic alignment | [`onnx-community/wav2vec2-base-arabic-without-reused-en-token-ids-v1-ONNX`](https://huggingface.co/onnx-community/wav2vec2-base-arabic-without-reused-en-token-ids-v1-ONNX) |
| English alignment | [`onnx-community/wav2vec2-base-960h-ONNX`](https://huggingface.co/onnx-community/wav2vec2-base-960h-ONNX) |

Check each model's page for its license before redistributing a build.

---

## Build locally (optional)

```bash
# JDK 17 and the Android SDK are required; the models must be in app/src/main/assets/models
./gradlew :app:assembleDebug
# -> app/build/outputs/apk/debug/app-debug.apk
```

## Troubleshooting

| Problem | What to do |
|---|---|
| "Invalid package" on install | Re-download in a browser, check the exact file size, free 1.5 GB, uninstall the old app. |
| Build has no APK | Open the failed run's log; the workflow fails with an `::error::` line naming the missing piece. |
| A few lines are seconds off | Read the alignment log at the top of the TTML. Send it together with the TTML when reporting. |
| Alignment is slow or the app is killed | Close other apps; do not use the `arabic_fp32` build on phones with little RAM. |
