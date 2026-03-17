# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

WhisperTux is a Linux voice dictation app that records audio, transcribes via whisper.cpp (offline), and injects text into the focused application using ydotool/xdotool. It uses hardware-level keyboard shortcuts (evdev) so it works across X11, Wayland, and TTYs.

## Running the App

```bash
# Via launcher
./whispertux

# Direct (with optional debug output)
python3 main.py [--debug]
```

## Setup / Installation

```bash
# One-command automated setup (installs deps, builds whisper.cpp, configures services)
python3 setup.py

# Individual setup scripts
bash scripts/prepare-system.sh      # system packages
bash scripts/build-whisper.sh       # compile whisper.cpp (CMake)
bash scripts/setup-ydotoold-service.sh
bash scripts/fix-uinput-permissions.sh
```

The setup creates a virtualenv at `./venv/`. whisper.cpp lives in `whisper.cpp/` as a submodule.

## Architecture

**`main.py`** — monolithic GUI + orchestration (~1720 lines). `WhisperTuxApp` owns all components and wires them together. `SettingsDialog` is also here.

**`src/` modules** — each is independent and can be tested in isolation:
- `audio_capture.py` — sounddevice/PortAudio, 16kHz mono float32, callback-based with circular buffer
- `whisper_manager.py` — subprocess wrapper around whisper.cpp binary; converts NumPy→WAV, parses stdout
- `text_injector.py` — ydotool primary, xdotool fallback; handles punctuation words ("period" → "."), word overrides
- `global_shortcuts.py` — evdev-based hardware keyboard capture; background thread with select multiplexing; 500ms debounce
- `config_manager.py` — reads/writes `~/.config/whispertux/config.json`
- `waveform_visualizer.py` — matplotlib embedded in tkinter, ~35 FPS
- `logger.py` — Rich-based colored CLI output

**Recording flow:**
1. evdev detects shortcut → `_on_shortcut_pressed()` in main
2. `AudioCapture` streams via sounddevice callback
3. On stop: NumPy buffer → `WhisperManager` → whisper.cpp subprocess → text
4. `TextInjector` sends text to focused app via ydotool
5. GUI updated with transcription

## Key Configuration

User config stored at `~/.config/whispertux/config.json`. Defaults:
- `primary_shortcut`: `F12`
- `model`: `base`
- `injection_tool`: `xdotool`
- `push_to_talk`: `false` (hold vs. toggle mode)
- `language`: `en`
- `key_delay`: `15` (ms)
