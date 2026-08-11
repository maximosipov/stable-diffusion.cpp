# Building on a 2019 MacBook Pro / AMD Radeon Pro 5500M (4 GB)

Self-contained build + run notes for **text-to-image fully on the AMD Radeon Pro
5500M** of a 2019 Intel MacBook Pro, via ggml's **Vulkan backend over MoltenVK**
(plus a Metal build for comparison, and a CPU build as the correctness
reference). Nothing here depends on an external script — clone this fork, follow
this file.

Target: **MacBook Pro 16" 2019 · AMD Radeon Pro 5500M (4 GB, RDNA1 / Navi 14) ·
Intel UHD 630 (present, excluded) · macOS 14.x · MoltenVK 1.4.2.**

**No source patches are needed.** stable-diffusion.cpp builds cleanly for this
hardware as-is. This branch adds only these notes — but two of the flags below
are non-obvious and both produce *silently wrong output* if you omit them, which
is exactly why they need writing down.

## The two flags that decide whether this works

| Flag | If you omit it |
|---|---|
| `-DGGML_METAL=OFF` at build time (**in addition to** `-DSD_METAL=OFF`) | ggml auto-enables Metal on macOS, Metal wins device 0 at runtime, and your "Vulkan" build runs Metal — which on a discrete GPU trips the macOS watchdog (`kIOAccelCommandBufferCallbackErrorTimeout`) and never finishes an image |
| `--diffusion-conv-direct` at run time | the default im2col+matmul convolution is numerically broken on RDNA1/MoltenVK: you get **full-frame colourful noise**, not an error. Conv-direct is also **~3× faster** |

Everything else on this page is ordinary setup.

## 1. Toolchain (Homebrew)

```bash
brew install cmake libomp \
  vulkan-headers vulkan-loader molten-vk glslang shaderc spirv-headers
```

- `vulkan-loader` — link the **loader**, not `libMoltenVK` directly, and pick the
  driver at runtime via `VK_ICD_FILENAMES`. Lets you A/B MoltenVK against other
  ICDs without rebuilding.
- `glslang` + `shaderc` — ggml-vulkan compiles its compute shaders at build time.
- `spirv-headers` — ggml-vulkan `#include`s `spirv/unified1/spirv.hpp`.
- `libomp` — Clang on macOS has no bundled OpenMP.

MoltenVK **1.4.2 or newer** is strongly preferred: up to 1.4.1 it reported a
64-lane subgroup where Metal runs 32, which silently corrupts subgroup shaders
across the whole ggml stack.

## 2. Clone

```bash
git clone --recursive https://github.com/maximosipov/stable-diffusion.cpp.git
cd stable-diffusion.cpp
git checkout macbook-pro-2019-radeon-5500m-4gb
git submodule update --init --recursive   # ggml + libwebp/libwebm are submodules
```

## 3. Build

Each backend gets its own build dir so they coexist. The CLI binary is `sd-cli`;
`sd-server` is also produced and is what you want in practice — model load and
on-the-fly quantization dominate a 4-step turbo generation.

### Vulkan / MoltenVK — the GPU path

```bash
P="$(brew --prefix)"
cmake -B build-vulkan \
  -DSD_VULKAN=ON \
  -DSD_METAL=OFF -DGGML_METAL=OFF \
  -DVulkan_INCLUDE_DIR="$P/opt/vulkan-headers/include" \
  -DVulkan_LIBRARY="$P/opt/vulkan-loader/lib/libvulkan.dylib" \
  -DVulkan_GLSLC_EXECUTABLE="$P/opt/shaderc/bin/glslc" \
  -DVulkan_GLSLANG_VALIDATOR_EXECUTABLE="$P/opt/glslang/bin/glslangValidator" \
  -DOpenMP_ROOT="$P/opt/libomp" \
  -DCMAKE_CXX_FLAGS="-I$P/opt/spirv-headers/include" \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-vulkan --config Release -j"$(sysctl -n hw.ncpu)"
```

Gate — the binary must link Vulkan and not Metal:

```bash
otool -L build-vulkan/bin/sd-cli | grep -i vulkan   # expect libvulkan
otool -L build-vulkan/bin/sd-cli | grep -i Metal    # expect no output
```

### CPU — the correctness reference

```bash
cmake -B build-cpu \
  -DSD_METAL=OFF -DGGML_METAL=OFF -DSD_VULKAN=OFF -DGGML_VULKAN=OFF \
  -DOpenMP_ROOT="$(brew --prefix)/opt/libomp" \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-cpu --config Release -j"$(sysctl -n hw.ncpu)"
```

This is not optional if you are changing anything: "produced an image" is not a
correctness check on this card, because the broken convolution path also produces
an image. The check is the same prompt and seed on CPU and GPU, compared.

### Metal — the comparison path (does not work here; kept for measurement)

```bash
cmake -B build-metal \
  -DSD_METAL=ON -DSD_VULKAN=OFF \
  -DOpenMP_ROOT="$(brew --prefix)/opt/libomp" \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-metal --config Release -j"$(sysctl -n hw.ncpu)"
```

## 4. Runtime environment (Vulkan path)

```bash
P="$(brew --prefix)"
export VK_ICD_FILENAMES="$P/opt/molten-vk/etc/vulkan/icd.d/MoltenVK_icd.json"
export DYLD_LIBRARY_PATH="$P/opt/molten-vk/lib:$P/opt/vulkan-loader/lib:$DYLD_LIBRARY_PATH"
# Pin the AMD dGPU. Without this, MoltenVK also enumerates the Intel UHD 630 as
# Vulkan1 -- and the UHD advertises 32 GiB of shared system memory against the
# Radeon's real 4 GiB, so any "pick the biggest GPU" heuristic picks the wrong one.
export GGML_VK_VISIBLE_DEVICES=0
```

## 5. Build gate — confirm the AMD enumerates

```bash
GGML_VK_VISIBLE_DEVICES=0 ./build-vulkan/bin/sd-cli --list-devices
```

Expected (only the AMD visible, because of the pin):

```
ggml_vulkan: Found 1 Vulkan devices:
ggml_vulkan: 0 = AMD Radeon Pro 5500M (MoltenVK) | uma: 0 | fp16: 1 | bf16: 0 | warp size: 32 | shared memory: 65536 | int dot: 0 | matrix cores: none
Vulkan0	AMD Radeon Pro 5500M
```

`warp size: 32` confirms MoltenVK >= 1.4.2 (1.4.1 and earlier said `64`). Drop
the pin and you will also see `Vulkan1  Intel(R) UHD Graphics 630`, which
confirms the ICD sees both GPUs and that the pin is what isolates the dGPU.
`Vulkan0` is the device name you pass to `--backend`.

## 6. Generate

```bash
# SD-Turbo -- the daily driver. 4 steps, cfg 1.
./build-vulkan/bin/sd-cli --diffusion-conv-direct \
  -m models/sd_turbo.safetensors --type q8_0 \
  -p "a red apple on a wooden table, studio lighting" \
  --steps 4 --cfg-scale 1 --sampling-method euler -W 512 -H 512 --seed 42 -o apple.png

# SDXL-Turbo -- better photorealism, at the edge of 4 GB, VAE on the CPU.
./build-vulkan/bin/sd-cli --diffusion-conv-direct --vae-on-cpu \
  -m models/sd_xl_turbo_1.0_fp16.safetensors --type q8_0 \
  -p "an astronaut riding a horse on the moon, detailed photograph" \
  --steps 4 --cfg-scale 1 --sampling-method euler -W 512 -H 512 --seed 42 -o astronaut.png

# SD 1.5 -- baseline, best LoRA/ControlNet ecosystem. 20 steps, cfg 7.
./build-vulkan/bin/sd-cli --diffusion-conv-direct \
  -m models/v1-5-pruned-emaonly-fp16.safetensors \
  -p "an astronaut riding a horse on the moon, detailed photograph" \
  --steps 20 --cfg-scale 7 --sampling-method euler_a -W 512 -H 512 --seed 42 -o sd15.png
```

## 7. What this card can do

Backend comparison, SD 1.5 fp16, 512×512, seed 42, identical prompt, verified
against the CPU reference image:

| Backend | Fits? | Correct? | Speed |
|---|---|---|---|
| Metal | targets the AMD, 4278 MB set | ❌ **GPU watchdog timeout** | — (the shader library alone takes 40–1000 s to load) |
| Vulkan, no extra flags | ✅ ~2.6 GB | ❌ **colourful noise** | ~11 s/step |
| **Vulkan + `--diffusion-conv-direct`** | ✅ ~2.6 GB | ✅ **matches CPU** | **~3.4 s/step** |
| CPU (ground truth) | n/a | ✅ | ~24 s/step |

Model fit and throughput:

| Model | Load config | VRAM | s/step | Notes |
|---|---|---|---|---|
| SD-Turbo | `--type q8_0` | **2049 MB** | **~1.7** | comfortable; a 4-step image in ~7 s of compute |
| SD 1.5 | fp16 | 2035 MB | ~3.4 | baseline, 20 steps |
| SDXL-Turbo | `--type q8_0 --vae-on-cpu` | **3836 MB** | ~9.3 | fits, but tight against the 4278 MB usable |

Check fit against **4278 MB** usable (4080 MiB), not 4096.

## This card's landmines

- **Peak VRAM is at VAE decode, not sampling.** A model can load comfortably and
  still die at the last step. `--vae-on-cpu` is the standard 4 GB lever.
- **VAE decode in fp16 → NaN → black/noise image**, on SDXL in particular. Keep
  the VAE off the GPU (`--vae-on-cpu`) or use an fp16-fixed VAE (madebyollin).
  The same flag fixes it and lowers the peak — usually the right default here.
- **bf16 weights produce NaN on Vulkan/AMD** (this GPU reports `bf16: 0`). Prefer
  fp16 or GGUF-quantized weights; avoid bf16 checkpoints.
- **`--diffusion-fa` only works on NVIDIA coopmat2.** This card reports
  `matrix cores: none` — leave flash attention off.
- **4 GB fit:** SD 1.5 / SD-Turbo (fp16 ≈2 GB, q8 ≈1 GB) are comfortable.
  SDXL / SDXL-Turbo need q8 UNet + CPU VAE. DiT models (PixArt-Σ, SD3.5-M,
  Flux-schnell) need quantized weights **and** the text encoder on CPU
  (`--backend ...,clip=cpu`), and are borderline.

## Verify

Run the same prompt, seed, sampler, step count and resolution on `build-cpu` and
on `build-vulkan --diffusion-conv-direct`, and compare the images. A recognisable,
matching subject means the whole path is correct. Colourful noise means
`--diffusion-conv-direct` did not take effect; a hang or
`kIOAccelCommandBufferCallbackErrorTimeout` means Metal got compiled in.

Wall clock includes model load and on-the-fly quantization of the safetensors
checkpoint (~30 s for SD-Turbo), which dominates a 4-step generation — use
`sd-server` to amortize it.

## Note on the ggml submodule

This project vendors [leejet/ggml](https://github.com/leejet/ggml), a separate
lineage from the [ggml fork](https://github.com/maximosipov/ggml/tree/macbook-pro-2019-radeon-5500m-4gb)
that carries this hardware's Vulkan fixes. Those fixes are **not** in this build,
and have not been needed: on MoltenVK 1.4.2 the subgroup misreport that motivated
most of them is corrected in the driver, and the diffusion path's own defect is
handled by `--diffusion-conv-direct`. Whether the integer-dot enablement from
that fork would speed up diffusion here is untested.
