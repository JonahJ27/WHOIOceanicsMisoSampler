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

## 6. Connecting over Ethernet with a Moxa NPort

Normally the DeepSee sampler plugs into the computer with a USB/serial cable and
shows up as a COM port. A **Moxa NPort** lets you connect that same sampler over
an **Ethernet (network) cable** instead. It's a small box: the sampler's serial
cable plugs into the NPort, and an Ethernet cable runs from the NPort to the
computer. A piece of Moxa software then makes the sampler show up on the
computer as a normal COM port — the DeepSee GUI can't tell the difference.

This walkthrough sets things up so the sampler appears as **COM8**, connected
through **port 1** of an NPort whose network address is **192.168.1.101**.

> **You'll need the NPort's network address (IP).** This guide uses
> `192.168.1.101`. If your NPort is set to a different address, either use that
> address throughout, or change the NPort to `192.168.1.101` (see §6.5).

### 6.1 Is the Moxa software already on the computer? (Usually no)

Windows does **not** come with the Moxa NPort software. On a brand-new machine
you have to install it yourself before COM8 will work. To check whether it's
already there, look in the Windows Start menu for **"NPort Windows Driver
Manager"** or **"NPort Administrator"**. If you don't see them, follow §6.2 to
install. If you do, skip to §6.3.

### 6.2 Install the Moxa NPort software

1. On a computer with internet access, go to **moxa.com** and search for your
   NPort model (it's printed on the label of the box — e.g. *NPort 5150*,
   *5210*, *5610*).
2. Download the **NPort Windows Driver Manager** (it comes inside the **NPort
   Administration Suite** — the free Windows software/driver package for that
   model).
3. Run the downloaded installer and click through with the default options.
   When Windows asks to trust/install the Moxa driver, allow it.
4. When it finishes you should have **NPort Windows Driver Manager** and **NPort
   Administrator** in the Start menu.

*(On a machine with no internet, download the installer elsewhere and copy it
over on a USB stick.)*

### 6.3 Plug everything in

1. Connect the sampler's serial cable into **Port 1** on the NPort.
2. Connect an Ethernet cable from the NPort to the computer (or into the same
   network switch the computer uses).
3. Power on the NPort.

### 6.4 Put the computer on the same network as the NPort

The computer and the NPort have to be on the same network "neighborhood" to talk
to each other. Give the computer a fixed network address that matches the
NPort's:

1. Open **Settings → Network & Internet → Ethernet → Change adapter options**
   (or **Control Panel → Network and Sharing Center → Change adapter
   settings**).
2. Right-click the **Ethernet** connection → **Properties**.
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**.
4. Choose **Use the following IP address** and enter:
   - **IP address:** `192.168.1.100`
   - **Subnet mask:** `255.255.255.0`
   - Leave the gateway blank.
5. Click **OK**.

To confirm the computer can reach the NPort, open **Command Prompt** and type:

```bat
ping 192.168.1.101
```

You should see replies. If it says "Request timed out," re-check the cabling and
the address you typed above.

### 6.5 (Only if needed) Set the NPort's address to 192.168.1.101

Skip this if the NPort is already at `192.168.1.101`. Otherwise:

1. Open **NPort Administrator**.
2. Click **Configuration**, then **Search** — it finds NPorts on the network and
   lists their current addresses.
3. Select your NPort, open the **Network** tab, and set the IP to
   **`192.168.1.101`** with subnet mask **`255.255.255.0`**. Save/apply (the
   NPort may restart).

### 6.6 Make the sampler appear as COM8

1. Open **NPort Windows Driver Manager**.
2. Click **Add**.
3. Let it search the network, or type the NPort's address **`192.168.1.101`**
   by hand.
4. Pick your NPort, then choose **Port 1**.
5. Set the COM port number to **COM8**. (If it suggests COM8 already, accept it;
   otherwise select the entry, click **Port Setting** / **Modify**, and change
   the number to **COM8**.)
6. Click **Apply** / **OK**.

That's it — the sampler is now available as **COM8**, and the setting sticks
even after restarting the computer.

### 6.7 Check that it worked

1. Open the DeepSee GUI.
2. At the port prompt, type **`COM8`** and continue. (The **Unsure** button also
   lists available ports — COM8 should be in the list.)
3. If the sampler connects and data starts flowing, you're done.

### 6.8 If something isn't working

- **`ping` fails / NPort not found:** re-check the Ethernet cabling and that you
  entered the computer's IP exactly as in §6.4. Try **Search** in NPort
  Administrator.
- **COM8 connects but no data appears:** in NPort Administrator, confirm Port 1
  is set to **Real COM** mode, and double-check the sampler's serial cable is in
  Port 1.
- **It says COM8 is already in use:** in NPort Windows Driver Manager, choose a
  different free number, or remove the old COM8 entry first.

---

## 7. Known rough edges / TODO

- Unused imports (`posixpath.split`, `pyparsing.col`) should be removed.
- High COM numbers (`COM10+`) on Windows may need the `\\.\` prefix — consider
  normalizing this inside `open_serial`.
- The port dialog raises `serial.SerialException("No COM port selected!")` if
  closed without a choice; in a packaged `--windowed` build this exits silently.
  Consider showing a `messagebox` instead.
- Heartbeat parsing in `process_serial` assumes fixed offsets
  (`buffer[13:15] == "HB"` and a `ml_Pumped` split) — fragile if the firmware
  message format changes.
