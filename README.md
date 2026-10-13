# ESP32-P4 Live2D Convert Tools

A standalone desktop app that converts **Live2D Cubism models** into the compact
C arrays / binary data files used by the **ESP32-P4 Live2D renderer**. It bakes
the model, textures, motions and voice clips on your PC, so the P4 itself never
needs the (closed-source, x86/ARM-only) Cubism Core library.

---

## Downloads

| Platform | File |
|---|---|
| macOS | `ESP32-P4 Live2D Convert Tools - macOS.zip` |
| Windows | `ESP32-P4 Live2D Convert Tools - Windows.zip` |
| Linux | `ESP32-P4 Live2D Convert Tools - Linux.zip` |

The OS name is part of the app name so you always know which build you have.

### Installing

- **macOS** — unzip, then double-click the `.app`.
  If macOS refuses to open it ("unidentified developer"), **right-click → Open →
  Open** once, or run:

  ```bash
  xattr -dr com.apple.quarantine "ESP32-P4 Live2D Convert Tools - macOS.app"
  ```

- **Windows** — unzip and double-click the `.exe`. If SmartScreen warns you,
  click *More info → Run anyway*.
- **Linux** — unzip, then `chmod +x` the executable and run it.

No Python, no Pillow/numpy, no C compiler needed — everything is bundled.

---

## Quick start

<img width="1320" height="848" alt="App view" src="https://github.com/user-attachments/assets/65787b48-7bc7-46e4-978f-06f3bfd5df14" />

1. **Import** — drag your whole Live2D model folder onto the drop zone, or click
   **Choose model folder…** / **Choose .moc3 file…**.
   The app auto-detects the `.moc3`, up to three `texture_*.png` pages and every
   `*.motion3.json` (the `*_idle*` motion becomes the standby animation).
2. **Convert and preview** — click **Convert and preview**. The character appears
   in the preview pane on the right; click or drag on it to poke the head/body and
   see the touch reactions. The current FPS is shown in the status bar.
   - **Texture size** = the exported texture edge length (`256` / `512` / `1024`).
   - **Preview resolution** only affects preview speed, never the exported data.
3. **Pick motions / audio** — the middle list shows every motion
   (`Motion N` + filename + the group name from `.model3.json` + duration) and
   every audio file. Untick anything you do not want to ship.
4. **Export to…** — choose a destination and the app writes a
   `<model>_data/` folder (see below). Copy that whole folder to the SD card.

---

## Interface reference

### Conversion options

| Option | Meaning |
|---|---|
| **Texture size** | Edge length of the exported textures: `256`, `512` or `1024`. **Must match `TEX_SZ` in the firmware**, otherwise the model renders as garbage. |
| **Preview resolution** | Preview render size only (`Smooth 120x200`, `Fast 240x400`, `Standard 480x800`). Does not change the export. |
| **Language** | `中文` / `English` / `日本語`. The choice is remembered in `~/.live2d_converter.json`. |

### Motions / Audio list

The **Export** column is the checkbox — clear it to leave that motion or audio
file out of the export.

| Column | Contents |
|---|---|
| **Kind** | Motion or Audio. |
| **Content** | Motion number + filename, or the audio filename. |
| **Info** | Motion: group name and duration. Audio: whether the device can play it as-is (`16000 Hz / mono / 16-bit`) or whether it will be converted on export. |

Extra controls:

- **Play: Loop / Once** — the playback mode used for the motion you select.
- Click any `Motion N` row to play it immediately in the preview using firmware
  semantics (index 0 is the looping idle, the rest are tap motions).
  **Back to idle** returns to the default standby motion.
- **Play audio** — audition the selected clip in your system player.
- **Include audio when exporting** — untick to export a completely silent model.
- **Auto preview** — when on, selecting a motion that has a `Sound` entry in
  `.model3.json` plays that clip automatically.
- **Select all / Select none / Motions only / Audio only** — bulk toggles.

### Preview pane

- Click / drag on the character to apply head and body touch feedback.
- **Fit to window** and **Zoom** control the view.
- The preview and the texture pane are **drag-resizable**: grab the divider
  between the panels to change their widths.

### Textures / Parts pane

- **Show texture highlight** — toggles the texture inspector on the right.
- **Highlight active parts** — outlines, on each texture page, the regions that
  the *currently playing* pose actually samples, so you can see which artwork is
  in use in real time.
- **Highlight all used texture areas** — highlights every area the model can ever
  use, instead of only the current pose.
- The part names (arm, body, head, hair, …) of the highlighted regions are listed
  underneath.
- **Save PNG…** — export the current highlight view as an image.

---

## Export layout

Exporting produces one folder at your chosen destination:

```
<model>_data/
├── <model>_data.c            # model + texture C arrays — compile into the firmware
├── <model>_data.h
└── data/                     # read from the SD card at runtime
    ├── <model>.l2d           # baked model (vertices, UVs, deform weights)
    ├── <model>.l2d.parts     # drawable -> part map
    ├── <model>.motion        # baked motion curves, pose groups, voice table
    ├── <model>_tex.rgb565    # texture page 0, RGB565
    ├── <model>_tex.alpha     # texture page 0, 8-bit alpha
    ├── <model>_tex1.rgb565   # page 1 (only if the .moc3 uses a 2nd page)
    ├── <model>_tex1.alpha
    ├── <model>_tex2.rgb565   # page 2 (only if the .moc3 uses a 3rd page)
    ├── <model>_tex2.alpha
    └── sounds/               # the model's own WAV clips (if any)
        └── *.wav
```

Notes:

- The **`data/` level must be kept** — the firmware builds its paths as
  `/sdcard/<model>_data/data/<model>.l2d`.
- Voice clips live in **`data/sounds/`**, i.e. next to the `.l2d` file, because
  the firmware derives the sound directory from the `.l2d` path.
- Only the texture pages the `.moc3` actually references are exported; extra
  `texture_NN.png` files are ignored with a log message.
- Audio is only usable on the device if it is **16-bit PCM WAV**. Anything else
  is listed but skipped, with a warning in the log. Non-16 kHz or stereo WAVs are
  converted to 16 kHz mono during export.
- A motion whose audio you unticked is exported silently (it is removed from the
  voice table as well, so the firmware never looks for a missing file).

### Deploying to the board

1. Copy the whole `<model>_data/` folder to the **root of the SD card**.
2. Add `<model>_data.c` to the firmware build (drop it into
   `main/live2d/<model>_data/`).
3. Set `TEX_SZ` in `main/live2d_manager.c` to the **same value** you picked in
   *Texture size*.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Textures look scrambled or torn | `TEX_SZ` in the firmware does not match the **Texture size** used at export. Re-export at 512 or change `TEX_SZ`. |
| No sound on the device | The clip is not 16-bit PCM WAV, or *Include audio when exporting* was off, or the `sounds/` folder was not copied into `data/`. |
| A motion is silent although it has audio | Its audio row was unticked; the motion is exported without a voice entry. |
| "No texture PNG found" | The model folder has no `texture_*.png` next to the `.moc3`. |
| Motion preview says "please convert again" | The preview needs the `.l2d` from a completed conversion — run **Convert and preview** first. |
| The app will not start on macOS | Right-click → Open once, or clear the quarantine flag with `xattr` as shown above. |

---

## Building the app from source

Requires Python 3 with `tkinter`, plus `pillow`, `numpy`, `tkinterdnd2` and
`pyinstaller`, and a C compiler to build the model baker.

```bash
# from the repository root
python -m PyInstaller --noconfirm --clean \
  --distpath dist --workpath build \
  tools/live2d_converter_gui.spec
```

The bundle name automatically includes the target OS
(`ESP32-P4 Live2D Convert Tools - macOS`, `… - Windows`, `… - Linux`).
Set `L2D_APP_ARCH=1` to also append the CPU architecture
(`ESP32-P4 Live2D Convert Tools - macOS-arm64`).

---


