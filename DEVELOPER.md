# DEVELOPER.md — WHOI Oceanics DeepSee Sampler

Technical notes for maintaining, running from source, and packaging the DeepSee
Sampler GUI (`DeepSeeGUI4_FINAL_20230213.py`) into a distributable Windows
`.exe`.

---

## 1. Overview

The application is a single-file Tkinter GUI that talks to the DeepSee sampler
over a serial (COM) port at **38400 baud**.

- **Entry point:** `DeepSeeGUI4_FINAL_20230213.py`
- **GUI toolkit:** Tkinter (standard library)
- **Serial I/O:** `pyserial` (`serial`, `serial.tools.list_ports`)
- **Plotting:** `matplotlib` + `numpy`
- **Concurrency:** a background `SerialThread` reads incoming lines and pushes
  them onto a `queue.Queue`; the Tk main loop drains the queue every 100 ms in
  `App.process_serial`.

### Startup flow

1. `prompt_for_port()` opens a small Tk window asking the user to type the port
   (with an **Unsure** button that lists `serial.tools.list_ports.comports()`
   in a read-only, copyable `Text` widget). Returns the chosen port string.
2. `open_serial(port, 38400)` opens the port with retries.
3. The main `App` window (16 `PumpFrame`s + `OtherButtons`) is built and
   `app.mainloop()` runs.
4. On window close, `App.on_close` stops the thread and closes the port.

### Data / log files (written to the working directory)

- `volumes_feed_1.csv` — volumes from heartbeat (HB) messages only.
- `command_hist_1.csv` — commands sent + ACKs.
- `sampler_Feed_1.csv` — raw dump of everything received.

---

## 2. Running from source

### Prerequisites

- Python 3.8+ (Tkinter is included with the standard python.org installer).

### Setup

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

pip install -r requirements.txt
```

### Run

```bash
python DeepSeeGUI4_FINAL_20230213.py
```

### Cross-platform port note

The serial call is `serial.Serial(rf'{port}', 38400, timeout=timeout)`.

- **Windows:** users type `COM3`, etc. For port numbers **COM10 and above**,
  Windows requires the `\\.\COM10` prefix. If you hit "could not open" on high
  COM numbers, prepend `\\.\` to the port string before opening.
- **Linux/macOS:** ports look like `/dev/ttyUSB0` or `/dev/tty.usbserial-XXXX`.

---

## 3. Dependencies

`requirements.txt` should cover the third-party packages:

```
pyserial
numpy
matplotlib
pyparsing
```

Notes:
- `tkinter`, `queue`, `csv`, `threading`, `time` are standard library.
- `from posixpath import split` and `from pyparsing import col` are unused
  leftovers from earlier edits and can be removed safely.

---

## 4. Packaging into `WHOIOceanicsSampler.exe`

We use **PyInstaller**. **Build on Windows** — PyInstaller does not
cross-compile, so a Windows `.exe` must be produced on a Windows machine (or a
Windows VM/CI runner).

### 4.1 Install the build tool

```bash
pip install pyinstaller
```

### 4.2 One-line build

```bash
pyinstaller --onefile --windowed --name WHOIOceanicsSampler --collect-all matplotlib --hidden-import numpy DeepSeeGUI4_FINAL_20230213.py
```

Flag meanings:

- `--onefile` — bundle everything into a single `.exe` (easiest to hand to
  scientists; slightly slower first launch as it unpacks to a temp dir).
- `--windowed` (a.k.a. `--noconsole`) — no black console window pops up behind
  the GUI. **Omit this while debugging** so you can see tracebacks/`print`s.
- `--name WHOIOceanicsSampler` — produces `dist/WHOIOceanicsSampler.exe`, the
  exact filename referenced in `README.md`.
- `--collect-all matplotlib` — pulls in every matplotlib submodule, data file,
  and dynamic library so graphing works in the packaged build.
- `--hidden-import numpy` — ensures numpy is bundled even though PyInstaller's
  static analysis may not catch every reference.

The finished executable is written to **`dist/WHOIOceanicsSampler.exe`**.

### 4.3 Optional: icon

```bash
pyinstaller --onefile --windowed --name WHOIOceanicsSampler --icon app.ico DeepSeeGUI4_FINAL_20230213.py
```

### 4.4 Reproducible builds with a spec file

After the first run PyInstaller writes `WHOIOceanicsSampler.spec`. Commit it and
rebuild with:

```bash
pyinstaller WHOIOceanicsSampler.spec
```

Edit the spec (not the command line) when you need to add hidden imports or data
files.

### 4.5 matplotlib gotcha

The `--collect-all matplotlib` flag in the build command above already bundles
matplotlib's submodules and data files, which covers most graphing failures. If
graphing still errors on a backend, also force the Tk backend near the top of
the script, before `import matplotlib.pyplot`:

```python
import matplotlib
matplotlib.use("TkAgg")
```

### 4.6 Test the artifact

- Copy `dist/WHOIOceanicsSampler.exe` to a **clean Windows machine without
  Python installed** and confirm it launches, the port dialog appears, and
  serial + graphing work.
- The `.csv` logs are written next to wherever the `.exe` is run from — make
  sure that folder is writable (i.e., not inside `Program Files`).

---

## 5. Distribution checklist

- [ ] Build on Windows with the exact name `WHOIOceanicsSampler.exe`.
- [ ] Test on a clean (no-Python) Windows box.
- [ ] Ship `WHOIOceanicsSampler.exe` together with `README.md`.
- [ ] Confirm the target folder is user-writable for the CSV logs.
- [ ] (Optional) Code-sign the `.exe` to reduce SmartScreen warnings.

---

## 6. Known rough edges / TODO

- Unused imports (`posixpath.split`, `pyparsing.col`) should be removed.
- High COM numbers (`COM10+`) on Windows may need the `\\.\` prefix — consider
  normalizing this inside `open_serial`.
- The port dialog raises `serial.SerialException("No COM port selected!")` if
  closed without a choice; in a packaged `--windowed` build this exits silently.
  Consider showing a `messagebox` instead.
- Heartbeat parsing in `process_serial` assumes fixed offsets
  (`buffer[13:15] == "HB"` and a `ml_Pumped` split) — fragile if the firmware
  message format changes.
