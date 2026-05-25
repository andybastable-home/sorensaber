# Soren's Combat Saber — Model Kappa (Base-lit RGBX)

Sound/config files for a Combat Saber Model Kappa. The board reads an SD card laid out in a specific structure; this directory mirrors what gets copied onto that card.

## Directories

- `factory-backup/` — pristine factory copy taken off the SD card before any edits. **Do not modify.** Use as reference for original wav format, file names, config defaults.
- `soren/` — Soren's working copy. This is what gets copied to the SD card.
- `factory-backup/config_backup.ini` — duplicate of the original `setting/config.ini`, kept at the top level as a safety net.

## SD card layout (what the board expects)

```
<root>/
  1/  fontconfig.ini  +  ~65 wav files     (font slot 1)
  2/  fontconfig.ini  +  ~65 wav files     (font slot 2)
  ...
  N/  fontconfig.ini  +  ~65 wav files     (font slot N, contiguous from 1)
  setting/  config.ini  +  system wav files (UI sounds, blade effects, etc.)
```

**Font folders MUST be numbered contiguously starting at 1.** Named folders (e.g. `Anakin/`) are ignored by the board — that's the original mistake that caused "no sounds, won't switch fonts."

`setting/` must contain a file literally named `config.ini` (not `config.tmp` or anything else).

## Audio format requirements

Every wav file (font folders AND `setting/`) must be:

- **Codec:** PCM signed 16-bit little-endian (`pcm_s16le`)
- **Sample rate:** 44100 Hz
- **Channels:** mono (1)
- **Container:** RIFF wav

Verify a file:
```bash
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bits_per_sample -of default=noprint_wrappers=1 'path/to/file.wav'
# expected: pcm_s16le, 44100, 1, 16
```

Convert anything else (mp3, stereo wav, 48kHz, etc.) to the target format:
```bash
ffmpeg -y -i input.mp3 -ac 1 -ar 44100 -acodec pcm_s16le 'output (1).wav'
```

The `-ac 1` downmixes to mono, `-ar 44100` resamples, `-acodec pcm_s16le` sets the codec.

Scan the whole tree for non-conforming wav files:
```bash
cd soren && find . -name '*.wav' -print0 | while IFS= read -r -d '' f; do
  cn=$(ffprobe -v error -show_entries stream=codec_name -of default=noprint_wrappers=1:nokey=1 "$f" 2>&1 | tr -d '\r\n')
  if [ "$cn" != "pcm_s16le" ]; then echo "NOT-PCM: $f -> $cn"; fi
done
```

## fontconfig.ini format

One line per font folder. Format documented in `factory-backup/setting/config.ini` near the bottom:

```
NAME=(R,G,B),A,B,C,D,E,F,G,H
```

- `NAME` — display name (free text, shown by app/voice)
- `(R,G,B)` — default blade color, 0-255 each
- `A` — hum effect: 0=fire, 1=steady, 2=unstable, 3=rainbow, 4=candy, 5=crack, 6=pulse, 7=flashing
- `B` — blaster effect: 0/1/2
- `C` — force effect: 0/1
- `D` — lockup effect: 0
- `E` — flash-on-clash effect: 0/1/2
- `F` — style: 0=standard, 1=velocity, 2=torch, 3=blaster, 4=ghost, 5-11=special pre-on
- `G` — on-speed (ms-ish), 200 default, higher = slower
- `H` — off-speed, 500 default, higher = slower

Example: `Luke ROTJ=(0,255,80),1,0,0,0,0,0,125,400` — green steady blade, default effects, slightly fast on, default off.

## Wav file conventions inside a font folder

Files use a `name (N).wav` pattern with a space before the parenthesis. The board picks one randomly when there are multiple variants. Common files:

- `font (1).wav` — font name announcement (played when cycling to this font). Missing this = silent when selected.
- `hum (1).wav` — looping idle hum
- `in (1..N).wav` / `out (1..N).wav` — ignite / retract
- `swing (1..16).wav`, `swingh (1..N).wav`, `swingl (1..N).wav` — swings (high/low pitch)
- `clash (1..16).wav` — clash
- `blaster (1..N).wav` — blaster deflect
- `force (1..N).wav` — force effect
- `lock (1).wav`, `beginlock (1).wav`, `endlock (1).wav` — lockup
- `melt (1).wav`, `beginmelt (1).wav`, `endmelt (1).wav` — melt
- `drag (1).wav`, `begindrag (1).wav`, `enddrag (1).wav` — drag
- `stab (1..N).wav` — stab
- `spin (1..N).wav` — spin (optional)
- `fontconfig.ini` — required

Not every folder needs every file — the board falls back gracefully. But `font (1).wav`, `hum (1).wav`, and basic `in/out/swing/clash` are essentially mandatory for the font to feel right.

## Loudness normalisation for font (1).wav

All `font (1).wav` intro announcements must be normalised to **-17 LUFS integrated** (matching slot 1 Ahsoka as the reference). Source files are often 10–20 dB quieter than this — always normalise, never skip it.

Measure the actual loudness with ebur128 (more accurate than loudnorm's built-in measurement):

```bash
ffmpeg -i 'soren/N/font (1).wav' -filter:a ebur128=peak=true -f null - 2>&1 | grep "I:" | tail -1
```

Normalise by computing the exact gain needed and applying it with a peak limiter:

```bash
# 1. Measure
measured=$(ffmpeg -i 'input.wav' -filter:a ebur128=peak=true -f null - 2>&1 | grep "I:" | tail -1 | sed 's/.*I:\s*\([-0-9.]*\).*/\1/')
# 2. Compute gain
gain=$(python3 -c "print(-17 - ($measured))")
# 3. Apply
ffmpeg -y -i 'input.wav' -ac 1 -ar 44100 -acodec pcm_s16le \
  -filter:a "volume=${gain}dB,alimiter=limit=0.98:attack=5:release=50:level=false" \
  'soren/N/font (1).wav'
```

Do NOT use `loudnorm` alone — its single-pass gain estimation is inaccurate for very quiet sources (can miss the target by 5+ dB). The ebur128 → volume approach is reliable.

After normalising, verify: the output should read between -19 and -16 LUFS.

## Current Soren font assignments

| # | Folder name (was) | fontconfig name | Color |
|---|---|---|---|
| 1 | Ahsoka white     | Ahsoka Tano White   | white       |
| 2 | Anakin           | Anakin Skywalker    | blue        |
| 3 | cal kestis       | Cal Kestis Cyan     | cyan        |
| 4 | Dooku            | Count Dooku         | red         |
| 5 | Kenobi           | Obi-Wan Kenobi      | blue        |
| 6 | Luke ROTJ        | Luke ROTJ           | green       |
| 7 | Maul             | Darth Maul Red      | red         |
| 8 | Palpertine       | Emperor Plapatine   | red         |
| 9 | Plo Koon         | Plo Koon Orange     | orange      |
| 10 | Qui-Gon         | Qui-Gon Jinn        | green       |
| 11 | Revan (purple)  | Revan Purple        | purple      |
| 12 | Revan (Red)     | Darth Revan Red     | red         |
| 13 | Vader           | Vader Hallway       | red         |
| 14 | Windu           | Mace Windu          | purple      |
| 15 | Yoda            | Yoda                | green       |
| 16 | Ezra Bridger    | Ezra Bridger        | blue        |
| 17 | Kanan Jarrus    | Kanan Jarrus        | blue        |

`soren/_unused_chosen/` — the original empty `1/` folder (only had a "The Chosen" fontconfig.ini, no wavs). Kept as a safety preserve; can be deleted before copying to SD card (or just left — the board ignores non-numeric folders).

## Adding a new intro / replacing a wav

When given a downloaded clip (mp3 or anything else) to use as a font's intro:

1. Identify which font slot (e.g. Kenobi = folder 5).
2. Convert and place as `font (1).wav` inside that folder:
   ```bash
   ffmpeg -y -i 'input.mp3' -ac 1 -ar 44100 -acodec pcm_s16le 'soren/5/font (1).wav'
   ```
3. If replacing an existing file, the overwrite (`-y`) just works.
4. Verify: `ffprobe ...` on the new file (see above).

## `setting/config.ini` (system config)

Top-level board settings: button count, blade pixel count, motion sensitivities, default font on boot (`current_font=N`), preon timings, blade modes. Soren's only non-default tweak is `clash_sensitivity=1.5` (vs original `2.3` — easier to trigger clash). The trailing `FONT1=...FONTN=...` lines mentioned in the comments are not present in either config and don't appear to be required — per-font settings live in each folder's `fontconfig.ini`.

## Syncing local ↔ SD card (IMPORTANT)

**The saber firmware writes back to `fontconfig.ini` on the SD card when Soren tweaks colors/effects on the saber itself.** This means the SD card can hold more recent state than `soren/` on disk. A blind `cp -r soren/* /e/` would silently overwrite his on-saber tweaks.

Always **diff before pushing**, and prefer pulling SD changes back into local when they're newer:

```bash
cd /c/UnitySrc/git-personal/sorensaber/soren && for n in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17; do
  local_cfg=$(cat "$n/fontconfig.ini" | tr -d '\r\n')
  sd_cfg=$(cat "/e/$n/fontconfig.ini" 2>/dev/null | tr -d '\r\n')
  if [ "$local_cfg" != "$sd_cfg" ]; then
    echo "$n DIFFERS"
    echo "  SD:    $sd_cfg"
    echo "  local: $local_cfg"
  fi
done
```

For each diff, decide direction (pull = preserve Soren's saber tweak, push = override with the local edit you just made), then `cp -v` the chosen direction. After targeted edits, only push the affected files — never blanket-copy.

When syncing audio (e.g. new `font (1).wav` intros): same pattern, but audio files are unlikely to be modified by the firmware, so push is usually safe.

Also: **the saber caches state**. After updating fontconfig.ini on the card, eject + power-cycle the saber to make changes take effect — otherwise the old color/effect will appear to persist.

## Saber-side persistent overrides

The saber stores per-font customizations in its **onboard flash memory**, not on the SD card. These OVERRIDE `fontconfig.ini` defaults whenever they exist. There is **no button combo to reset a single font's color back to the SD-card default.**

Triggers: any time Soren uses the on-saber color-change combo (hold power + twist hilt downward, clockwise then counter-clockwise), the picked color is saved per-font and persists across SD card edits.

Diagnosis tell: two fonts whose fontconfig.ini both say `(255,0,0)` displaying *different* colors on the saber → it's saved overrides, not a file/format issue.

Fixes:

1. **Xeno Configurator Bluetooth app — "one key restore factory settings"** wipes all per-font color overrides at once. Saber has `bluetooth=1` in config.ini. Cleanest option, but also resets volume / sensitivity / motion / current_font (anything stored only in saber memory). Soren's SD-card config tweaks (e.g. `clash_sensitivity=1.5`) survive because they're on the card, not in saber memory.

   **WARNING — factory reset is also destructive to the SD card** (observed 2026-05-16): the reset writes back to the SD too. Several `fontconfig.ini` files got clobbered to `XENO=(...)` factory defaults (slots 1, 8, 10 in our case — slot 8's XENO blue is why Palpatine showed up as a blue saber). Other slots had hum mode reverted. Worst: `setting/config.ini` got renamed to `config.tmp` mid-write and never finished — the board would have failed to boot config until restored. **Always re-sync local → SD after any factory reset via the app**, and verify `setting/config.ini` exists (not `config.tmp`).
2. **Manual re-cycle per font:** hold power + twist hilt down → "Color Change" → rotate hilt to cycle colors → click power → "Color Selected". Sets a new override (doesn't restore fontconfig.ini default) but at least picks the right color. Per font, fiddly.

Hum/effect changes made on the saber (button-hold cycles) write back to `fontconfig.ini` on the SD. **Colour changes also write back** (observed 2026-05-16: Luke's `(0,255,80)` was rewritten to `(8,255,0)` on the SD after on-saber use). Previous documentation here was wrong — assume *any* on-saber tweak can mutate the SD's `fontconfig.ini`.

If local `fontconfig.ini` differs in colour from the on-saber displayed colour, **trust the local file as the desired state**. Pushing it sets the fontconfig default; if a flash override is in place the saber will still show the old colour until the override is cleared (manual on-saber re-cycle preferred over Xeno app factory reset — see SD-write caveat above).

## Common gotchas

- **Folder names matter.** The board only reads numeric folders. Any named folder (`Yoda/`, `Anakin/`) is invisible to it.
- **`config.tmp` is not `config.ini`.** A partially-saved or wrongly-named config file = no boot config. **This happens routinely** (observed 2026-05-16, multiple sessions): the firmware writes `config.tmp`, truncates `config.ini` to 0 bytes, and frequently fails to complete the rename. After *any* on-saber session that writes config (volume change, font switch, factory reset, etc.) check `/e/setting/` and expect to push local `config.ini` + delete the orphan `config.tmp`.
- **Empty font slot = silent or confused.** A folder with only `fontconfig.ini` and no wavs will appear as a slot but have nothing to play.
- **Stereo or 48kHz wavs may play wrong or not at all.** Always convert.
- **Filename pattern is `name (N).wav` with a space before `(`.** Without the space the board may not find the file.
- **Saber may cache fontconfig at boot.** If a color change isn't taking effect, eject the SD and power-cycle the saber.
- **Firmware writes back to fontconfig.ini.** See "Syncing local ↔ SD card" above — never blanket-copy in either direction without diffing first.
