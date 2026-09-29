# OCR + GPU TTS setup (mango-config-dms) — exact recreation guide

What this gives you: `SUPER+SHIFT+X` region-to-text OCR, `SUPER+SHIFT+T` clipboard
spoken aloud by Kokoro (`af_heart:0.4,af_bella:0.6`) on NVIDIA GPU. Fully offline
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
bind=SUPER+SHIFT,x,spawn_shell,region=$(slurp) || exit 0; grim -g "$region" - | tesseract stdin stdout -l eng 2>/dev/null | wl-copy; notify-send "OCR" "$(wl-paste | head -c 200)"
# Speak clipboard aloud (Kokoro Heart/Bella, GPU)
bind=SUPER+SHIFT,t,spawn_shell,wl-paste --no-newline | /home/fury/.local/bin/dusky-kokoro speak --stdin --mode interrupt
```

Use the **absolute** trigger path — mango's `spawn_shell` does not inherit
`~/.local/bin` on PATH (bare `dusky-kokoro` dies silently). Mango hot-reloads
`config.conf`; validate with `mango -p -c ~/.config/mango/config.conf`.

Noctalia note: this DMS-only repo must never gain `keymode=noctalia`
sections, `SUPER+ALT+N/D` shell-swaps, or `XF86` Noctalia binds.

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
