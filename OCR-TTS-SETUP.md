# OCR + GPU TTS setup (mango-config-dms) — exact recreation guide

What this gives you: `SUPER+X` region-to-text OCR, `SUPER+T` clipboard
spoken aloud by Kokoro (`af_heart:0.4,af_bella:0.6`) on NVIDIA GPU, `SUPER+M`
dictation via Parakeet. Fully offline
after first-run downloads. DMS-only variant — no Noctalia keys anywhere here.

Architecture (hybrid, on purpose): NixOS provides system packages; the Kokoro
engine is a `uv` venv with pinned PyPI CUDA wheels (nix-building onnxruntime
+CUDA takes hours); binds live in this repo's `mango/config.conf`.

---

## Part 0 — NixOS system packages

Add to `environment.systemPackages` (e.g. `modules/packages/system-packages.nix`):

```nix
grimblast tesseract wtype uv python314 portaudio espeak-ng songrec alsa-utils
```

(`grim`, `slurp`, `mpv`, `wl-clipboard`, `curl`, `libnotify`, PipeWire are
assumed present. Replace `fury` below with your username.)

```bash
sudo nixos-rebuild switch --flake /etc/nixos#nixos
which tesseract wtype uv espeak-ng songrec; python3.14 --version
# expect python3.14 >= 3.14.6
```

---

## Part 1 — OCR test (no downloads)

```bash
region=$(slurp) || exit 0; grim -g "$region" - | tesseract stdin stdout -l eng 2>/dev/null | wl-copy; wl-paste | head -c 300
```

Drag a box over text — it echoes back. If this works, the OCR bind will work.

---

## Part 2 — Kokoro GPU engine (uv venv)

```bash
git clone https://github.com/dusklinux/dusky ~/dusky-upstream
cd ~/dusky-upstream/user_scripts/tts_stt/dusky_kokoro
```

### 2a — Create the missing `trigger.sh` (never committed upstream)

Write this file as `trigger.sh` next to `kokoro_installer.sh`:

```bash
#!/usr/bin/env bash
# Minimal trigger.sh shim for NixOS (upstream never committed one).
# dusky_main.py carries every client command (speak/stop/pause/status/
# doctor/voices/reload/unload/shutdown) on stdlib only — this just execs it.
# Installed by kokoro_installer.sh to $TRIGGER_DIR and symlinked as
# ~/.local/bin/dusky-kokoro. Requires CPython 3.14+ (PEP 758 syntax).
set -euo pipefail
HERE=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
MAIN="$HERE/dusky_main.py"
if [[ ! -f "$MAIN" ]]; then
    MAIN="$HOME/contained_apps/uv/dusky_kokoro/dusky_main.py"
fi
# Prefer the venv interpreter (has onnxruntime/kokoro for daemon/doctor/
# synth); pure socket clients work on any 3.14+. NixOS needs the driver
# libs first in LD_LIBRARY_PATH (libcuda) — nix-ld dir covers libstdc++.
VENV_PY="$HOME/contained_apps/uv/dusky_kokoro/.venv/bin/python"
if [[ -x "$VENV_PY" ]]; then
    PY="$VENV_PY"
else
    PY="python3.14"
fi
export LD_LIBRARY_PATH="/run/current-system/sw/share/nix-ld/lib:/run/opengl-driver/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
exec "$PY" "$MAIN" "$@"
```

```bash
chmod +x trigger.sh
```

### 2b — Run the installer (NVIDIA, half-precision GPU model)

```bash
DUSKY_TRIGGER_DIR="$HOME/contained_apps/dusky_kokoro" ./kokoro_installer.sh --hw nvidia --models fp16-gpu -y
```

First run dies at `verify_runtime` with
`ImportError: libstdc++.so.6: cannot open shared object file` — expected on
NixOS (no `/usr/lib`). Re-run with the driver/toolchain libs visible:

```bash
export LD_LIBRARY_PATH="/run/current-system/sw/share/nix-ld/lib:/run/opengl-driver/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
DUSKY_TRIGGER_DIR="$HOME/contained_apps/dusky_kokoro" ./kokoro_installer.sh --hw nvidia --models fp16-gpu -y
```

This downloads ~2.5 GB (CUDA wheels + `kokoro-v1.0.fp16-gpu.onnx` 177 MB +
`voices-v1.0.bin`), writes `~/.config/dusky-kokoro/config.toml`
(`provider = "cuda"`), installs `dusky-kokoro.socket/.service` user units,
and symlinks `~/.local/bin/dusky-kokoro`. The bundled self-test reports
problems due to an upstream bug (next step) — install itself completes.

### 2c — Patch the upstream `run_synth` crash (deployed copy)

`~/contained_apps/uv/dusky_kokoro/dusky_main.py`, in `run_synth`'s `finally:`
block, guard the executor shutdown (`_executor` is `None` outside the
synth-worker, so `doctor --synth` always crashes hiding a good result):

```python
    finally:
        with contextlib.suppress(Exception):
            engine._unload_sync("synth done")
        # NIXOS FIX (upstream bug): _executor is None unless Engine runs as
        # synth-worker; unguarded shutdown() crashes doctor --synth cleanup.
        if engine._executor is not None:
            engine._executor.shutdown(wait=False, cancel_futures=True)
```

### 2d — systemd override so the daemon sees driver libs

Write `~/.config/systemd/user/dusky-kokoro.service.d/nixos.conf`:

```ini
# NixOS: no /usr/lib — driver + toolchain libs must come via LD_LIBRARY_PATH.
[Service]
Environment="LD_LIBRARY_PATH=/run/current-system/sw/share/nix-ld/lib:/run/opengl-driver/lib"
```

```bash
systemctl --user daemon-reload
systemctl --user status dusky-kokoro.socket
```

### 2e — Verify (expect CUDA provider, RTF ~0.14 on GTX 1660S)

```bash
export LD_LIBRARY_PATH="/run/current-system/sw/share/nix-ld/lib:/run/opengl-driver/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
~/contained_apps/uv/dusky_kokoro/.venv/bin/python ~/contained_apps/uv/dusky_kokoro/dusky_main.py doctor --synth
```

Look for `"active_providers": ["CUDAExecutionProvider", "CPUExecutionProvider"]`,
`"voice": "af_heart:0.4,af_bella:0.6"`, `"degraded": false`, and a low `rtf`.

```bash
~/contained_apps/dusky_kokoro/trigger.sh status
~/contained_apps/dusky_kokoro/trigger.sh speak --text "Dusky Kokoro is installed."
~/contained_apps/dusky_kokoro/trigger.sh voices   # all 54 styles
```

Voice default is already the Heart/Bella blend. To change it, edit
`~/.config/dusky-kokoro/config.toml` `[voice] spec`, then
`trigger.sh reload`. Stop/pause: `trigger.sh stop`, `trigger.sh pause`.

---

## Part 3 — Binds in this repo (`mango/config.conf`)

Place after the fullscreen-to-satty line in the screenshots cluster:

```
# Region select to text (OCR extract)
bind=SUPER,x,spawn_shell,region=$(slurp) || exit 0; grim -g "$region" - | tesseract stdin stdout -l eng 2>/dev/null | wl-copy; notify-send "OCR" "$(wl-paste | head -c 200)"
# Speak clipboard aloud (Kokoro Heart/Bella, GPU)
bind=SUPER,t,spawn_shell,wl-paste --no-newline | /home/fury/.local/bin/dusky-kokoro speak --stdin --mode interrupt
```

Use the **absolute** trigger path — mango's `spawn_shell` does not inherit
`~/.local/bin` on PATH (bare `dusky-kokoro` dies silently). Mango hot-reloads
`config.conf`; validate with `mango -p -c ~/.config/mango/config.conf`.

Layer convention: bare `SUPER+X/T/M` are the AI layer (OCR/speak/dictate);
the previous occupants moved up one layer — power-quick to `SUPER+SHIFT+X`,
theme toggle to `SUPER+SHIFT+T`, process list to `SUPER+SHIFT+M`.

Noctalia note: this DMS-only repo must never gain `keymode=noctalia`
sections, `SUPER+ALT+N/D` shell-swaps, or `XF86` Noctalia binds.

---

## Models on disk (what, where, how to re-fetch by hand)

Yes — everything is downloaded to this PC, nothing streams. The installer
fetched them in step 2b; this section documents exact locations so you can
back them up or re-fetch without re-running the installer.

```bash
~/contained_apps/uv/dusky_kokoro/models/kokoro-v1.0.fp16-gpu.onnx  # 170 MB (177,464,787 bytes)
~/contained_apps/uv/dusky_kokoro/models/voices-v1.0.bin           # 27 MB (54 voice styles)
~/contained_apps/uv/dusky_kokoro/.venv/                           # 2.4 GB (onnxruntime-gpu 1.30.0 + CUDA 13.4 wheels + kokoro-onnx 0.6.1)
~/.cache/dusky-kokoro/audio/                                      # spoken output archive (grows over time)
```

Source of truth (pinned in `kokoro_installer.sh` as `RELEASE_BASE`):

```bash
BASE="https://github.com/thewh1teagle/kokoro-onnx/releases/download/model-files-v1.0"
mkdir -p ~/contained_apps/uv/dusky_kokoro/models && cd ~/contained_apps/uv/dusky_kokoro/models
curl -sSL -O "$BASE/kokoro-v1.0.fp16-gpu.onnx"
curl -sSL -O "$BASE/voices-v1.0.bin"
ls -la   # expect 177464787 and 28214398 bytes respectively
```

Verify integrity any time without the daemon:

```bash
export LD_LIBRARY_PATH="/run/current-system/sw/share/nix-ld/lib:/run/opengl-driver/lib"
~/contained_apps/uv/dusky_kokoro/.venv/bin/python ~/contained_apps/uv/dusky_kokoro/dusky_main.py doctor --synth
# expect: model fp16-gpu ok, voices 54 styles, providers include CUDAExecutionProvider
```

NOT on disk (never installed here): Parakeet STT models, Silero VAD, Piper
voices — only the Kokoro TTS engine above was set up.

---

## Part 4 — STT dictation (hyprwhspr-rs + Parakeet, Nix-native)

Unlike TTS this is a real NixOS service, no `uv` venv. Engine: `hyprwhspr-rs`
with NVIDIA Parakeet TDT 0.6B v3 (`istupakov/parakeet-tdt-0.6b-v3-onnx`) on CPU.

### 4a — NixOS module (`ai-stt`)

`modules/programs/ai-stt.nix` (import it, all OFF by default):

```nix
ai-stt.enable = true;   # service + CLI on PATH
# ai-stt.cuda = true;   # NVIDIA build (only if CPU latency bothers you)
```

Notes: module adds you to the `input` group (re-login after first enable),
uses `unstablePkgs.hyprwhspr-rs` 0.3.33 (stable's 0.3.27 has a broken
virtual-keyboard injector + broken sendshortcut Lua on Hyprland 0.55).

```bash
sudo nixos-rebuild switch --flake /etc/nixos#nixos
systemctl --user start hyprwhspr-rs
```

### 4b — Parakeet model download (~2.4 GB, upstream script's file list)

```bash
mkdir -p ~/.local/share/hyprwhspr-rs/models/parakeet/parakeet-tdt-0.6b-v3-onnx
cd ~/.local/share/hyprwhspr-rs/models/parakeet/parakeet-tdt-0.6b-v3-onnx
BASE="https://huggingface.co/istupakov/parakeet-tdt-0.6b-v3-onnx/resolve/main"
for f in encoder-model.onnx encoder-model.onnx.data decoder_joint-model.onnx vocab.txt; do
  curl -sSL -O "$BASE/$f"
done
ls -la   # encoder .data alone is ~2.3 GB
```

### 4c — Config (`~/.config/hyprwhspr-rs/config.jsonc`)

```jsonc
{
  "audio_feedback": true,
  "audio_device": null,   // null = PipeWire default source (set yours with wpctl)
  "transcription": {
    "provider": "parakeet"
  }
}
```

```bash
systemctl --user restart hyprwhspr-rs
journalctl --user -u hyprwhspr-rs --since "2 min ago" | grep -i "parakeet.*ready"
# expect: Parakeet TDT transcription ready
```

Mic check: `wpctl status` → Sources → `*` marks default. Hardware mute
buttons (e.g. USB mic switches) bypass software volume — check the physical
switch first when captures come back empty.

### 4d — Bind in this repo (`mango/config.conf`)

```
# STT record toggle (hyprwhspr-rs, Parakeet)
bind=SUPER,m,spawn,hyprwhspr-rs record toggle
```

`hyprwhspr-rs` is a system package (`/run/current-system/sw/bin`), so the
bare name works here — unlike the Kokoro trigger, no absolute path needed.
Press → start sound → talk 3–5 s → press again → text pastes + clipboard.
Noctalia note: this DMS-only repo stays Noctalia-free; the bind is
keymode-agnostic.

### 4e — Verify + troubleshoot

```bash
hyprwhspr-rs record toggle; sleep 4; hyprwhspr-rs record toggle
journalctl --user -u hyprwhspr-rs --since "3 min ago" | grep -iE "transcribing|inject|clipboard|empty"
```

- `Transcribing Xs` + `Injecting text` → working.
- `Empty transcription, nothing to inject` → mic heard silence: physical mute,
  wrong default source, or you spoke outside the record window.
- Text in clipboard but not in textbox → paste fallback limits: on Hyprland
  0.3.27's sendshortcut is broken (0.3.33 fixed input handling); on Mango
  there is no Hyprland IPC at all, so `Ctrl+V` is the workflow.

---

## Troubleshooting

```bash
journalctl --user -u dusky-kokoro.service --since "15 min ago"   # daemon log
trigger.sh status                                                # socket alive?
nvidia-smi                                                       # VRAM held?
```

- Keypress does nothing, no job in logs → bind PATH issue (use absolute path).
- `libcuda`/`libcudart` import errors → `LD_LIBRARY_PATH` missing driver dir.
- Silence but jobs finish → `mpv`/PipeWire output selection, check `pavucontrol`.
- Daemon holds VRAM at rest → `trigger.sh unload`, or disable warm preload;
  socket-activated idle exit frees everything after ~30 s.
