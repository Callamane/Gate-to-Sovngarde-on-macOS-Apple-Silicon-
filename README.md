# Gate to Sovngarde on macOS (Apple Silicon)

A complete, tested guide to installing and running the **Gate to Sovngarde** Wabbajack modlist (1,961 mods / 2,212 plugins) on an Apple Silicon Mac using CrossOver.

This works. It is also genuinely involved — this guide exists because I did it the hard way over one very long night, hit every wall, and wrote down what actually mattered. Follow it in order and you'll skip all of that.

---

## Honest expectations (read this first)

**What you get:** Gate to Sovngarde running on macOS, full screen, with good framerates in most areas. It is a real, playable, beautiful Skyrim.

**What you don't get:**

- **Community Shaders does not work.** It's disabled in this guide. GtS's advanced lighting/visual layer cannot render through Mac's D3D→Metal translation (see [Dead Ends](#dead-ends-save-yourself-the-time)). The game still looks great — you're running 1,900+ mods of textures, meshes, weather, and world overhauls — but not GtS's *intended* shader stack.
- **Perfectly smooth 120fps everywhere.** Dense cities and heavy combat will dip. This isn't a config you're missing; it's the cost of running a 2,200-plugin Windows game through three translation layers (Rosetta → Wine → Metal) on an engine whose main loop is single-threaded. Background apps on your Mac make this dramatically worse — see [Performance](#part-8-performance-tuning).

**Hardware note:** This was built and tested on an M5 Max / 128 GB. More RAM doesn't help much — GPU VRAM usage sits around 1.5 GB. **Single-core performance and free CPU headroom are what matter.**

---

## Requirements

| Requirement | Notes |
|---|---|
| **Apple Silicon Mac** | M1 or newer. Tested on M5 Max. |
| **CrossOver** (licensed) | v26.x. This is a paid app — there's no free path, Whisky is discontinued. |
| **Parallels Desktop** (licensed) | Needed **only** to run Wabbajack. See [Part 3](#part-3-build-the-modlist-in-a-windows-vm). |
| **Windows 11 VM** | Inside Parallels. |
| **Skyrim Special Edition** on Steam | You must own it. |
| **Nexus Mods Premium** | Strongly recommended — without it, Wabbajack requires a manual click per file across hundreds of mods. |
| **~350 GB free disk** | Modlist is ~110 GB installed, plus downloads, plus a duplicate during transfer. |

---

## Why this takes two tools

The single most important thing to understand up front:

> **Wabbajack itself will not run under CrossOver.** It depends on WebView2 for the Nexus login, which doesn't work in Wine. But **Mod Organizer 2 and Skyrim run *better* in CrossOver than in a VM.**

So the architecture is:

```
1. Parallels Windows VM  →  run Wabbajack, build the modlist
2. Copy the finished modlist  →  into your CrossOver bottle
3. CrossOver  →  run Mod Organizer 2 + Skyrim (this is where you play)
```

You only need the VM once, for the build.

---

## Part 1: Prepare the CrossOver bottle

**Use a dedicated bottle.** Don't install this into a bottle with your other games — modlists want isolation, and you'll be setting DLL overrides you don't want bleeding into other titles.

1. In CrossOver, create a new bottle: **Windows 10, 64-bit**. Name it something like `Skyrim-Modding`.

2. **Install VC++ 2015–2022 x64** into the bottle. Download [vc_redist.x64.exe](https://aka.ms/vs/17/release/vc_redist.x64.exe) and run it in the bottle (CrossOver → Run Command). Silent install:
   ```
   vc_redist.x64.exe /install /quiet /norestart
   ```

3. **Set the `concrt140` DLL override to native.** This prevents crash-fix mods (Buffout-style) from failing. In CrossOver → Wine Configuration → Libraries, add `concrt140` and set to **Native (Windows)**.

   Or via command line:
   ```bash
   CX="$HOME/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/bin"
   "$CX/cxstart" --bottle "Skyrim-Modding" reg add \
     "HKCU\\Software\\Wine\\DllOverrides" /v concrt140 /d native /f
   ```

4. **Enable MSync** in the bottle's Advanced Settings (performance).

---

## Part 2: Install Skyrim SE in the bottle

1. Install **Steam** into the bottle (CrossOver's app installer, or run `SteamSetup.exe`).
2. Log in and install **Skyrim Special Edition**.
3. **Launch the game once** to let it initialize, then quit.

> Steam must be running whenever you play — Skyrim needs it for DRM.

---

## Part 3: Build the modlist in a Windows VM

Wabbajack runs here, not on the Mac side.

### 3.1 — Disk space

Gate to Sovngarde needs **a lot** of local disk in the VM: roughly 110 GB installed plus ~90 GB of downloads. A default Windows VM won't have room.

**Recommended:** add a second virtual disk to the VM rather than resizing the boot drive. With the VM shut down:

```bash
prlctl set "Windows 11" --device-add hdd --size 409600 --iface sata
```

Boot Windows, then in **Disk Management**: initialize (GPT) → New Simple Volume → format NTFS. You'll get a fresh drive (e.g. `E:`).

Create folders on it:
```
E:\Modlists
E:\Downloads
```

### 3.2 — Install Skyrim SE in the VM too

Yes, twice. Wabbajack builds the modlist *against* a real Skyrim install, so the VM needs its own copy. Install Steam in Windows, install Skyrim SE (put it on `E:` to save boot-drive space), and launch it once.

### 3.3 — Run Wabbajack

1. Download **Wabbajack** from [wabbajack.org](https://www.wabbajack.org/) into the VM (e.g. `C:\Modding\Wabbajack\`).
2. Launch it and log into Nexus. (WebView2 ships with Windows 11, so this just works here — this is the whole reason for the VM.)
3. Select **Gate to Sovngarde**.
4. Set **Installation Location** to `E:\Modlists\GTSAV`
5. Set **Download Location** to `E:\Downloads`

> **Both paths must be on a real local drive.** Do not point Wabbajack at a Parallels shared folder — its extractors and BSA tooling will fail or corrupt.

6. Start the install and go do something else. This takes hours.

---

## Part 4: Move the modlist to the Mac

Once Wabbajack finishes, you need the installed modlist folder on the Mac side. It's ~110 GB and ~230,000 files.

**Do not drag 230k files across a shared folder** — it's brutally slow and error-prone. Archive it first, inside the VM:

```powershell
# In the VM. Excludes downloads — you don't need them to play.
tar -cf E:\GTSAV.tar -C E:\Modlists\GTSAV --exclude=downloads --exclude=__temp__ .
```

Then copy `GTSAV.tar` to the Mac (shared folder is fine for a single large file), and extract it into your bottle:

```bash
BOTTLE="$HOME/Library/Application Support/CrossOver/Bottles/Skyrim-Modding"
mkdir -p "$BOTTLE/drive_c/Modlists/GTSAV"
tar -xf /path/to/GTSAV.tar -C "$BOTTLE/drive_c/Modlists/GTSAV"
```

The modlist must end up at a **local path inside the bottle** — `C:\Modlists\GTSAV` from Windows' perspective.

### Fix the paths

MO2 stores absolute Windows paths from the VM. Open `ModOrganizer.ini` in the modlist folder and make sure every path points at the bottle's location (`C:\Modlists\GTSAV\...`), not the VM's old `E:\` paths.

Also create `Game Root\steam_appid.txt` containing:
```
489830
```

---

## Part 5: The critical shader compiler fix

**This is the single most important fix in this guide.** Without it you'll get:

```
ERROR: 3417 shaders failed to compile. Check installation and CommunityShaders.log
```

**Cause:** CrossOver's built-in reimplementation of `d3dcompiler_47.dll` (Microsoft's HLSL compiler) can't handle modern shaders. Every single one fails.

**Fix:** use the real Microsoft DLL. Conveniently, **Mod Organizer 2 already ships one** in the modlist folder.

```bash
BOTTLE="$HOME/Library/Application Support/CrossOver/Bottles/Skyrim-Modding"
G="$BOTTLE/drive_c/Modlists/GTSAV"
CX="$HOME/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/bin"

# Copy MO2's genuine Microsoft compiler next to the game executable
cp "$G/dlls/d3dcompiler_47.dll" "$G/Game Root/d3dcompiler_47.dll"

# Tell Wine to prefer it over its own
"$CX/cxstart" --bottle "Skyrim-Modding" reg add \
  "HKCU\\Software\\Wine\\DllOverrides" /v d3dcompiler_47 /d "native,builtin" /f
```

Verify it's ~4.7 MB (the real one), not ~200 KB (Wine's stub).

---

## Part 6: Disable Community Shaders

Community Shaders cannot render correctly through any Mac translation layer currently available. Symptoms if you leave it on: **black skin/faces, black sky, broken lighting** — the game is unplayable.

This is a known, documented limitation (it uses AVX instructions and compute-shader features Wine on Apple Silicon doesn't expose).

In **Mod Organizer 2**, uncheck:

- `Community Shaders`
- `Upscaling - Community Shaders`
- `CS Light`

Or edit `profiles/<profile name>/modlist.txt` and change the `+` to `-` on those lines. (MO2 caches this — press **F5** to refresh if it's open.)

Everything else in the list stays on. You keep all the textures, meshes, world overhauls, quests, and gameplay mods.

---

## Part 7: Graphics backend and display config

### Backend: use DXMT

In your bottle's **Advanced Settings → Graphics**, select **DXMT**.

Why: DXMT writes its compiled shader cache **to disk permanently**. D3DMetal does not — meaning with D3DMetal, *every single launch* recompiles the entire shader set (a 10–15 minute black screen, every time). With DXMT you pay that once.

### Display: borderless + upscale

Edit `SSEDisplayTweaks.ini` (in the modlist's SKSE plugins folder):

```ini
Fullscreen = false
Borderless = true
BorderlessUpscale = true
SwapEffect = flip_discard
Resolution = 2056x1329
EnableVSync = true
```

**Why not exclusive fullscreen:** it forces a display-mode change, which on a high-DPI Mac panel produces a badly zoomed image, and it captures your cursor so hard that a hang leaves you unable to escape. Borderless + upscale renders at your chosen resolution and scales up to fill the screen — full screen appearance, Dock hidden, one cursor, no display-mode switch.

`Resolution` is your **render** resolution. Lowering it is your biggest single FPS lever, since it upscales to fill the screen regardless.

---

## Part 8: Performance tuning

### The most important thing: free up CPU

Skyrim's main loop is **single-threaded**. It needs one performance core with nothing else on it. If you have AI apps, Electron apps, or sync daemons running, they camp on those cores and you get stutter — no graphics setting fixes this.

**Before playing:**
- Quit background apps (AI assistants, Electron apps, anything busy)
- **Plug in.** On battery, Apple Silicon is power-limited and can't sustain load
- Quit other CrossOver bottles' Steam clients — they linger and burn CPU

This made a bigger difference than any single graphics setting.

### INI tuning

`SSEDisplayTweaks.ini`:
```ini
FramerateLimit = 120          ; match your display
FramerateLimitMode = 0        ; favor consistent frametimes
MaxFrameLatency = 2           ; absorbs frametime spikes
EnableVSync = true            ; eliminates tearing
```

`SkyrimPrefs.ini` (in `profiles/<profile name>/`):
```ini
uLargeRefLODGridSize = 5      ; from 11 — biggest anti-stutter win for exteriors
fShadowDistance = 4096.0000   ; from 8145 — halves shadow-map cost
fAutosaveEveryXMins = 0.0000  ; timed autosaves cause periodic hitches
```

Also worth doing, in-game: **MCM → Faster HDT-SMP → Performance preset**. Cloth/hair physics is a real CPU sink in crowds.

### Expect first-launch compilation

The first launch after enabling DXMT will sit on a black screen for **10–20 minutes** at high CPU while it compiles shaders. **This is not a hang.** Check with:

```bash
ps -o pid,%cpu,etime -p $(pgrep -f "SkyrimSE.exe" | head -1)
```

High/climbing CPU = working. Near-zero CPU = actually hung. **Let it finish** — killing it throws away all the work and the next launch starts over.

---

## Part 9: Audio fix

If you get crackling/tearing during rain or music, that's an audio buffer underrun — the audio thread misses its deadline during CPU spikes.

Add to `Skyrim.ini` under `[General]`:
```ini
bMultiThreadAudio=1
```

---

## Part 10: A one-click launcher

Rather than manually starting Steam then MO2 each time, save this as an app with **Script Editor** (File → Export → File Format: Application):

```applescript
on run
	set shellCmd to "
CLEAN=\"$HOME/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/bin\"
BOT=\"Skyrim-Modding\"
REAL=\"$HOME/Library/Application Support/CrossOver/Bottles/$BOT\"

LOCK=\"/tmp/gts-launcher.lock\"
if ! mkdir \"$LOCK\" 2>/dev/null; then exit 0; fi
trap 'rmdir \"$LOCK\" 2>/dev/null' EXIT

if pgrep -f 'SkyrimSE.exe' >/dev/null 2>&1; then exit 0; fi
RUNNING_MO2=$(pgrep -f 'ModOrganizer.exe' | while read -r p; do ps -o command= -p \"$p\" 2>/dev/null | grep -vi winewrapper; done)
if [ -n \"$RUNNING_MO2\" ]; then exit 0; fi

WINEPREFIX=\"$REAL\" \"$CLEAN/wineserver\" -k >/dev/null 2>&1
sleep 2

if ! pgrep -f 'steam.exe' >/dev/null 2>&1; then
  nohup \"$CLEAN/cxstart\" --bottle \"$BOT\" \"$REAL/drive_c/Program Files (x86)/Steam/Steam.exe\" >/dev/null 2>&1 &
  disown
  sleep 4
fi

nohup \"$CLEAN/cxstart\" --bottle \"$BOT\" \"$REAL/drive_c/Modlists/GTSAV/ModOrganizer.exe\" >/dev/null 2>&1 &
disown
sleep 8
"
	do shell script shellCmd
end run
```

The lockfile and process checks prevent launching duplicate MO2 instances, which will corrupt your virtual filesystem and freeze the game.

---

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `3417 shaders failed to compile` | Wine's fake `d3dcompiler_47.dll`. See [Part 5](#part-5-the-critical-shader-compiler-fix). |
| Black skin, black sky, broken lighting | Community Shaders is enabled. Disable it — [Part 6](#part-6-disable-community-shaders). |
| Black screen 10+ min on launch | Normal first-run shader compile on DXMT. Check CPU before killing it. |
| Black screen on going outside | You're on GPTK4/D3DMetal 4 beta. Use D3DMetal 3.x or DXMT. |
| `d3d11CreateDeviceAndSwapChain failed` | DXVK selected without proper MoltenVK integration. Switch backend to DXMT. |
| Game zoomed in / wrong scale | Exclusive fullscreen changing display mode. Use borderless + upscale — [Part 7](#part-7-graphics-backend-and-display-config). |
| Mouse trapped, can't escape | Fullscreen input capture during a hang. From Terminal: `pkill -9 -f SkyrimSE.exe` |
| MO2 window won't appear | Stale Wine session. `wineserver -k` on the prefix, then relaunch. |
| Two MO2 windows | Never run two instances — it breaks the VFS. Kill both and start one. |
| Crash referencing `hdtsmp64.dll` | HDT-SMP cloth physics. Use the Faster HDT-SMP **Performance** preset. |
| Stutter every few seconds | Background apps stealing CPU cores. See [Part 8](#part-8-performance-tuning). |

---

## Dead ends (save yourself the time)

Things I tried that **do not work**. Documented so you don't repeat them.

### ❌ Game Porting Toolkit 4 / D3DMetal 4.0 beta

GPTK4 promises big performance gains and is genuinely exciting — but **D3DMetal 4.0b1 black-screens on Skyrim exteriors.** Interiors render fine; step outside and the world goes black.

I confirmed this the thorough way: first with a manual framework graft (which failed *and* corrupted the CrossOver install — see below), then again with a **correct CXPatcher integration** including MoltenVK, DXVK, and all shims. Same result both times. It's a beta bug in the framework, not an installation problem. Revisit when Apple ships a fixed build.

### ❌ Manually grafting GPTK frameworks into CrossOver

Copying `D3DMetal.framework` and GPTK's wine DLLs into `CrossOver.app/Contents/SharedSupport/CrossOver/lib64/apple_gptk/` **breaks your entire CrossOver installation globally** — including bottles and backends you didn't touch. It caused a full GPU lockup requiring a hard reboot, and made previously-stable DXMT start crashing.

If you want a newer GPTK, use [**CXPatcher**](https://github.com/italomandara/CXPatcher), which integrates it properly with the required shims. And **back up your CrossOver.app first** — I couldn't, and had to restore from a stale copy.

### ❌ DXVK

Renders exteriors correctly (unlike GPTK4), but routes through MoltenVK (Vulkan→Metal), adding a translation hop. **Measurably slower** than DXMT/D3DMetal in practice. Worth knowing it *works*, but it's not the performance answer.

### ❌ Community Shaders

Covered above. Not fixable today at any settings level.

---

## Credits & references

- [Gate to Sovngarde](https://github.com/FlimsyParking/Gate-to-Sovngarde-Wabbajack) — the modlist
- [Wabbajack](https://www.wabbajack.org/)
- [CXPatcher](https://github.com/italomandara/CXPatcher)
- [CodeWeavers CrossOver forums](https://www.codeweavers.com/support/forums/) — the Skyrim + Community Shaders thread was invaluable
- [oliveryasuna/Wabbajack-macOS](https://github.com/oliveryasuna/Wabbajack-macOS) — the VM + CrossOver hybrid approach

---

## A final note

If your goal is **maximum smoothness** rather than specifically Gate to Sovngarde, consider a **Steam Deck-tuned modlist** instead (Tuxborn, for example). The Deck is the closest analog to a Mac's translated environment, and lists built for it run dramatically better through CrossOver — on this exact same setup.

GtS is one of the heaviest modlists in existence. Getting it running on a Mac at all is a real achievement. Getting it *buttery* is asking a translated single-threaded engine to do something no configuration can deliver.

It's still a great way to play Skyrim on a Mac. Enjoy it.
