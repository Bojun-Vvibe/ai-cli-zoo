# sox

> **"Swiss Army knife of sound processing":** read, write, convert,
> and apply DSP effects to audio files (and live devices) from one
> 30-year-old C binary. The de-facto reference for command-line
> audio. Source mirror at <https://github.com/chirlu/sox>; licensed
> under **GPL-2.0** ([LICENSE.GPL](https://github.com/chirlu/sox/blob/master/LICENSE.GPL))
> for the `sox` binary and **LGPL-2.1**
> ([LICENSE.LGPL](https://github.com/chirlu/sox/blob/master/LICENSE.LGPL))
> for `libsox`. Last upstream release **14.4.2** (2015); actively
> packaged by every major distro and Homebrew.

Source: <https://github.com/chirlu/sox> (community mirror of the
SourceForge upstream at <https://sox.sourceforge.net>).

## TL;DR

`sox` is a single command-line tool that knows how to read and
write ~30 audio formats (WAV, AIFF, FLAC, Ogg Vorbis, Opus, MP3
via LAME, raw PCM, μ-law, A-law, AU, CAF, …), apply ~50 DSP effects
(resample, trim, fade, normalize, compand, equalizer, reverb,
tempo, pitch, silence-detect, spectrogram, …), and chain them in
left-to-right pipelines on the command line. It also ships two
sibling binaries — `play` (decode + send to default audio device)
and `rec` (capture from default device) — and a `soxi` metadata
inspector. The whole pipeline is a deterministic, scriptable
in/out box: input file(s) → effect chain → output file (or device).
No GUI, no project file, no plugin marketplace.

## Install

```bash
# Homebrew (macOS)
brew install sox
# with MP3 + LAME support compiled in
brew install --with-lame sox 2>/dev/null || brew install sox lame

# Linux package managers
# Debian / Ubuntu: apt install sox libsox-fmt-all
# Fedora:          dnf install sox
# Arch:            pacman -S sox
# Nix:             nix-env -iA nixpkgs.sox

# from source (autotools)
git clone https://github.com/chirlu/sox.git
cd sox && autoreconf -i && ./configure && make -j && sudo make install

# verify
sox --version    # SoX v14.4.2
```

On Debian-family systems install `libsox-fmt-all` to pick up the
optional format plugins (FLAC, Vorbis, Opus, MP3); the base `sox`
package only ships uncompressed formats.

## License

- **`sox` binary:** GPL-2.0 — see
  [LICENSE.GPL](https://github.com/chirlu/sox/blob/master/LICENSE.GPL).
  Distributing modified `sox` binaries triggers source-disclosure
  obligations.
- **`libsox` library:** LGPL-2.1 — see
  [LICENSE.LGPL](https://github.com/chirlu/sox/blob/master/LICENSE.LGPL).
  Linking `libsox` into a closed-source application is permitted
  if you allow library replacement (dynamic linking is the
  straightforward path).

The dual licensing is intentional: end-user CLI is copyleft, the
embeddable library is permissive-enough for proprietary apps.

## One Concrete Example

A speech-pipeline pre-processing chain — convert arbitrary input,
trim silence, normalise loudness, resample for an ASR model, and
emit a spectrogram for QA.

```bash
# 1. inspect what you have
soxi recording.m4a
# Input File     : 'recording.m4a'
# Channels       : 2
# Sample Rate    : 44100
# Precision      : 16-bit
# Duration       : 00:03:42.18 = 9799542 samples
# File Size      : 4.21M
# Bit Rate       : 152k
# Sample Encoding: AAC

# 2. format conversion: m4a -> 16-bit mono 16 kHz WAV (ASR-ready)
sox recording.m4a -r 16000 -c 1 -b 16 recording.wav

# 3. one pipeline: trim leading/trailing silence + normalise + resample
sox recording.m4a -r 16000 -c 1 -b 16 cleaned.wav \
    silence 1 0.1 -50d reverse silence 1 0.1 -50d reverse \
    norm -1

# 4. concatenate three takes into one file (gapless)
sox take1.wav take2.wav take3.wav combined.wav

# 5. mix three stems down to a stereo mix-bus, with per-stem gain
sox -m -v 0.7 vocal.wav -v 0.4 guitar.wav -v 0.5 drums.wav mix.wav

# 6. split a long recording at every 2-second silence into chapter files
sox podcast.wav chapter.wav silence 1 0.5 1% 1 2.0 1% : newfile : restart

# 7. apply a parametric EQ + light compression for podcast loudness
sox raw.wav podcast.wav \
    highpass 80 \
    equalizer 3000 1.0q +3 \
    compand 0.3,1 6:-70,-60,-20 -5 -90 0.2

# 8. generate a PNG spectrogram for visual QA
sox cleaned.wav -n spectrogram -o cleaned.png -t "cleaned.wav spectrogram"

# 9. live record from the default mic to a 24-bit FLAC, stop on Ctrl-C
rec -c 1 -b 24 -r 48000 capture.flac

# 10. play any input on the default device (decoder + driver in one binary)
play recording.m4a trim 30 10    # play 10 s starting at 0:30
```

Effect chains compose left-to-right; `--combine` selects how
multiple inputs merge (`concatenate` / `mix` / `merge` / `sequence`).
Every command is a pure function of (inputs, effect chain) → output,
which makes `sox` ideal as a stage in a `Makefile` or a dataset-
preparation pipeline.

## Niche It Fills

**Deterministic, scriptable audio I/O + DSP for offline pipelines.**
GUI editors (Audacity, Adobe Audition, Logic) own the audio data
inside a project file and require a human at the mouse. Modern
ML-audio libraries (`librosa`, `torchaudio`) require a Python
runtime, dependencies, and a script for every operation.
`ffmpeg` covers transcoding masterfully but its DSP effect
language is comparatively narrow. `sox` sits in the gap: a tiny
binary you can call from `bash`, `make`, a CI job, or an LLM
agent, with a vocabulary of audio-engineering primitives
(silence-detection, companding, parametric EQ, reverb, vocoder,
spectrogram) that no other CLI matches in breadth.

## Why use it

1. **Audio-engineering vocabulary as CLI verbs.** `silence`,
   `compand`, `equalizer`, `gain -n` (normalize), `loudness`,
   `tempo` (time-stretch without pitch shift), `pitch` (pitch
   shift without tempo change), `vad` (voice-activity-detect
   trim), `synth` (generate tones / noise), `spectrogram` —
   each is a documented effect, not a flag on a generic
   transcoder. `man sox` is a 100-page DSP reference.
2. **Reproducible across decades.** The format and effect
   semantics have been stable since the 14.x series; a `sox`
   pipeline written in 2012 still produces a byte-identical
   output today on the same input. For dataset preparation
   (ASR / TTS / music-IR research) this matters more than
   incremental new features.
3. **Pairs cleanly with `ffmpeg`, not against it.** Common
   pattern: `ffmpeg` for container / codec wrangling
   (demux, transcode video-with-audio, hardware-accelerated
   decode), pipe raw PCM into `sox` for the actual DSP, then
   pipe back to `ffmpeg` to remux. `sox -t raw -r 48000 -e
   signed -b 16 -c 2 - … -t raw -r 48000 -e signed -b 16 -c 2
   -` makes that bridge trivial.

For LLM-driven workflows, `sox` is a tool surface where every
verb has a one-line synopsis, output is one file (or stdout),
and exit code reflects success — which is what an agent needs to
chain audio steps without screen-scraping a GUI.

## Vs Already Cataloged

- **Vs the broader media stack we don't yet catalog (`ffmpeg`,
  `mediainfo`):** `ffmpeg` is the king of containers / codecs /
  video; `sox` is the king of audio DSP. They complement each
  other — most production pipelines use both, with `ffmpeg` on
  the boundary and `sox` doing the audio-domain work in the
  middle. Neither is a replacement for the other.
- **Vs Python audio libs (`librosa`, `torchaudio`,
  `soundfile`):** those require a Python interpreter, NumPy,
  and (for `torchaudio`) PyTorch. `sox` is a single 1 MB binary
  with no runtime. For a one-shot resample-and-trim in a
  shell script or a Dockerfile RUN line, `sox` saves the
  ~600 MB of PyTorch deps.
- **Vs GUI editors:** GUIs are right when a human needs to
  *listen and decide*. `sox` is right when the operation is
  known in advance and needs to apply to 10 000 files in a
  loop, or to fit in a `Makefile`, or to run on a server with
  no display.

## Caveats

- **Upstream is in maintenance, not active development.** The
  last tagged release is 14.4.2 (2015). Distros carry small
  patch sets; the [`chirlu/sox`](https://github.com/chirlu/sox)
  fork collects them. Bugfixes land slowly. New format support
  (e.g., recent Opus revisions) lags `ffmpeg` by years.
- **MP3 + AAC support is conditional.** `sox` historically
  could not ship MP3 encoders due to patent encumbrance; even
  now, MP3 read/write requires `libmad` / `libmp3lame` linked
  in at build time, and AAC (`.m4a` / `.aac`) is unsupported in
  most distro builds — fall back to `ffmpeg | sox` for those
  formats. Homebrew on macOS includes MP3; Debian does not by
  default.
- **GPL-2.0 on the binary.** Bundling `sox` into a proprietary
  shipped product (a desktop app, a closed-source server
  appliance) requires either re-licensing impossibility or
  invoking the binary out-of-process and shipping source on
  request. For internal pipelines and ML data prep this is a
  non-issue; for productized redistributions it is.
- **Effect-chain syntax is positional and quiet.** Effects
  apply left-to-right after the output filename, with no
  delimiter; `sox in.wav out.wav gain -3 norm -1` and
  `sox in.wav out.wav norm -1 gain -3` produce different
  audio, and a misplaced flag silently becomes a different
  effect's argument. Read `man sox` once before composing a
  long chain, and prefer one effect per shell-continuation
  line for review.
