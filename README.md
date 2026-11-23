# Direct3D-V3 Clean for ComfyUI

**ComfyUI adapter made by [@MostAadTech](https://github.com/Mstafa-awad).**

Windows custom nodes that refine a coarse 3D mesh with [Direct3D-V3](https://github.com/DreamTechAI/Direct3D-V3) and a reference image. They run high-token 1536 refinement (150k+ patches) on an **8 GB GPU** and export raw marching-cubes geometry, so decimation, smoothing and cleanup stay in your own workflow.

## ⚡ New: SuperFast builds — up to 2.2x faster

The SuperFast builds add custom Triton GPU kernels (by @MostAadTech) and a new **`accel_mode`** option on the Refine nodes:

| `accel_mode` | What it does | 1536 sampling time* | Quality vs original |
| --- | --- | --- | --- |
| `off` | Original path | 925 s | — |
| **`lossless`** (default) | Fused FP16 attention, decoder and RoPE kernels, async transfers | **576 s (1.61x)** | Same as FP16; only rounding-level differences |
| **`fast`** | + INT8 attention scores and SmoothQuant INT8 for the least sensitive layers | **459 s (2.02x)** | Tiny deviation (avg 0.085 voxel) |
| **`fastest`** | + INT8 MLP layers | **425 s (2.18x)** | Small deviation (avg 0.134 voxel) |

\*RTX 5050 Laptop GPU, 8 GB VRAM. Single-stage 1536, 152,121 patches, 8 steps, CFG 4. Total time per mesh in ComfyUI went from **16:09 to 10:04** in `lossless` mode.

The speedups also apply to the 512 and 1024 stages and to the full 512→1024→1536 cascade:

| Run | `off` | `lossless` | `fast` |
| --- | --- | --- | --- |
| 512 single stage | 35.9 s | 23.4 s (1.53x) | 22.7 s (1.58x) |
| 1024 single stage | 242 s | 156 s (1.55x) | 138 s (1.75x) |
| Cascade 512→1024→1536 (total) | 1315 s | 864 s (1.52x) | 706 s (1.86x) |
| 1536 single stage | 925 s | 576 s (1.61x) | 459 s (2.02x) |

![off / lossless / fast / fastest comparison](./Direct3D-V3-modes-comparison.png)

SuperFast builds also include:

* **Lower memory spikes.** The DINOv3 encoder now loads straight into the GPU, cutting peak RAM from 5.3 GB to 0.8 GB with identical output. Marching cubes reserves 6.4 GB of VRAM instead of 10.5 GB at 152k patches, so it no longer spills into shared memory.
* **Stays under 8 GB of VRAM** (about 7.3 GB peak at 152k patches). Weights stream from system RAM. Task Manager may list that RAM as "Shared GPU memory", but it is ordinary system memory that the GPU copies from.
* **Safe fallback.** GPUs older than the RTX 30 series, a missing Triton install, or BF16 dtype run the original path automatically.

## Download

| Build | Worker Python | PyTorch | Acceleration |
| --- | --- | --- | --- |
| ⭐ [**cu128 SuperFast ZIP**](./ComfyUI-Direct3D-V3-Windows-cu128_int8_superfast.zip?raw=1) | 3.12 x64 | 2.8.0+cu128 | ✅ `accel_mode` (tested) |
| [**cu130 SuperFast ZIP**](./ComfyUI-Direct3D-V3-Windows-cu130_int8_superfast.zip?raw=1) | 3.13 x64 | 2.12.1+cu130 | ✅ `accel_mode` (awaiting runtime test) |
| [cu128 ZIP (classic)](./ComfyUI-Direct3D-V3-Windows-cu128.zip?raw=1) | 3.12 x64 | 2.8.0+cu128 | — |
| [cu130 ZIP (classic)](./ComfyUI-Direct3D-V3-Windows-cu130.zip?raw=1) | 3.13 x64 | 2.12.1+cu130 | — |

**Which one?** Pick **cu128 SuperFast** for most setups. It is the build tested end to end. The classic builds are the earlier releases without `accel_mode`.

Every ZIP includes prebuilt voxelizers for NVIDIA GeForce **RTX 20, 30, 40 and 50 series**. You need a current NVIDIA driver and the listed 64-bit Python version. The worker uses its own Python environment, so ComfyUI keeps its existing Python and PyTorch. Setup needs no CUDA Toolkit or Visual Studio.

| GPU | Classic builds | SuperFast `accel_mode` |
| --- | --- | --- |
| RTX 50 series | ✅ | ✅ tuned and tested (RTX 5050 Laptop) |
| RTX 30 / 40 series | ✅ | ✅ supported, not yet benchmarked |
| RTX 20 series | ✅ | falls back to `off` automatically |

**Build status.** The cu128 SuperFast build was validated on an RTX 5050 Laptop GPU with 8 GB VRAM. That covered the 512, 1024 and 1536 single stages, the full cascade, and a second mesh. The cu130 builds share the same code but have not had a runtime test yet. They were compiled with CUDA Toolkit 13.5 against PyTorch cu130, so use a driver with CUDA 13.5 support.

## Install

1. Choose **one** ZIP and extract its `ComfyUI-Direct3D-V3` folder into `ComfyUI/custom_nodes`. All builds register the same node IDs, so replace the old folder when switching builds.
2. Request access to [DINOv3 ViT-H+ on Hugging Face](https://huggingface.co/facebook/dinov3-vith16plus-pretrain-lvd1689m). If setup asks for authentication, use a read token from the account that has access.
3. Run `setup.cmd` from the extracted node folder. It creates the private `.runtime` environment and downloads PyTorch, Triton (SuperFast builds), the worker dependencies, DINOv3 and the 1536 geometry checkpoint. Allow several gigabytes of disk space.
4. Restart ComfyUI.

For cascade mode, download all three geometry stages with `setup.cmd --models-only --stage all` after setup. Add `--birefnet` to cache the optional background remover.

## Use in ComfyUI

1. Add **Direct3D-V3 Clean | Mesh File Path** and **Direct3D-V3 Clean | Refine (Mesh File, Low VRAM)**, then connect the mesh path to the refiner. Use **Refine (Connected Mesh, Low VRAM)** if your workflow already provides a mesh object.
2. Connect the reference image and its matching foreground mask, with white as foreground. For an image with transparency, enable `use_alpha`.
3. Start with `resolution = 1536`, `stage_mode = single_stage` and `accel_mode = lossless`. Queue the workflow to export a raw mesh.
4. For more speed, switch `accel_mode` to `fast` or `fastest`. Use a fixed seed when comparing results.

**Settings notes**

* `accel_mode` needs `dtype = fp16`. The INT8 layers in `fast` and `fastest` are calibrated for the 1536 stage. On 512 and 1024 these modes use the lossless kernels plus INT8 attention.
* `preserve_scale` keeps the full normalized mesh extent and reports the token count needed if it exceeds the cap. `shrink_to_fit` shrinks the extent until the mesh fits. More tokens means more detail, more time and more memory.
* `attention_backend` (`efficient` or `cudnn_fp16`) applies when `accel_mode = off`.
* Use FP16 on RTX 20-series GPUs. The 512 and 1024 stages suit GPUs with less VRAM.

**Environment switches (optional)**

| Variable | Effect |
| --- | --- |
| `DIRECT3D_V3_ACCEL=0` | Disable all acceleration |
| `DIRECT3D_V3_PIN_WEIGHTS=0` / `DIRECT3D_V3_PIN_SKIPS=0` | Never page-lock system RAM (identical output, about 1% slower). By default RAM is pinned only when plenty is free. |

## Credits

ComfyUI adapter, low-VRAM pipeline and SuperFast acceleration by **[@MostAadTech](https://github.com/Mstafa-awad)**.

Built on [Direct3D-V3](https://github.com/DreamTechAI/Direct3D-V3), the [Direct3D-V3 geometry checkpoints](https://huggingface.co/wushuang98/Direct3D-V3) and [DINOv3](https://huggingface.co/facebook/dinov3-vith16plus-pretrain-lvd1689m). Each archive includes the adapter and upstream code licenses. Model weights download separately under the terms on their model pages.


## 🚀 SUPPORT MOSTAADTECH

### ❤️ Enjoying this project / workflow?

I'm **MostAadTech**, I create FREE ComfyUI workflows, local AI tools, 3D pipelines, and open-source projects.

If this project or workflow helped you, **please consider following me or supporting my work**. It helps me keep building, testing, and releasing more free tools and workflows.

---

## 💜 Support Me on Patreon

👉 **[Support MostAadTech on Patreon](https://www.patreon.com/cw/MostafaAwad/membership)**

Your support helps me spend more time developing **FREE AI tools, ComfyUI workflows, and 3D pipelines**.

---

## 🌐 Follow MostAadTech

* ▶️ **[YouTube](https://www.youtube.com/@MostAadTech)** — Tutorials, workflows & AI projects
* 📸 **[Instagram](https://www.instagram.com/mostaadtech/)** — Projects, updates & behind the scenes
* 𝕏 **[X / Twitter](https://x.com/MostAadTech)** — Updates, releases & experiments
* 💻 **[GitHub](https://github.com/Mstafa-awad)** — Open-source projects & code

---

### ⭐ One Follow Helps

**Follow • Star • Share • Support**

Every follow, GitHub star, share, and Patreon supporter helps me continue making **FREE tools for the AI community.**

**Thank you for supporting MostAadTech! ❤️**
