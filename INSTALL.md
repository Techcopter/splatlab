# Installing Tech Splat Generator

Start to finish this takes about 20–30 minutes, most of it waiting on downloads.
Every command below is PowerShell, run from the folder you want the tool to live
in.

There is no installer and nothing is written outside the project folder. To
uninstall, delete the folder.

---

## 0. Before you start

Check each of these. Skipping one is the cause of most failed installs.

| Need | Check it with | Expected |
|---|---|---|
| Windows 10 or 11 | `winver` | 10 or 11 |
| Python 3.12 | `py -3.12 --version` | `Python 3.12.x` |
| Git | `git --version` | any version |
| NVIDIA GPU + driver | `nvidia-smi` | your card, and a CUDA version |
| Free disk space | — | ~15 GB (packages + model weights) |

**Python 3.12 specifically.** The pinned package versions are built against it.
Get it from [python.org](https://www.python.org/downloads/release/python-3120/)
and tick *Add python.exe to PATH* during setup. If `py -3.12 --version` fails
but `python --version` reports 3.12, use `python` in place of `py -3.12` below.

**Git is not optional.** Two packages (MoGe-2 and utils3d-moge) install directly
from GitHub, so `pip` shells out to `git`. Without it on PATH, step 3 fails.

**No NVIDIA GPU?** The tool will not work usefully. Depth, matting and rendering
all assume CUDA.

---

## 1. Get the code

```powershell
git clone https://github.com/Techcopter/splatlab.git
cd splatlab
```

Everything from here runs inside this folder.

## 2. Create and activate a virtual environment

```powershell
py -3.12 -m venv .venv
.venv\Scripts\activate
```

Your prompt should now be prefixed with `(.venv)`. **It must stay that way for
every command below,** and in any new terminal you open later — re-run
`.venv\Scripts\activate` each time.

> If activation is blocked with *"running scripts is disabled on this system"*,
> allow it for your user once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

## 3. Install PyTorch first

PyTorch must go in **before** `requirements.txt`, and it must be the CUDA build
that matches your driver. This is the cu129 build:

```powershell
pip install torch==2.8.0 torchvision==0.23.0 --index-url https://download.pytorch.org/whl/cu129
```

This is the big download (~2.5 GB). If your driver is older, pick the matching
build from [pytorch.org](https://pytorch.org/get-started/locally/) instead — but
keep the `torch==2.8.0 torchvision==0.23.0` versions.

Confirm CUDA is visible before moving on:

```powershell
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
```

You want `2.8.0+cu129 True`. **If it prints `False`, stop and fix it here** —
every later step depends on it. See
[CUDA reports False](#torchcudais_available-returns-false) below.

## 4. Install everything else

```powershell
pip install -r requirements.txt
```

FastAPI, OpenCV, transformers, MoGe-2 and the rest. The two `git+https://`
packages make this slower than a normal install.

## 5. Optional extras

Skip this section entirely if you like — every feature below simply reports
that it is not installed.

```powershell
pip install -r requirements-optional.txt
```

That covers **MatAnyone** (soft-edged video matte; non-commercial licence) and
**Rerun** (live training telemetry).

**Depth Anything 3** needs two more installs, and both must use `--no-deps`.
Its own dependency list pins `numpy<2` and pulls in open3d, moviepy 1.x and a
pinned gsplat, which would break this environment. Depth-only inference needs
none of them:

```powershell
pip install --no-deps evo==1.37.1
pip install --no-deps "depth-anything-3 @ git+https://github.com/ByteDance-Seed/Depth-Anything-3@3d835ec1a5802d64a8b8b15f817a1ab54809bfe4"
```

## 6. Add your fal.ai API key

Video generation runs on [fal.ai](https://fal.ai) and is **billed per clip on
your own account**. Get a key at https://fal.ai/dashboard/keys.

```powershell
copy .env.example .env
notepad .env
```

Set the one line and save:

```
FAL_KEY=your-key-here
```

`.env` is listed in `.gitignore` and stays on your machine. A real `FAL_KEY`
environment variable also works and takes precedence over the file, so a stale
`.env` can never silently override an exported key.

> The approval window in step 4 of the pipeline shows exactly what will be sent
> before anything is uploaded or charged. Nothing reaches fal.ai until you press
> *Approve & send*.

## 7. Install Brush (the splat trainer)

Brush is a separate project and a separate ~160 MB download. It is never
bundled here.

1. Get the Windows build from the
   [Brush releases page](https://github.com/ArthurBrussee/brush/releases).
2. Put the executable at **`brush\brush_app.exe`** inside this folder, renaming
   it if the release uses a different file name.

Keeping Brush somewhere else? Make a config file and point at it:

```powershell
copy config.example.json config.json
notepad config.json
```

```json
{ "brush": "D:/tools/brush/brush_app.exe" }
```

`config.json` is local and gitignored. Every key is optional — see
[Configuration](README.md#configuration) for the full list.

## 8. Run it

```powershell
.venv\Scripts\activate
python run.py
```

Open **http://127.0.0.1:8771**.

Check the **Environment** chip in the header: green means the fal key and Brush
were both found, red means one is missing — click it for the detail.

## 9. Verify the install

Drop any photo into **Source**. A successful first run looks like this:

1. The lens is measured and **Cloud** opens on its own, with no button presses.
2. A point cloud appears in the viewport.
3. The console at the bottom logs each file it touched, with full paths.

**The first lift is slow** — a minute or more, because MoGe-2 downloads (~1.5 GB)
and loads once. After that, depth is cached per photo and re-lifts take well
under a second. A first lift that looks frozen is almost always just the model
download; watch the console.

That is the install verified. The remaining steps cost fal.ai credits, so stop
here if you only wanted to confirm the setup works.

---

## Troubleshooting the install

### `py -3.12` is not recognised
Python 3.12 is not installed, or not on PATH. Reinstall from python.org with
*Add python.exe to PATH* ticked. If `python --version` already reports 3.12, use
`python` instead of `py -3.12`.

### `torch.cuda.is_available()` returns False
In order of likelihood:
- **Wrong wheel.** A CPU-only build was installed. Check with
  `python -c "import torch; print(torch.__version__)"` — you need a `+cu129`
  suffix, not a bare `2.8.0`. Reinstall with the `--index-url` from step 3.
- **Driver too old for cu129.** Run `nvidia-smi` and read the CUDA version in
  the top-right. If it is below 12.9, either update your NVIDIA driver or
  install the matching older build from
  [pytorch.org](https://pytorch.org/get-started/locally/).
- **No NVIDIA GPU present.** Nothing to fix; the tool needs one.

### `git` errors, or "could not find a version that satisfies moge"
Git is missing from PATH. Install [Git for Windows](https://git-scm.com/download/win),
**open a new terminal**, re-activate the venv, and re-run step 4.

### pip fails building a wheel
Some packages need the Microsoft C++ build tools. Install the
*Desktop development with C++* workload from
[Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/),
then re-run step 4.

### Install succeeded but imports fail at runtime
Usually the venv is not active — no `(.venv)` in your prompt. Run
`.venv\Scripts\activate` and try again. Confirm you are on the right
interpreter with `python -c "import sys; print(sys.executable)"`; it should be
the `.venv\Scripts\python.exe` inside this folder.

### numpy or opencv errors after installing Depth Anything 3
DA3 was installed without `--no-deps` and has downgraded numpy. Restore the
pinned versions:

```powershell
pip install -r requirements.txt --force-reinstall --no-deps
```

Then reinstall DA3 with `--no-deps` as step 5 shows.

### "FAL_KEY is not set"
`.env` must sit next to `run.py`, contain `FAL_KEY=...`, and the server must be
restarted after it changes. Check for a stray `.env.txt` — Notepad adds the
extension if the save dialog's type is left as *Text Documents*.

### "Brush not found"
The Environment panel prints the exact path being checked. Either move the
executable there, or set `"brush"` in `config.json` to where it actually is.

### Port 8771 already in use
Set another port in `config.json`:

```json
{ "port": 8790 }
```

### CUDA out of memory
Close other GPU applications, or use **⋯ → Free VRAM** in the header. Models
are held on the CPU between calls and moved to the GPU only while running, so
this is usually another process rather than this one.

---

## Updating

```powershell
git pull
.venv\Scripts\activate
pip install -r requirements.txt
```

Your `.env`, `config.json` and `projects\` folder are untouched by an update —
all three are gitignored.

## Notes on running it

- **There is no auto-reload.** Restart the server after editing any `.py` file.
  After editing anything in `web/`, hard-refresh the browser with Ctrl+F5.
- **The server has no authentication.** Leave `host` as `127.0.0.1`. Binding it
  to `0.0.0.0` exposes full read and write access to your projects folder to
  anyone who can reach the machine.
- **Model weights are downloaded on first use**, not bundled, and cached in your
  Hugging Face cache (`%USERPROFILE%\.cache\huggingface` by default). Each model
  carries its own licence — see
  [Models and licences](README.md#models-and-licences) before any commercial
  use.
