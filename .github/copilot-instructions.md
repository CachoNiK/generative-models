<!-- Copied guidance for AI coding agents specific to Stability-AI/generative-models -->
# Copilot instructions for contributors and AI agents

This file gives concentrated, actionable context so an AI coding agent can be productive immediately.

- Big picture:
  - The project is a config-driven generative-models repo. Core runtime is in `sgm/` and components are assembled from YAML configs in `configs/` using `instantiate_from_config()` (see README).
  - Major areas:
    - `sgm/modules/` — model building blocks (encoders, denoisers, samplers, guiders). Example files: `sgm/modules/diffusionmodules/denoiser.py`, `sgm/modules/diffusionmodules/sampling.py`, `sgm/modules/diffusionmodules/guiders.py`, `sgm/modules/encoders/modules.py`.
    - `scripts/` — inference and demo entrypoints (e.g. `scripts/sampling/simple_video_sample_4d2.py`, `scripts/sampling/simple_video_sample_4d.py`, `scripts/demo/gradio_app_sv4d.py`). Use these to run/validate inference changes.
    - `configs/` — YAML-driven configs that are merged left-to-right when passed to `main.py --base` for training or inference. Small code changes often require corresponding config updates.

- Key developer workflows (commands you can run directly):
  - Install & dev env (recommended from README):
    ```bash
    python3 -m venv .pt2
    source .pt2/bin/activate
    pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
    pip3 install -r requirements/pt2.txt
    pip3 install .
    ```
  - Run an inference demo (example):
    ```bash
    python scripts/sampling/simple_video_sample_4d2.py --input_path assets/sv4d_videos/camel.gif --output_folder outputs
    # place model weights in checkpoints/ (e.g. sv4d2.safetensors)
    ```
  - Run example training via merged configs:
    ```bash
    python main.py --base configs/example_training/toy/mnist_cond.yaml
    ```
  - CI/test helper in `pyproject.toml`: see `tool.hatch.envs.ci.scripts.test-inference` for how inference tests are run (installs specific torch wheel, `pip install -r requirements/pt2.txt`, then `pytest -v tests/inference/test_inference.py`).

- Patterns & conventions an agent must follow:
  - Config-first: prefer adding/adjusting YAML entries in `configs/` when behavior should be configurable; code changes should keep backward-compatible config keys where possible.
  - Instantiate-from-config: new components should be constructible via `instantiate_from_config()` so they can be plugged into existing configs.
  - Separation of concerns: guiders (guidance logic) live separately from samplers; avoid coupling changes across `guiders.py` and `sampling.py` without tests.
  - Denoiser framework: many files assume continuous-time denoisers; consult `sgm/modules/diffusionmodules/denoiser.py` before refactoring denoising math.

- Integration points & external dependencies:
  - Model weights are expected under `checkpoints/` for inference scripts.
  - Third-party heavy deps (PyTorch wheels) are installed explicitly in CI/test scripts; use `requirements/pt2.txt` for other deps.
  - Data pipelines: training at scale uses the external `sdata` package (installed in README via Git URL).

- Where to make small, self-contained changes (good for PRs):
  - Add new CLI flags to `scripts/sampling/*` for inference options (keep default behavior intact).
  - Add unit/integration tests in `tests/inference/` that run a lightweight forward pass (the repo already has `tests/inference/test_inference.py`).
  - Add new config fragments in `configs/` and reference them in `configs/example_training/` for reproducibility.

- Testing & verification guidance for agents:
  - Local quick test: run a targeted script, e.g. `python scripts/sampling/simple_video_sample_4d.py --input_path assets/sv4d_videos/test_video1.mp4 --output_folder /tmp/out` with a tiny number of steps (`--num_steps`).
  - Run the single inference test: `pytest -q tests/inference/test_inference.py` after installing `requirements/pt2.txt`.
  - For CI-like reproduction, check `pyproject.toml` -> `tool.hatch.envs.ci.scripts.test-inference` to mimic environment (torch wheel + requirements + pytest).

- Quick heuristics for code edits:
  - If the change affects tensor shapes or conditioning keys, search for `input_key` and `emb_models` in `sgm/modules/encoders` and update corresponding `conditioner_config` examples in `configs/`.
  - When adding network blocks, wire them into `network_config` and ensure they are serializable via `instantiate_from_config()`.
  - Keep sampling behavior configurable (expose params in `sampler_config`) rather than hardcoding in scripts.

- Where to look for examples of important patterns:
  - Config-driven model assembly: `configs/` + `main.py` (entrypoint that merges `--base` configs).
  - Sampling & inference entrypoints: `scripts/sampling/simple_video_sample_4d2.py`, `scripts/sampling/simple_video_sample.py`.
  - Core modules: `sgm/modules/diffusionmodules/`, `sgm/modules/encoders/`.

If anything here is unclear or you want a different level of detail (e.g., concrete constructor signatures to prefer), tell me which area to expand and I will iterate.
