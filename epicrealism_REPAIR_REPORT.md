# EpicRealism notebook: repair report

This report covers `epicrealism.ipynb`. The untouched original is saved as `epicrealism_original_backup.ipynb`.

**Short version:** the MicrosoftExcel WebUI is kept, along with the original cells, `!curl` lines and extensions. It broke because Colab moved to Python 3.13, the launch line installed unpinned package versions, and one GitHub repository that `launch.py` clones has disappeared. The fix gives the WebUI its own Python 3.10 environment with the versions it was released with. **Nothing has been run on a GPU yet.** Section 8 lists what was actually tested and what you need to check in Colab.

---

## 1. What changed

The notebook keeps its original layout: the main setup and launch cell, the LoRA headings, the LoRA and relaunch cell, and the footer. Every changed line is marked `# [FIX]` or `# [SECURITY]`.

| # | Change | Why |
|---|---|---|
| 1 | **Added a Python 3.10 environment** in `/content/sdenv`, set up with `uv`. The WebUI runs as `/content/sdenv/bin/python launch.py` | Colab's runtime 2026.10 uses Python 3.13 on Ubuntu 24.04 ([Colab runtime versions](https://research.google.com/colaboratory/runtime-version-faq.html)). WebUI 1.5.1 needs torch 2.0.1, numpy 1.23.5 and gradio 3.32. None of these install on Python 3.13, and gradio 3.32 also imports `audioop`, which Python 3.13 removed. |
| 2 | **`REQS_FILE="requirements_versions.txt"`** (the original used `requirements.txt`) | `requirements.txt` in WebUI 1.5.1 has almost no version pins, so every run installed the latest numpy 2, Pillow, pydantic 2 and so on, and the 2023 code broke. `requirements_versions.txt` is the pinned list shipped with the WebUI. |
| 3 | **A constraints file** (`/content/constraints.txt`, applied through `PIP_CONSTRAINT`) | It keeps `launch.py` and the extension installers from upgrading packages past what this WebUI supports. Each pin is explained in section 3. |
| 4 | **`STABLE_DIFFUSION_REPO=https://github.com/w-e-w/stablediffusion.git`** | `github.com/Stability-AI/stablediffusion` is gone: `git ls-remote` now asks for a login. `launch.py` clones it at commit `cf1d67a6…` on first start. The w-e-w mirror has that exact commit, and current AUTOMATIC1111 switched to the same mirror (commit `fd68e0c3`, Dec 2025). |
| 5 | **CLIP is pre-installed with `--no-build-isolation`** | CLIP's `setup.py` needs `pkg_resources`, which recent setuptools no longer ships, so `launch.py`'s own CLIP install fails. AUTOMATIC1111 fixed this the same way in Feb 2026 (commit `76759a18`). |
| 6 | **`pip==24.0`, `setuptools==69.5.1`** inside the environment | pip 24.1 and later reject `torchsde==0.2.5` (pinned by the WebUI) because of its old-style version specifier. I confirmed pip 24.0 prints that warning. |
| 7 | **torch 2.0.1 + CUDA 11.8 and xformers 0.0.20** installed up front | These are WebUI 1.5.1's own defaults (`launch_utils.py`). Installing them up front means `launch.py` doesn't try to download them itself. |
| 8 | Removed `--disable-safe-unpickle` and `--enable-insecure-extension` from both launch lines | See section 6. |
| 9 | `unzip -o` and `mkdir -p` | Rerunning a cell no longer stops at an "overwrite?" prompt or a "folder exists" error. |
| 10 | ADetailer (commented line) pinned with `-b v23.8.1` | ADetailer 24.6 and later require WebUI 1.6 or newer (its CHANGELOG, 2024-06-16). v23.8.1 is the last release before it started using 1.6 features. |
| 11 | Optional Google Drive block (commented out) | Section 5. |

**Kept unchanged:** the memfix / tcmalloc lines, the MicrosoftExcel zip, every extension `git clone` line, the images browser, embeddings and upscalers bundles, the EpicRealism `curl` line with its custom file name, all 22 ControlNet/T2I-Adapter lines with their custom file names, the LoRA cell and examples, `--share`, `--no-half-vae`, `--xformers`, `--gradio-queue` (which does nothing in 1.5.1 but is harmless), and the cloudflared alternative.

**Custom local file names were not touched.** In ControlNet they actually matter: each local name matches a `.yaml` config that ships inside `microsoftexcel-controlnet/models/`. For example, `control_v11p_sd15_depth_fp16` from Hugging Face is saved as `control_v11f1p_sd15_depth.safetensors` so it pairs with `control_v11f1p_sd15_depth.yaml`. Both depth models use the same SD 1.5 ControlNet architecture, so this is not a mismatch.

## 2. Is the original WebUI still restorable?

Yes, as far as I could check without a GPU. Nothing found so far requires moving to another WebUI.

* **MicrosoftExcel 1.5.1** is AUTOMATIC1111 v1.5.1 under another name. The extensions are nolanaatama's renamed forks of the standard A1111 extensions. The file name `microsoftexcel151`, the 1.5.1 era of the extension forks (July–August 2023), and the commit message in the original repository ("1.5.1 web ui + new memfix") all point to this.
* **I could not open the zip itself.** My sandbox blocks huggingface.co, so `launch.py`, `requirements_versions.txt` and the zip contents were not inspected directly. Everything about its internals comes from the AUTOMATIC1111 v1.5.1 tag. If nolanaatama changed anything beyond the name, the first Colab run will show it.
* No fallback is proposed. If Phase 1 in section 8 fails, send me the error and I'll work out whether the problem is in the zip or the environment before anything gets replaced.

## 3. Selected dependency versions

The core versions are WebUI 1.5.1's own `requirements_versions.txt` and `launch_utils.py` defaults. The rest are pins I added because later releases break this WebUI. Several of them were confirmed by testing (see section 8).

| Package | Version | Source / reason |
|---|---|---|
| Python | 3.10 (standalone build via `uv`) | WebUI 1.5.1 targets 3.10.6 |
| torch / torchvision | 2.0.1+cu118 / 0.15.2+cu118 | WebUI 1.5.1 default |
| xformers | 0.0.20 (`--no-deps`) | WebUI 1.5.1 default |
| numpy | 1.23.5 | requirements_versions.txt |
| Pillow | 9.5.0 | requirements_versions.txt |
| gradio | 3.32.0 | requirements_versions.txt |
| **gradio_client** | **0.2.5** | **Confirmed break:** gradio 3.32 accepts any `gradio_client>=0.2.4`, so pip installed 2.7.1, and `import gradio` then fails (`cannot import name 'media_data'`). |
| fastapi | 0.94.0 | requirements_versions.txt |
| pydantic | <2 (gets 1.10.x) | fastapi 0.94 and gradio 3.32 need pydantic 1 |
| httpx | 0.24.1 | gradio 3.x breaks on httpx 0.28 |
| **websockets** | **11.0.3** | **Confirmed break:** websockets 16 changed its API, and gradio's queue client fails (`unexpected keyword argument 'extra_headers'`). |
| uvicorn | 0.23.2 | Release from the same period as websockets 11 |
| transformers | 4.25.1 | requirements_versions.txt |
| pytorch_lightning | 1.9.4 | requirements_versions.txt |
| open-clip-torch | 2.20.0 | requirements_versions.txt. Not in the constraints file, because `launch.py` installs 2.7.0 from a pinned commit first and then upgrades it. |
| safetensors | 0.3.1 | requirements_versions.txt |
| huggingface-hub | <0.26 | diffusers 0.20 (SuperMerger) and transformers 4.25 use `cached_download`, which was removed in 0.26 |
| opencv-python (and -headless, -contrib) | <4.12 (gets 4.11) | opencv 4.12 requires numpy 2 |
| protobuf | <4 (3.20.3) | needed by mediapipe 0.10.9 and ADetailer v23.8.1 |
| **mediapipe** | **0.10.9** | **Confirmed break:** without a pin, ControlNet's installer pulls mediapipe 1.2.2 (Nov 2026), which no longer has `mediapipe.solutions`. ControlNet's mediapipe-face preprocessor and ADetailer use it. |
| diffusers | <0.21 | SuperMerger (2023) |
| ultralytics | <8.1 | ADetailer v23.8.1 |
| pip / setuptools | 24.0 / 69.5.1 | Section 1, row 6 |

**tcmalloc / memfix:** `memfix.zip` contains `libtcmalloc_minimal.so.4` from gperftools 2.5, built against glibc 2.34. Colab's Ubuntu 24.04 has glibc 2.39, so it should load, and it is kept as in the original. If you ever see `ERROR: ld.so: object ... cannot be preloaded`, delete the `%env LD_PRELOAD` line. The WebUI runs without it and only uses a bit more RAM when you switch checkpoints.

**GPU:** torch 2.0.1 with CUDA 11.8 supports T4, L4, A100 and H100. It does **not** support Blackwell GPUs (compute capability 10 or higher). If Colab gives you one, choose T4 or L4 instead.

## 4. Extensions

These statuses are based on reading each repository and checking that its dependencies install. None of them could be loaded in a running WebUI here.

| Extension | Status | Notes |
|---|---|---|
| microsoftexcel-tunnels | Unverified, likely works | Adds `--cloudflared`. Installs `pycloudflared` (0.2.0 installs fine). |
| microsoftexcel-controlnet | Unverified, likely works | ControlNet v1.1.238 (Aug 2023), made for 1.5.x. Its dependencies (mediapipe, svglib, fvcore, opencv>=4.8) install cleanly with the pins above. Repository is reachable. |
| openpose-editor (fkunn1326) | Unverified | The latest commit (Oct 2023) includes a "fix for new gradio version" for WebUI 1.6. If its image upload misbehaves on gradio 3.32, run `git -C /content/microsoftexcel/extensions/openpose-editor checkout 722bca6` (June 2023). |
| microsoftexcel-3d-open-pose-editor | Unverified | On first start its installer downloads mediapipe pose files from cdn.jsdelivr.net (blocked in my sandbox). |
| a1111-microsoftexcel-tagcomplete | Unverified, likely works | No Python dependencies. |
| a1111-microsoftexcel-locon | **Obsolete but harmless** | The author marked it deprecated. WebUI 1.5.0 added LoCon/LyCORIS support to the built-in Lora, so those files already work without it. In 1.5.1 its patches target the old `lora` module, which the new code no longer calls, so it does nothing. Kept as in the original. You can comment its line out. |
| Images browser (zip) | Unverified | Downloaded from Hugging Face, which my sandbox blocks. |
| ADetailer | Optional, pinned to v23.8.1 | Its dependencies install with the pins above (ultralytics 8.0.239, mediapipe 0.10.9). The original line that clones its models from Hugging Face needs git-lfs. ADetailer also downloads the models itself on first use. |
| Deforum | Optional, unverified | Dependencies resolve. |
| Mov2mov | Optional, unverified | Its installer runs `pip install ffmpeg`, which is a useless package. Harmless. |
| SuperMerger | Optional, unverified | Resolves to diffusers 0.20.x under the pins. |
| TemporalKit | Optional, unverified | Dependencies resolve (moviepy 1.x under numpy 1.23). |
| Ultimate SD Upscale | Optional, unverified | No dependencies. Its 2024 change uses a WebUI 1.7 hook inside a try/except, so it's harmless on 1.5.1. |
| roop | **Not recommended** | Colab's FAQ forbids "creating deepfakes" on every runtime. Its pins (`opencv-python==4.7.0.72`, `onnxruntime==1.15.0`) also conflict with ControlNet's `opencv-python>=4.8`. Left commented out. |

Each optional extension is still an ON/OFF switch: remove or add the `#` in front of its line, as before. If an optional extension fails to install, `launch.py` prints the error and keeps going.

## 5. How to add models, and Google Drive

Adding models works exactly as before. Copy a `!curl` line, change the link and the local file name:

```
!curl -Lo /content/microsoftexcel/models/Stable-diffusion/<your-name>.safetensors <link>     # checkpoints (SD 1.5, SD 2.1, SDXL)
!curl -Lo /content/microsoftexcel/models/Lora/<your-name>.safetensors <link>                 # LoRA / LoCon / LyCORIS
!curl -Lo /content/microsoftexcel/embeddings/<your-name>.safetensors <link>                  # textual inversion
!curl -Lo /content/microsoftexcel/models/VAE/<your-name>.safetensors <link>                  # VAE
!curl -Lo /content/microsoftexcel/models/ESRGAN/<your-name>.pth <link>                       # upscalers
!curl -Lo /content/microsoftexcel/extensions/microsoftexcel-controlnet/models/<name>.safetensors <link>  # ControlNet
```

* On Hugging Face, use `/resolve/main/` links, not `/blob/main/`. If a download is only a few KB, it's an error page: the link is wrong, or the model needs you to log in.
* Every checkpoint in `models/Stable-diffusion` shows up in the checkpoint dropdown, and you switch between them in the WebUI without rerunning anything.
* **Don't mix model families.** LoRAs, embeddings and ControlNet models made for SD 1.5 only work with SD 1.5 checkpoints, and the same goes for SD 2.1. The 22 ControlNet lines in the notebook are all SD 1.5. VAEs are shared between SD 1.5 and 2.1.
* **SD 2.1:** Stability AI's `stabilityai/stable-diffusion-2-1` repository has returned 404 on Hugging Face since about November 2025 ([forum thread](https://discuss.huggingface.co/t/error-404-https-huggingface-co-stabilityai-stable-diffusion-2/170425)), so the repo's `sd2.1.ipynb` link is dead too. Its `.yaml` link also pointed at the deleted GitHub repository. The working config URL is `https://raw.githubusercontent.com/w-e-w/stablediffusion/main/configs/stable-diffusion/v2-inference-v.yaml`. For the weights, use a mirror you trust.
* If you are allowed to view a `.safetensors` file, its header shows the family: a cross-attention width of 768 means SD 1.x, 1024 means SD 2.x, and 2048 means SDXL.

**Google Drive persistence (optional):** in the first cell, remove the `#` from the four lines under `(OPTIONAL) Google Drive`. On each run they mount Drive and replace the WebUI's `Stable-diffusion`, `Lora`, `VAE`, `ESRGAN` and `embeddings` folders with links to `MyDrive/microsoftexcel/...`. Anything already in those folders is copied into Drive first. After a model has been downloaded once, put `#` in front of its `!curl` line, because `curl -Lo` downloads it again otherwise. You can also drop model files straight into those Drive folders and press 🔄 in the WebUI. ControlNet models stay in the runtime, because that folder also holds ControlNet's `.yaml` files.

## 6. Security

* **`--disable-safe-unpickle` (removed):** `.ckpt`, `.pt` and `.pth` files are Python pickles and can contain code that runs when they're loaded. The WebUI's safe unpickler blocks that. Turning it off was only ever needed for unusual `.pt` files. EpicRealism and all the ControlNet models here are `.safetensors`, which can't contain code. Add the flag back only for a specific file you trust that gets refused as "may be malicious", and prefer a `.safetensors` version when one exists. The Deliberate example link is a `.ckpt` and should load with the check on.
* **`--enable-insecure-extension` (removed):** with `--share`, this flag lets **anyone with your link** install extensions, which means running any code on your Colab machine. Extensions are installed from the notebook, so the WebUI doesn't need it. The Extensions tab still lists them. It just can't install new ones.
* **Public link:** anyone who has the `gradio.live` or `trycloudflare.com` link can use your WebUI and your GPU while it's running. Add `--gradio-auth user:password` to `COMMANDLINE_ARGS` to require a login.
* **Tokens:** none are needed for the default downloads. If a model needs one, store it as a Colab Secret (🔑 in the left bar), never in the notebook.
* All the extensions come from nolanaatama's forks or well-known upstream repositories, and every one of them was reachable. They're still third-party code that runs with full access to the runtime.

## 7. Colab usage rules

From the [Colab FAQ](https://research.google.com/colaboratory/faq.html): free runtimes, meaning no compute units, may not be used for "bypassing the notebook UI to interact primarily via a web UI", and Google may end them without warning. **A WebUI like this one needs a paid plan or a positive compute-unit balance.** The "microsoftexcel" name doesn't change that, and this notebook doesn't try to get around it. All runtimes prohibit creating deepfakes, which is why roop stays off.

## 8. Testing report

**What I actually ran** (in a cloud sandbox without a GPU or Colab, and with huggingface.co and download.pytorch.org blocked):

| Test | Result |
|---|---|
| Original notebook matches the repository copy | ✅ identical |
| All 13 extension repositories and the 5 repositories `launch.py` clones respond on GitHub | ✅ all except `Stability-AI/stablediffusion`, which is ❌ gone; the w-e-w mirror has the pinned commit |
| `memfix.zip` (GitHub) downloaded and inspected | ✅ gperftools 2.5, needs glibc 2.34 or newer |
| Python 3.10 created with `uv python install 3.10` | ✅ |
| torch 2.0.1, xformers 0.0.20, GFPGAN, CLIP (`--no-build-isolation`), then WebUI 1.5.1's `requirements_versions.txt`, all under the constraints | ✅ installs (I used PyPI's cu117 torch build here because the cu118 index is blocked in my sandbox) |
| Importing the whole dependency stack (torch, xformers, gradio, fastapi, pydantic, transformers, open_clip, basicsr, gfpgan, realesrgan, cv2 and more) | ✅ after adding the gradio_client pin; ❌ without it |
| A small gradio 3.32 app with the queue on, page load plus a call over websockets | ✅ after adding the websockets/uvicorn pins; ❌ without them |
| ControlNet, Tunnels and ADetailer dependencies installed; ControlNet, ADetailer, Deforum, SuperMerger, TemporalKit and roop dependencies resolved | ✅ after adding the mediapipe pin, which keeps `mediapipe.solutions` |
| Notebook cells parse as valid IPython, including the Drive block once uncommented | ✅ |
| Drive folder-linking commands, run twice against test folders | ✅ files preserved, links correct |

**Not tested (needs your Colab GPU session):** downloading the MicrosoftExcel zip, starting `launch.py`, the browser UI, generating images, extensions loading, the gradio share link, and Google Drive. I wasn't permitted to run the WebUI's own `launch.py` and extension code in my sandbox, and it has no GPU. None of these should be considered working until you've seen them work.

**Steps for you to run in Colab, in order:**

* **Phase 1 (base WebUI):** Run the first cell on a T4. Expect `Python 3.10.x` near the top of the launch output (the `Version:` line may say unknown if the zip has no git history), then `Running on public URL`. Open the link. EpicRealism should be in the checkpoint dropdown. Generate a 512×512 image with your usual settings. It should appear in the gallery and in `/content/microsoftexcel/outputs/txt2img-images`. Uncomment the Deliberate line (or add another checkpoint), rerun, and switch models in the dropdown.
* **Phase 2 (assets):** Stop the first cell, uncomment Detail Tweaker in the LoRA cell, run it, and use `<lora:detailtweaker:0.8>`. Try one of the bundled embeddings in the negative prompt, a VAE in Settings → SD VAE, and Hires.fix with one of the bundled upscalers.
* **Phase 3 (extensions):** Uncomment one or two ControlNet lines, for example canny and openpose. Run a ControlNet canny generation, then the OpenPose Editor → "Send to txt2img" flow. After that, turn on one optional extension at a time.
* **Phase 4 (persistence):** Uncomment the Drive lines and run. Then do Runtime → Disconnect and delete runtime, run again with the model `curl` lines commented out, and check that the models are still listed.

If any step fails, paste the last 30 lines of the cell output and I'll fix it from there.

## 9. Could not be restored or checked

* The MicrosoftExcel zip, the images browser, the embeddings and upscalers bundles, and every Hugging Face model link could not be checked (huggingface.co is blocked in my sandbox).
* The official SD 2.1 weights are no longer on Hugging Face (section 5).
* roop is left off because of Colab's terms (section 4).
* LoCon does nothing in this WebUI version; the built-in Lora replaces it (section 4).
