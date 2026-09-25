# idevice-photo-rescue

Extract the **synced photo library** from an old Apple iPod Touch to your PC —
no paid tools, no shady downloads — using the official
[`pymobiledevice3`](https://github.com/doronz88/pymobiledevice3) CLI.

> **Status: pilot.** Tested and working on an **iPod Touch 3G running iOS 5.1.1**,
> from which 2880 photos were recovered. This documents what actually worked.
> The goal is to extend it to other Apple devices over time.

---

## Key finding

On this iPod, the synced photos are stored as **plain JPGs** under
`/PhotoData/Sync/` (in `NNNSYNCD` subfolders). **There is no proprietary format
to decode** — they copy across with a single command. (Older *classic*
click-wheel iPods are different — they use `.ithmb` files — but that's out of
scope for this pilot.)

---

## What worked (the method)

With the iPod connected over USB, **unlocked**, on the home screen, and the
virtual environment active:

```bash
# 1. Does the device connect? (go/no-go check; prints model, iOS, etc.)
python -m pymobiledevice3 lockdown info

# 2. List the media root (/var/mobile/Media)
python -m pymobiledevice3 afc ls /

# 3. Locate the synced photos (JPGs in SYNCD subfolders)
python -m pymobiledevice3 afc ls -r /PhotoData

# 4. Copy EVERYTHING to the PC (-i = keep going if a file fails)
python -m pymobiledevice3 afc pull /PhotoData/Sync ./out/photos -i
```

`pull` **only copies**; it never deletes or modifies anything on the device.
The commands that write to the iPod are separate and explicit (`rm`, `push`).

### Verify the copy

```bash
# How many JPGs were copied?
find ./out/photos -iname '*.jpg' | wc -l

# Any 0-byte corrupt files? (should print nothing)
find ./out/photos -iname '*.jpg' -size 0
```

Open a few JPGs from different folders to confirm they render, and **copy the
result to a second location** (external drive, cloud) before calling the rescue
done. A rescue isn't finished until there are two copies.

---

## Requirements

Tested on **Windows 11**. The `pymobiledevice3` commands are identical on
macOS/Linux; only the venv activation and the drivers differ.

1. **iTunes installed** (Windows) — provides the *Apple Mobile Device USB*
   driver and the `usbmux` service that `pymobiledevice3` needs to talk over USB.
   Without that driver, Windows leaves the iPod in **MTP** mode and only `/DCIM`
   is visible.
2. **Python 3.10+** and a virtual environment:

```bash
python -m venv .venv

# Activate the venv:
source .venv/Scripts/activate     # Windows (Git Bash)
# .\.venv\Scripts\Activate.ps1    # Windows (PowerShell)
# source .venv/bin/activate       # macOS / Linux

pip install -r requirements.txt
```

On PowerShell, if it blocks activation due to the script policy:
`Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`.

To leave the venv: `deactivate`.

---

## If the iPod doesn't show up (Windows: MTP mode)

Check in **Device Manager** that the iPod appears under *Universal Serial Bus
controllers* as **"Apple Mobile Device USB Driver"**. If it instead shows up
under *Portable Devices* as MTP, Windows didn't pick up the Apple driver and you
won't reach `/PhotoData` over AFC (you'll only see `/DCIM`).

What fixed it: Device Manager → right-click the iPod → **Uninstall** (tick
"delete the driver software") → disconnect → **reboot the PC** → reconnect. With
iTunes installed, on reconnect it picks up the Apple driver and moves to
*Universal Serial Bus controllers*.

---

## What we learned

- **Explore the filesystem before assuming anything.** The initial plan assumed
  the photos would be proprietary `.ithmb` files that needed decoding. They
  weren't — they were JPGs. An `afc ls -r /PhotoData` up front would have shown
  that and saved half the design.
- **The CLI avoids the async mess.** `pymobiledevice3` v11 uses an async API
  (`create_using_usbmux` is a coroutine); calling it as a sync function fails
  with `coroutine ... was never awaited`. The CLI wraps all of that — for a
  one-off rescue, it's the simplest, most reliable route.
- **No paid software or jailbreak needed.** Just the right driver and AFC.

---

## Notes

- **Quality:** it's whatever iTunes put on the device when syncing (here,
  1152×768). There's no higher-resolution original inside the iPod; for that
  you'd need the computer it was originally synced from.
- **Dates:** `IMG_XXXX.JPG` filenames don't guarantee chronological order, and
  EXIF may be missing. Sorting by real date, if wanted, is a separate step.
- **Privacy:** `out/` is in `.gitignore`. Don't push recovered photos to a
  public repository.

---

## Repo structure

```
idevice-photo-rescue/
├── README.md          # this file: the real story + the commands
├── requirements.txt   # pymobiledevice3
├── .gitignore         # ignores out/, .venv/, etc.
└── .vscode/           # recommended extensions
```