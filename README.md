# Direct3D-V3 Clean for ComfyUI

ComfyUI adapter made by [@MostAadTech](https://github.com/MostAadTech).

Windows custom nodes for refining a coarse 3D mesh with Direct3D-V3 and a reference image. The node exports raw marching-cubes geometry so you can handle decimation, smoothing, and cleanup in your own workflow.

## Download

| Windows build | Worker Python | PyTorch |
| --- | --- | --- |
| [cu128 ZIP](./ComfyUI-Direct3D-V3-Windows-cu128.zip?raw=1) | 3.12 x64 | 2.8.0+cu128 |
| [cu130 ZIP](./ComfyUI-Direct3D-V3-Windows-cu130.zip?raw=1) | 3.13 x64 | 2.12.1+cu130 |

Both archives target NVIDIA RTX 50-series GPUs (SM 120) and include prebuilt voxelizers. You need a current NVIDIA driver and the listed 64-bit Python version. The worker has its own Python environment, so ComfyUI can keep its existing Python and PyTorch. The prebuilt extensions let setup run without a CUDA Toolkit or Visual Studio.

**Build status.** The cu128 configuration is the one used on an RTX 5050 Laptop GPU with 8 GB VRAM. The cu130 build is experimental and awaiting runtime testing. Its extensions were compiled with CUDA Toolkit 13.5 against PyTorch cu130. Use a driver with CUDA 13.5 support for the initial test.

## Install

1. Choose one ZIP and extract its `ComfyUI-Direct3D-V3` folder into `ComfyUI/custom_nodes`. Use a fresh folder when switching builds, since both register the same node IDs.
2. Request access to [DINOv3 ViT-H+ on Hugging Face](https://huggingface.co/facebook/dinov3-vith16plus-pretrain-lvd1689m). If setup prompts for authentication, use a read token from the account with access.
3. Run `setup.cmd` from the extracted node folder. It creates the private `.runtime` environment and downloads PyTorch, the worker dependencies, DINOv3, and the 1536 geometry checkpoint. Allow several gigabytes of disk space for the downloads.
4. Restart ComfyUI normally.

For cascade mode, download all three geometry stages by running `setup.cmd --models-only --stage all` after setup. Add `--birefnet` to cache the optional background remover.

## Use in ComfyUI

1. Add **Direct3D-V3 Clean | Mesh File Path** and **Direct3D-V3 Clean | Refine (Mesh File, Low VRAM)**. Connect the mesh path to the refiner. Use **Refine (Connected Mesh, Low VRAM)** if your workflow already provides a mesh object.
2. Connect the reference image and its matching foreground mask. Use white for foreground. For an image with transparency, enable `use_alpha`.
3. Start with `resolution = 1536` and `stage_mode = single_stage` after the default setup. Queue the workflow to export a raw mesh.

`preserve_scale` keeps the full normalized mesh extent and reports the token count required if it exceeds the cap. `shrink_to_fit` reduces the extent until the mesh fits the cap. Larger accepted token counts can increase runtime and memory use substantially.

`efficient` is the default attention backend. With FP16, you can try `cudnn_fp16` for eligible attention calls. It can change numerical rounding and the resulting surface. Use a fixed seed when comparing exports.

## Credits

Built on [Direct3D-V3](https://github.com/DreamTechAI/Direct3D-V3) with [Direct3D-V3 geometry checkpoints](https://huggingface.co/wushuang98/Direct3D-V3) and [DINOv3](https://huggingface.co/facebook/dinov3-vith16plus-pretrain-lvd1689m). Each archive includes the adapter and upstream code licenses. Model weights are downloaded separately under the terms on their model pages.
