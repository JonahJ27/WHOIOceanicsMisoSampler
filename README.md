# WHOI Oceanics DeepSee Sampler — User Guide

This guide is for operating the DeepSee Sampler control program. **You do not
need any programming or computer experience to use it.** Just follow the steps
below in order.

---

## 1. Starting the program

1. Find the file named **`WHOIOceanicsSampler.exe`**.
2. **Double-click it.**
3. Wait a **5-15 seconds**. Two things will happen:
   - A small window titled **"Select COM Port"** appears first.
   - After you finish that step, the main control window opens.

> If Windows shows a blue "Windows protected your PC" box, click
> **More info → Run anyway**. This is normal for programs that are not from the
> Microsoft Store.

---

## 2. Choosing the COM port (connecting to the sampler)

The **COM port** is simply the name your computer gives to the cable that
connects to the sampler. You have to tell the program which one to use.

1. Make sure the sampler is **plugged in via the ethernet cable and powered on** before this step.
2. In the **"Select COM Port"** window, type the port name into the box. The default is **COM8**
3. Click **OK**.

### If you don't know the port name

1. Click the **"Unsure"** button at the bottom of the window.
2. A list of every connected device appears. It looks something like this:

   ```
   COM3  -  USB Serial Port
   COM4  -  Communications Port
   ```

3. The sampler is almost always the one that says **"USB Serial Port"** (or
   similar). You can **highlight the name and copy it** (select it, then press
   Ctrl-C), then paste it into the box with Ctrl-V.
4. Click **OK**.

> **Tip:** If you're not sure which one is the sampler, unplug it, click
> "Unsure" to see the list, then plug it back in and click "Unsure" again. The
> port that **appears the second time** is your sampler.

---

## 3. Using the system

Once the main window opens you'll see 16 pumps and a row of buttons along the
bottom.

### ⚠️ IMPORTANT: Click **Start Mission** first

**Before you press pump button, click the `Start Mission` button.**
Nothing will work correctly until you do this. Do it every time you begin.

### Running the pumps

- Each pump has a green **Start** button and a red **Stop** button.
- Press **Start** to run a pump; press **Stop** to stop it.
- The **State** shows whether the pump is On or Off, and **Volume** shows how
  much it has pumped.

### The bottom row of buttons

- **Stop All** — immediately stops every pump. Use this if anything looks wrong.
- **End Mission** — finishes the session. Click this when you are done.
- **Graph it!** — shows a chart of the volumes collected.
- **Ping** — checks that the sampler is responding.
- The other buttons (Clear EEPROM, Lights Off, Calibrate) are for advanced or
  setup use — you normally won't need them.

### Finishing up

1. Click **Stop All** to be safe.
2. Click **End Mission**.
3. Close the window by clicking the **X** in the top corner. The program shuts
   down the connection cleanly on its own.

---

## 4. Where your data goes

The program automatically saves data into files in the same folder as the
program:

- `volumes_feed_1.csv` — the volumes each pump collected.
- `command_hist_1.csv` — a record of every button you pressed.
- `sampler_Feed_1.csv` — everything the sampler reported back.

You can open these with Excel.

---

## 5. Troubleshooting


**The program won't start after I enter a port:**
- Make sure no other windows of this are running. They will keep control of the port and you won't be able to connect.
- Try some other ports if there aren't too many, maybe the port assignment changed.
- Alternatively, the configuration of your ports may be messed up. This is a more advanced issue so I'll walk through the solution, but it may be worth getting someone with some backgroud: you need to make sure that the Moxa port and the computer are on the same subnet. You will need to go to the ethernet confiuration in settings and make sure it is set manually to the correct subnet (192.168.1.X) and Gateway (255.255.255.0). If you see that it says DHCP, this is wrong. Also make sure the X isn't overlapping with anything important. Pick a number randomly between 220 and 280 for X and it is likely to be fine.

**A pump won't start:**
- Do other pumps still work?
  - Yes: Try toggling that pump on and off a few times with all other pumps off. If it still doesn't work it may be a mechanical or electrical issue
  - No: Make sure you select start mission before touching the pumps. If you didn't, you may have to quit the program and restart it.

**The window is frozen / not responding:**
- Force quit the window with the task manager. Then retry launching the program. If you are now stuck on.

**I clicked X when entering the port and saw an error message:**
- You are ok. This error will not impact anything. Don't fret.

**My computer fell in the ocean**
Get the flashdrive with the software on it. In it you should see a **`WHOIOceanicsSampler.exe`** file. Move that to your computer. You will also need to set up your computer to connect over ethernet to the MISO by configuring port 1 to link to the COM8 port. That protocol is linked on the flashdrive in the **DEVELOPER.md** file.

**My computer fell in the ocean with the flashdrive (Full New Machine Setup)**

Everything is stored online, so you can rebuild from a new **Windows computer**
with internet.

1. In a web browser, go to
   **https://github.com/JonahJ27/WHOIOceanicsMisoSampler**.
2. Click **`WHOIOceanicsSampler.exe`** in the file list, then click **Download**.
   (If the browser warns about the file, choose **Keep**.)
3. Set up the Ethernet connection: on that same GitHub page, open
   **`DEVELOPER.md`** and follow **Section 6 ("Connecting over Ethernet with a
   Moxa NPort")** to install the Moxa software and get the sampler onto **COM8**.
4. Double-click the downloaded **`WHOIOceanicsSampler.exe`**, then continue from
   **Section 1** of this guide as usual (use port **COM8**).

**Other notes:**
- When in doubt, wait 30 seconds to see if things progress, and then restart the computer.
