## SpleeterGui 3.0 — Music Source Separation Desktop App
Windows desktop front-end for AI-powered music source separation.
Isolate and mute instruments in any song. Create custom backing tracks in seconds with AI.
Aísla y mutea instrumentos en cualquier canción. Crea pistas de acompañamiento personalizadas en segundos con IA.

> **Version 3.0** is a complete rewrite. The old Spleeter + TensorFlow engine has been replaced by **audio-separator 0.44** running on **Python 3.13** with **PyTorch CPU**, supporting modern Mel-Band Roformer and HTDemucs models.

---

## What's New in v3.0

### Engine Replacement
| | v2.x (old) | v3.0 (new) |
|---|---|---|
| Separation engine | Spleeter 2.4 | audio-separator 0.44 |
| Python | 3.10 embeddable | 3.13 embeddable |
| ML framework | TensorFlow / Keras | PyTorch CPU |
| Primary model | `2stems` (TF format) | `model_bs_roformer_ep_317_sdr_12.9755.ckpt` |
| 4-stem model | `4stems` | `htdemucs.yaml` (HTDemucs v4) |
| 5-stem model | `5stems` | `htdemucs_6s.yaml` (HTDemucs 6-stem) |
| FFmpeg | 4.x (2019) | 7.x (2026, shared build) |
| Installer | NSIS | Inno Setup 6 |
| Input formats | mp3, wav, ogg, m4a, wma, flac | All FFmpeg formats (audio + video) |

### Portable Bundle Structure (`bin/Release/`)
```
bin/Release/
├── python313/          # Python 3.13 embeddable
├── lib/                # All packages (audio_separator, torch, numpy, etc.)
│   ├── numpy.libs/     # OpenBLAS DLLs (delvewheel bundle)
│   └── torch/lib/      # PyTorch DLLs
├── pretrained_models/  # Model cache (.ckpt files)
├── languages/          # 10 language XML files
├── run_separator.py    # Portable audio-separator CLI wrapper
├── hybrid_pipeline.py  # Hybrid pipeline (3-step karaoke extraction)
├── setup_portable.py   # Bundle creation script
├── ffmpeg.exe          # FFmpeg N-123858 (2026-04-07, shared build)
├── ffprobe.exe
├── ffplay.exe
└── avcodec-62.dll      # + 6 other shared DLLs for FFmpeg
```

---

## Hybrid Pipeline — Lead Vocal Extraction

The hybrid pipeline solves a fundamental limitation: standard 2-stem models group **all voice content** (lead + backing vocals/choruses) into a single stem. The hybrid mode separates them.

### Pipeline Steps

```
Audio Input
    │
    ├── Step 1: Mel-Roformer V4
    │   model_bs_roformer_ep_317_sdr_12.9755.ckpt
    │   └──► vocals.wav  +  accompaniment.wav
    │
    ├── Step 2: Mel-Roformer Karaoke (applied to vocals.wav only)
    │   mel_band_roformer_karaoke_aufr33_viperx_sdr_10.1956.ckpt
    │   └──► lead_vocals  +  backing_vocals
    │
    └── Step 3: FFmpeg mix
            ├── accompaniment.wav              (pure instrumental, no voices)
            ├── vocals.wav                     (full vocal track with choruses)
            ├── {song}_Vocal_Sin_Coros.wav     (lead vocal only)
            ├── {song}_Coros_Aislados.wav      (isolated backing vocals / choruses)
            └── {song}_Instrumental_Con_Coros.wav  (instrumental + backing vocals)
```

### Hybrid Mode Checkboxes (GUI)
| Checkbox | Output file |
|---|---|
| ☑ Instr. sin Coros | `accompaniment.wav` (always generated in step 1) |
| ☑ Coros Aislados | `{song}_Coros_Aislados.wav` |
| ☑ Instr. + Coros | `{song}_Instrumental_Con_Coros.wav` |
| ☑ Vocal sin Coros | `{song}_Vocal_Sin_Coros.wav` |
| ☑ Vocal con Coros | `vocals.wav` (always generated in step 1) |

---

## Models

| Model file | Used for | Auto-download |
|---|---|---|
| `model_bs_roformer_ep_317_sdr_12.9755.ckpt` | All 2-stem separations + hybrid step 1 | Yes (~800 MB) |
| `mel_band_roformer_karaoke_aufr33_viperx_sdr_10.1956.ckpt` | Hybrid step 2 (karaoke split) | Yes (~800 MB) |
| `htdemucs.yaml` | 4-stem separation | Yes |
| `htdemucs_6s.yaml` | 5-stem separation (vocals, drums, bass, guitar, piano, other) | Yes |

Models are downloaded automatically on first use by audio-separator and cached in `pretrained_models/`.

---

## Supported Input Formats

All formats that FFmpeg can demux are accepted via file dialog and drag & drop:

| Category | Extensions |
|---|---|
| Audio | `.mp3` `.wav` `.flac` `.aac` `.m4a` `.ogg` `.opus` `.wma` `.aiff` `.aif` `.ape` `.mka` `.mp2` `.ac3` `.dts` `.amr` |
| Video | `.mp4` `.mkv` `.avi` `.mov` `.webm` `.flv` `.ts` `.wmv` `.m2ts` `.mts` `.vob` `.3gp` `.m4v` |

For video files, FFmpeg extracts the audio internally before passing it to the separation model.

---

## Localization — 10 Languages

All UI controls are translated via XML files in `languages/`:

`en.xml` · `es.xml` · `fr.xml` · `de.xml` · `it.xml` · `pt.xml` · `ru.xml` · `ja.xml` · `zh.xml` · `ar.xml`

New strings added in v3.0:
- `chkHybridMode` — "Hybrid Mode (Lead Vocal + Chorus)"
- `chkHybridSinCoros` — "Instr. no Chorus"
- `chkHybridCorosAisl` — "Isolated Chorus"
- `chkHybridConCoros` — "Instr. + Chorus"
- `chkHybridVocalSinCoros` — "Vocal w/o Chorus"
- `chkHybridVocalConCoros` — "Vocal + Chorus"

---

## Technical Notes

### DLL Loading (Windows / Portable)
`run_separator.py` calls `os.add_dll_directory()` **before** any imports to register all DLL-containing subdirectories (including `lib/numpy.libs/` with OpenBLAS). This is required for Python `.pyd` compiled extensions to find their native dependencies.

### FFmpeg (v7 shared build)
The bundled FFmpeg N-123858-g0510aff11b-20260407 is a shared build. The following DLLs must remain alongside the executables:
`avcodec-62.dll`, `avdevice-62.dll`, `avfilter-11.dll`, `avformat-62.dll`, `avutil-60.dll`, `swresample-6.dll`, `swscale-9.dll`

### Portable Bundle Creation (`setup_portable.py`)
Automates copying packages from a conda/pip environment into `lib/`:
1. Copies all packages from the source `site-packages`
2. Copies compiled `.pyd` files
3. Copies `*.libs` directories (e.g. `numpy.libs/` with bundled OpenBLAS DLLs)

---

## Version History

| Date | Version | Notes |
| ----: |:-------:| ----- |
| 2026-04-16 | **3.0** | Complete engine rewrite: audio-separator + PyTorch + Python 3.13. Hybrid pipeline (lead vocal + chorus separation). All FFmpeg formats supported (audio + video). Inno Setup 6 installer. 10-language support. FFmpeg 7.x. Multiple crash fixes. |
| 7/10/2023 | 2.9.5 | Rebuilt with Python 3.10.10 and Spleeter 2.4. Updated GUI. New website spleetergui.com |
| 5/04/2022 | 2.9.2 | Upgraded Spleeter to 2.3.0. Updated Python files. |
| 30/01/2021 | 2.9.1 | Upgraded Spleeter to 2.1.2. Updated command syntax. |
| 9/11/2020 | 2.9 | Upgraded Spleeter to 2.0.1 and Python. |
| 31/07/2020 | 2.8 | Upgraded the project to 64-bit. |
| 19/07/2020 | 2.7 | Updated help, set paths for python/ffmpeg. |
| 4/07/2020 | 2.6 | Recombine audio and multi-lingual update. |
| 10/05/2020 | 2.5 | UI update. Additional help menu items. |
| 4/05/2020 | 2.4 | Bug fix: "full bandwidth" mode. |
| 27/12/2019 | 2.3 | Accessibility update. |
| 24/12/2019 | 2.2 | New Windows MSI installer. Drag and drop. |
| 21/12/2019 | 2.0 | Interface update, batch processing. |
| 17/12/2019 | 1.1 | Added High quality/expert mode. |

---

![SpleeterGUI_app](/Spleeter_GUI.png)

This project is an open-source C# desktop front-end.  
Please consider donating to help pay for hosting and development: https://www.paypal.com/donate/?hosted_button_id=DVGCZBQZCAFYU

- Website: https://github.com/mananpa4/SpleeterGui-Installer
- Older versions: https://makenweb.com/#spleetergui
