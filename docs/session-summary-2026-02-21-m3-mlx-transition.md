# Session Summary: 2026-02-21 - M3 Hardware Optimization & MLX Transition (Rev 2)

## Strategic Context
This session focused on reconciling the **QMD (Query Markup Documents)** fine-tuning suite with the user's **Apple Silicon M3** hardware. While the QMD CLI (search/rerank) is optimized for Metal, the existing `finetune/` pipeline is strictly CUDA-based (PyTorch/TRL). We have identified **MLX-LM** (present in the sibling directory `../mlx-lm`) as the primary vehicle for enabling high-performance local SFT training on the M3.

## Key Decisions & Rationale
1.  **Runtime Pivot:** Determined that `bun` currently lacks support for dynamic extension loading required by `sqlite-vec` on macOS.
    - **Rationale:** Switched to using the `./qmd` shell wrapper (which defaults to Node.js) for CLI operations to ensure vector search availability.
2.  **Dataset Preparation:** Executed `prepare_data.py` to consolidate 12 input sources.
    - **Observation:** Identified significant duplication in the current `data/*.jsonl` glob (specifically `v2` and `v3` overlap).
3.  **Hardware Alignment:** Formulated a "Hybrid Training" strategy:
    - **Local SFT:** Use `mlx-lm` on M3 for the Supervised Fine-Tuning stage.
    - **Cloud GRPO:** Continue using HuggingFace Jobs for RL/GRPO until MLX equivalents are ready.

## Path Mapping & Environment
To avoid relative path friction when jumping between projects, use these absolute roots:
- **PROJECT_ROOT:** `/Users/rymalia/projects/qmd`
- **MLX_ROOT:** `/Users/rymalia/projects/mlx-lm`

**Environment:**
- Use the virtual environment in `MLX_ROOT/.venv/` for all MLX commands.
- **Tooling:** Ensure `mlx-lm` is installed in editable mode (`pip install -e .`) within that venv to ensure latest features and templates are available.

## System State & Technical Warnings
- **Dependencies:** `bun install` completed. `picomatch` and `node-llama-cpp` are correctly resolved.
- **Vector Search:** **CRITICAL:** Do not run vector-related commands via `bun src/qmd.ts`. Always use `./qmd` or `npx tsx src/qmd.ts`.
- **Hardware:** M3 GPU (Metal) is active and confirmed working for inference (11.8 GB free VRAM).

## Initial Assessment

**mlx-lm is extremely useful for a M3 Mac and can absolutely replace the need for CUDA for most training tasks.** 
Since you already have the mlx-lm project in your workspace, you are in a perfect position to bridge the two.

1. Does `qmd` HAVE to use CUDA?
No. While the current finetune/ scripts in QMD are written using trl and transformers (which are heavily optimized for CUDA), MLX is the native Mac equivalent.
- The Catch: The current GRPO (Reinforcement Learning) stage in QMD uses trl.GRPOTrainer, which is currently very difficult to run on Mac because it relies on specific DeepSpeed/CUDA kernels.
- The Solution: You can use mlx-lm for the SFT (Supervised Fine-Tuning) stage locally on your M3. This is usually the most time-consuming part and where mlx-lm shines.

2. Why mlx-lm is better for your M3:
- Unified Memory: On your M3, the GPU and CPU share the same 12GB+ of RAM. mlx-lm is designed to exploit this, allowing you to fine-tune 7B or even 14B models that would be impossible on a mid-range NVIDIA card.
- Speed: mlx-lm uses Apple's Accelerate and Metal frameworks directly. For a 1.7B model like Qwen3, SFT training would be near-instantaneous compared to a CPU-fallback.
- Quantized Training: mlx-lm supports training directly on 4-bit quantized models (QLoRA), which saves massive amounts of memory.

3. How it compares to the current QMD setup:

  ┌──────────────┬────────────────────────────┬──────────────────────────────────┐
  │ Feature      │ QMD finetune/ (Current)    │ mlx-lm (On M3)                   │
  ├──────────────┼────────────────────────────┼──────────────────────────────────┤
  │ Backend      │ PyTorch / CUDA             │ MLX / Metal                      │
  │ SFT Support  │ Excellent (via SFTTrainer) │ Excellent (via `mlx_lm.lora`)    │
  │ GRPO Support │ Native (via trl)           │ No (Requires custom reward loop) │
  │ Efficiency   │ High (on NVIDIA)           │ Highest (on M3)                  │
  │ Setup        │ Complex (Conda/NVCC)       │ Simple (`pip install mlx-lm`)    │
  └──────────────┴────────────────────────────┴──────────────────────────────────┘

4. Recommendation for your Workflow:
Since you have both projects side-by-side:

	1. Use `mlx-lm` for SFT: You can convert the QMD data/train/train.jsonl to the format mlx-lm expects (usually a simple mapping of fields) and run the SFT stage locally on your M3.
	2. Use Cloud for GRPO: If you want to use the specific GRPO reinforcement learning logic currently in QMD, continue using the hf jobs (HuggingFace Cloud) for that specific step, as it's built on CUDA-only libraries.
	3. Experimental: You could theoretically implement a "Reward-based SFT" in MLX by filtering your dataset for high-scoring examples (using reward.py) and training on those using mlx-lm.lora.



## Implementation Addendum: The "MLX Bridge"

### 1. Data Prep & Naming (Important)
- **Current State:** `finetune/data/train/train.jsonl` already contains a `messages` array, which `mlx-lm` natively supports.
- **Task:** Instead of a complex conversion, simply **move/link** and **rename** the files to match `mlx-lm` conventions.
- **Target Dir:** `PROJECT_ROOT/finetune/data/mlx/`
- **File Names:**
  - `train.jsonl` (from `train/train.jsonl`)
  - **`valid.jsonl`** (RENAME from `train/val.jsonl` to satisfy `mlx-lm` defaults).

### 2. Chat Templates & Qwen3
- `mlx-lm` generally handles Qwen2/3 via ChatML. If specific template issues arise, the `qmd` fine-tuning logic uses `/no_think` in the system/user prompt which should be preserved in the `messages` array.

### 3. Training & Deployment Workflow (The "Final Mile")
To move from local M3 training back to the `qmd` search engine:

1.  **Train:**
    ```bash
    cd PROJECT_ROOT/finetune
    source $MLX_ROOT/.venv/bin/activate
    python -m mlx_lm.lora --config configs/mlx_sft.yaml
    ```
2.  **Verify:** Check loss curves in logs and run a quick generation test using `mlx_lm.generate` with the `--adapter-path` flag to ensure the model isn't hallucinating.
3.  **Fuse:**
    ```bash
    python -m mlx_lm.fuse --model Qwen/Qwen3-1.7B --adapter-path PROJECT_ROOT/finetune/adapters/ --save-path PROJECT_ROOT/finetune/merged_model/
    ```
4.  **Quantize/Export for QMD:**
    - `qmd` uses `node-llama-cpp`, which requires **GGUF**.
    - Since `mlx-lm` typically exports to safetensors, use the existing `PROJECT_ROOT/finetune/convert_gguf.py` on the `merged_model/` directory to produce the final `.gguf` file.

## Unfinished Work & Next Steps (For Next Agent)

### 1. Configure `mlx_sft.yaml`
Create `PROJECT_ROOT/finetune/configs/mlx_sft.yaml` with:
- **model:** `Qwen/Qwen3-1.7B`
- **data:** `PROJECT_ROOT/finetune/data/mlx/`
- **adapter_path:** `PROJECT_ROOT/finetune/adapters/`
- **lora_layers:** 16 (or match `finetune/configs/sft.yaml` modules)
- **batch_size:** 4
- **learning_rate:** `2e-4`
- **epochs:** 5

### 2. Verify and Move Data
Execute the file renaming (`val.jsonl` -> `valid.jsonl`) and verify the `messages` structure once more before starting the first epoch.

## Summary Statistics
- **Model Size:** 1.7B parameters.
- **Training Samples:** 5,773.
- **Status:** QMD v1.0.8 healthy; Local SFT Ready for implementation.
