# Direct3D-V3 for ComfyUI — INT8 SuperFast (Windows)

**Made by Mostafa Awad ([@MostAadTech](https://github.com/Mstafa-awad)) —
[github.com/Mstafa-awad/Direct3D-V3_Comfyui_Nodes](https://github.com/Mstafa-awad/Direct3D-V3_Comfyui_Nodes)**

Turns a coarse 3D mesh (from TRELLIS.2, Pixal3D, Hunyuan3D, …) into a highly detailed mesh with
[Direct3D-V3](https://huggingface.co/wushuang98/Direct3D-V3), on 8 GB GPUs.

- **One node:** plug the `MESH` from your TRELLIS.2 / Pixal3D nodes straight in. No loader or file paths needed.
- **Mesh out:** connect the output to ComfyUI's **Save 3D Model** or any mesh node, or tick `save_file` to export GLB / OBJ / PLY / STL directly.
- **Up to about 2× faster** with custom Triton INT8 / FP16 kernels.
- **Small download:** INT8 models are 5.9 GB instead of 18 GB.
- **Multiview node:** add back, side, top or bottom photos.
- **Sharp-edges mode** (dual contouring) for buildings and hard-surface models.

## Basic workflow

```
[Load Image] ──────────────┬──────────────► image
                           ▼
[TRELLIS.2 / Pixal3D] ── MESH ──► [Direct3D-V3 Refine] ── mesh ──► [Save 3D Model]
                                       (or tick save_file + pick glb / obj / ply / stl)
```

Use the **same image** for the coarse model and for Direct3D-V3. An image with a transparent background works best;
otherwise connect a `mask`.

## Install (Windows, NVIDIA GPU)

1. Download the zip that matches your setup:

   | Zip | Python | CUDA / torch | GPUs |
   |---|---|---|---|
   | `ComfyUI-Direct3D-V3-Windows-cu128_int8_v2.zip` | 3.12 | 12.8 / torch 2.8 | NVIDIA RTX, 8 GB+ (tested on RTX 5050) |
   | `ComfyUI-Direct3D-V3-Windows-cu130_int8_v2.zip` | 3.13 | 13.0 / torch 2.12 | NVIDIA RTX, 8 GB+, needs a CUDA 13 driver |

2. Unzip into `ComfyUI/custom_nodes/` so that you get `ComfyUI/custom_nodes/ComfyUI-Direct3D-V3/`.
3. Run `setup.cmd` inside that folder. It creates a private Python runtime, so ComfyUI's own packages are not touched,
   and downloads the INT8 models.
   - `setup.cmd`: only the 1536 model, plus the image encoder.
   - `setup.cmd --stage all`: all stages (512, 1024, 1536), needed for `start_from`.
   - `setup.cmd --birefnet`: also adds automatic background removal for images without alpha.
4. Restart ComfyUI and search for **Direct3D-V3 Refine**.

The models come from the INT8 Hugging Face repository. They are the original Direct3D-V3 weights stored as INT8, about
3× smaller with near-identical output. `setup.cmd --original` downloads the original full-size weights instead.

## Direct3D-V3 Refine — settings

| Setting | What it does | Default |
|---|---|---|
| `mesh` | Coarse mesh (`MESH` output of TRELLIS.2 / Pixal3D / Hunyuan3D nodes) | – |
| `image` | The image the coarse mesh was made from | – |
| `resolution` | Final detail: **1536** = most detail, **1024** = balanced, **512** = quick preview | 1536 |
| `start_from` | Run lower stages first, e.g. 1024 → 1536. *same as resolution* = one stage (usually best) | same as resolution |
| `steps` | Denoising steps. 8 is enough; more is slower and rarely better | 8 |
| `guidance` | How strongly to follow the image | 4.0 |
| `seed` | Change for a different variation | 0 |
| `speed` | **lossless** (same quality, ~1.6× faster), **fast** (~2×, tiny change), **fastest** (~2.2×), **off** (original) | lossless |
| `max_tokens` | Patch limit. About 150k fits an 8 GB GPU at 1536 | 150000 |
| `fit_large_meshes` | On: very large meshes are scaled down to fit `max_tokens`. Off: stop with an error | on |
| `sharp_edges` | Keep hard edges and corners (buildings, props, mechanical). Off = smooth surfaces (characters, organic) | off |
| `save_file` | Also save the mesh to `ComfyUI/output/Direct3D-V3/` | off |
| `file_format` | Format for `save_file`: glb, obj, ply, stl | glb |
| `filename_prefix` | File name start for `save_file` | Direct3D-V3 |

Optional inputs:

| Setting | What it does | Default |
|---|---|---|
| `mask` | Foreground mask (white = object), if the image has no transparency | – |
| `guidance_method` | **cfg** standard, **apg** slightly gentler guidance, **fdg** extra guidance on fine detail | cfg |
| `skip_guidance_last_steps` | Run the last N steps without guidance (about 6% faster each, same quality at 1) | 0 |

**Outputs:**
- `mesh`: the refined `MESH`, for Save 3D Model or any mesh node.
- `file_path`: the saved file, or empty when `save_file` is off.

## Direct3D-V3 Refine Multiview

Same settings, plus optional `back`, `left`, `right`, `top` and `bottom` images, each with an optional mask. Every
surface patch takes its detail from the view that sees it most directly. Each extra view adds about one more sampling
pass. `multiview_speed = fast` makes views share one guidance pass, about 45% less time per view with a tiny change.

Camera directions: `left` is the model's left side (camera on +x, the right side of the front image) and `right` is the
model's right side. `top` shows the front of the model at the image's bottom edge; `bottom` shows it at the top edge.

## Tips

- **Very big or complex meshes** (houses, scenes) can need more than 150k patches. `fit_large_meshes` scales them down
  automatically. Running at 1024 with the mesh scaled down often keeps more image detail than 1536 heavily scaled down.
- **Watertight coarse meshes** need fewer patches than meshes with double walls or hidden inner faces.
- **Memory:** close other GPU apps. The node unloads ComfyUI models before it runs and uses a separate worker process.

## Speed (RTX 5050 Laptop 8 GB, 1536, 152k patches, 8 steps)

| speed | Sampling time |
|---|---|
| off (original) | 925 s |
| lossless | 576 s |
| fast | 459 s |
| fastest | 425 s |

## Older workflows

The previous nodes (mesh-file loader, connected-mesh refine) are still available under **Direct3D-V3/legacy**, so old
workflows keep working.

## Credits and license

- **Node, acceleration, INT8 models, multiview, guidance options and sparse dual contouring:**
  Mostafa Awad (@MostAadTech), MIT License (see `LICENSE`).
- **Direct3D-V3 model and code:** DreamTech, MIT License (see `vendor/Direct3D-V3/LICENSE`).
- **Image encoder:** Meta DINOv3, under the DINOv3 License.
- **Optional background removal:** BiRefNet (MIT).

If you share results or build on this work, please credit **@MostAadTech**.
