# Tech Splat Generator

**One photo in, a 3D Gaussian splat out.** A local web tool that lifts a single
photograph into a point cloud, flies a camera around it, has a video model turn
that fly-around into a photoreal clip, and builds a training dataset carrying
the exact camera poses the orbit was authored with.

> **Beta, for developers and tinkerers.** Tested on Windows with an NVIDIA GPU.
> Video generation runs on [fal.ai](https://fal.ai) and is **paid per clip** on
> your own account.

## How it works

The camera orbit is decided **before** anything is generated, so the control
render, the AI video and the trained splat all turn in lockstep: frame *i* is
the same camera in every one of them. That is the whole point of the design —
when the path is authored rather than solved after the fact, pose error is
removed by construction instead of being estimated.

The app walks you through five steps in a left-hand rail, with a 3D viewport on
the right that previews everything live.

| Step | What happens |
|---|---|
| **1 Source** | Drop a photo. MoGe-2 measures the lens automatically. |
| **2 Cloud** | The photo is lifted into a point cloud with the depth model you pick: MoGe-2 (metric, default), Depth Anything V2 Small / Large, or Depth Anything 3 Metric / Mono. Clean it up or isolate the subject; it re-lifts as you change settings. |
| **3 Shot** | Author the camera orbit (length, sweep, radius, aim height, lens). The server renders a *control video* of the point cloud from exactly those cameras. |
| **4 Generate** | The control video and your photo go to a fal.ai video model (LTX 2.3 render-to-real, Wan 2.2 VACE depth-control, Wan 3.0 Prime, MiniMax H3). An approval window shows exactly what will be sent before anything is paid for. |
| **5 Review** | Watch the clip, cut a matte of the subject (BiRefNet, SAM 2.1, RMBG-1.4, optional MatAnyone), retime if needed, then **Build Dataset**: the frames, their mattes, the authored poses and an init point cloud, written to `projects\<name>\dataset\`. |

That dataset is where this tool stops. Open the folder in
[Brush](https://github.com/ArthurBrussee/brush) and train there, with whatever
step count and settings you want. Brush writes its `.ply` exports into the
project's `splat_out\` folder, and the viewport's checkpoint switcher loads any
that are there.

Every file the tool reads, writes or uploads is logged with its full path in the
console at the bottom.

## Requirements

- **Windows 10 or 11.** Other platforms are untested (Brush paths and some
  defaults assume Windows).
- **NVIDIA GPU**, 12 GB of VRAM recommended, with a recent driver.
- **Python 3.12**
- **Git** — two packages install straight from GitHub.
- **A fal.ai account and API key** → https://fal.ai/dashboard/keys
- **Brush**, the Gaussian splat trainer → https://github.com/ArthurBrussee/brush/releases
- Several GB of free disk space for Python packages and model downloads.

## Install

Full step-by-step instructions, including the optional extras and how to verify
the install, are in **[INSTALL.md](INSTALL.md)**. The short version, in
PowerShell:

```powershell
git clone https://github.com/Techcopter/splatlab.git
cd splatlab

py -3.12 -m venv .venv
.venv\Scripts\activate

# PyTorch with CUDA first, then everything else
pip install torch==2.8.0 torchvision==0.23.0 --index-url https://download.pytorch.org/whl/cu129
pip install -r requirements.txt

copy .env.example .env      # then put your fal.ai key in it
```

Put `brush_app.exe` at `brush\brush_app.exe`, then:

```powershell
python run.py
```

Open **http://127.0.0.1:8771**. The **Environment** chip in the header turns red
if the fal key or Brush is missing.

The first lift is slow: MoGe-2 downloads and loads once. After that, depth is
cached per photo and re-lifts take well under a second.

The server has no auto-reload. Restart it after changing any `.py` file, and
hard-refresh the browser (Ctrl+F5) after changing anything in `web/`.

## Using it

- Drop a photo in **Source**. The lens is measured and the points are lifted
  without any button presses; **Cloud** opens by itself.
- Anything that runs shows a working card over the viewport with its progress,
  then a tick, a warning or a cross when it ends. Buttons show busy / done /
  failed on themselves.
- Playhead under the viewport: **← →** step a frame (Shift for ten), **Space**
  plays, **Home / End** jump.
- **Generate** re-renders an out-of-date control video for you, then opens the
  approval window. Nothing is uploaded until you press *Approve & send* there.
- **Build Dataset** at the end of Review is the last thing the tool does; it
  stays greyed out until the matte is cut. Training is Brush's job.
- Projects live in `projects\<name>\` (not committed). Use the **⋯** menu to
  rename, copy, export or import a project.

## Configuration

`config.json` is optional. Copy `config.example.json` to create one. Every key
has a default:

| Key | Default | Meaning |
|---|---|---|
| `brush` | `./brush/brush_app.exe` | The Brush executable |
| `projects` | `./projects` | Where projects are stored |
| `python` | the running interpreter | Shown in the Environment panel |
| `host` | `127.0.0.1` | Keep it local: the server has no authentication |
| `port` | `8771` | Change it if the port is taken |
| `log_ring` | `5000` | Console lines kept in memory |

## Models and licences

No model weights are stored in this repository. Each is downloaded from its
original source the first time a feature needs it, and each has **its own
licence**. Read them before any commercial use.

| Model | Used for | Source |
|---|---|---|
| MoGe-2 | Depth, point cloud, lens | [microsoft/MoGe](https://github.com/microsoft/MoGe), `Ruicheng/moge-2-vitl-normal` |
| Depth Anything V2 *(depth option)* | Relative depth | `depth-anything/Depth-Anything-V2-Small-hf` (Apache-2.0), `depth-anything/Depth-Anything-V2-Large-hf`. **Large is non-commercial** (CC-BY-NC-4.0) |
| Depth Anything 3 *(optional depth option)* | Metric or relative depth | [ByteDance-Seed/Depth-Anything-3](https://github.com/ByteDance-Seed/Depth-Anything-3), `depth-anything/DA3METRIC-LARGE`, `depth-anything/DA3MONO-LARGE` (Apache-2.0) |
| BiRefNet / BiRefNet-HR | Subject matte | `ZhengPeng7/BiRefNet`, `ZhengPeng7/BiRefNet_HR` |
| SAM 2.1 + Grounding DINO | Tracked and text-prompted matte | `facebook/sam2.1-hiera-small`, `IDEA-Research/grounding-dino-tiny` |
| RMBG-1.4 | Fast matte | `briaai/RMBG-1.4` |
| MatAnyone *(optional)* | Soft-edged video matte | `PeiqingYang/MatAnyone`. **Non-commercial** (NTU S-Lab 1.0) |
| fal.ai video models | Photoreal clip from the control video | Remote, billed by fal.ai under its terms |
| Brush | Splat training | Separate download |

## Troubleshooting

- **"FAL_KEY is not set"**: check `.env` is next to `run.py`, then restart the server.
- **"Brush not found"**: check the path shown in the Environment panel.
- **CUDA out of memory**: close other GPU apps, or use **⋯ → Free VRAM**.
- **Port already in use**: set another `port` in `config.json`.
- **Something failed**: the console at the bottom has the full error and every
  file path involved. The `error` filter shows only failures.

More cases, with fixes, are in [INSTALL.md](INSTALL.md#troubleshooting-the-install).

## Project layout

```
run.py              starts the server
server/             FastAPI routes, job runner, config
steps.py            geometry, rendering, depth, matting, datasets
falclient.py        fal.ai engines and prompts
web/                the browser app (plain ES modules, no build step)
web/vendor/         three.js, PlayCanvas, GSAP
projects/           your work (created on first run, not committed)
brush/              the Brush trainer you download (not committed)
```

## Credits

Built on [three.js](https://threejs.org) (MIT),
[PlayCanvas](https://github.com/playcanvas/engine) (MIT),
[GSAP](https://gsap.com) (GreenSock standard no-charge licence),
[FastAPI](https://fastapi.tiangolo.com),
[Hugging Face Transformers](https://github.com/huggingface/transformers),
[MoGe](https://github.com/microsoft/MoGe) and
[Brush](https://github.com/ArthurBrussee/brush).

## Licence

The code in this repository is released under the [MIT licence](LICENSE). That
covers this project's own code only, not the models it downloads or the
third-party libraries in `web/vendor/`, which keep their own licences.
