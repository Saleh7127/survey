# nv-tlabs__vipe

nv-tlabs__vipe is a Python CLI tool and Hydra-based pipeline for video processing, annotation, and SLAM/geometry tasks, aimed at developers working with deep learning and visual SLAM. It provides a configurable, CUDA-accelerated pipeline that integrates Depth-Anything-3 and related vision/depth priors, with support for multiple video stream backends such as raw MP4 files and frame directories.

## Table of Contents

- [Requirements / Prerequisites](#requirements--prerequisites)
- [Installation / Setup](#installation--setup)
- [Quick Start / Basic Usage](#quick-start--basic-usage)
- [Configuration](#configuration)
- [Advanced Usage / Commands / API](#advanced-usage--commands--api)
- [Features](#features)
- [Demo / Examples](#demo--examples)
- [Contributing](#contributing)
- [License](#license)

## Requirements / Prerequisites

- Python 3.10 or later.
- A CUDA-capable GPU with recent NVIDIA drivers is strongly recommended for SLAM and geometry-accelerated components.
- Supported platforms: Linux (primary); other platforms may work but are not the main target.
- Core runtime dependencies (installed as Python packages):
  - `torch` (with CUDA build recommended)
  - `hydra-core`
  - `omegaconf`
  - `xformers`
  - `depth-anything-3` (via optional `dav3` extra)
  - Plus standard libraries such as `numpy`, `einops`, `tqdm`, `imageio[ffmpeg]`, `opencv-python`, `transformers`, `timm`, `kornia`, `rerun-sdk`, and others declared in `pyproject.toml`.

## Installation / Setup

nv-tlabs__vipe is a Python package with CUDA-accelerated extensions. You will need a working Python environment with a compatible CUDA toolkit and compiler installed before building from source.

Install the project locally (editable or standard) from the repository root:

```bash
pip install .
```

Optional support for Depth-Anything-3 can be enabled via the `dav3` extra:

```bash
pip install ".[dav3]"
```

## Quick Start / Basic Usage

Run nv-tlabs__vipe either from the command line or as a Python module.

Command-line (recommended to start)

```bash
# Basic inference on a local MP4
vipe infer path/to/video.mp4 -o vipe_results

# Use a specific pipeline config (see configs/pipeline/)
vipe infer path/to/video.mp4 -p default

# Visualize results from a previous run
vipe visualize vipe_results --port 8080
```

Minimal Python entrypoint

```python
# save as run_vipe.py
from vipe.cli.main import main

if __name__ == "__main__":
    # behaves the same as the `vipe` CLI
    main()
```

Hydra-based programmatic usage

```python
import hydra
from omegaconf import DictConfig

from vipe.run import run  # adjust to the actual module path if needed

@hydra.main(version_base=None, config_path="configs", config_name="default")
def main(cfg: DictConfig) -> None:
    run(cfg)

if __name__ == "__main__":
    main()
```

This will use `configs/default.yaml` (which in turn selects `configs/streams/raw_mp4_stream.yaml` and a default pipeline) to process the input stream. For customizing streams or pipelines, see the [Configuration](#configuration) and [Advanced Usage / Commands / API](#advanced-usage--commands--api) sections.

## Configuration

nv-tlabs__vipe is configured via [Hydra](https://hydra.cc) config files. The main entry point uses `@hydra.main(config_path="configs", config_name="default")`, which loads a structured `DictConfig` that controls the entire pipeline.

Key config files:

- `configs/default.yaml`  
  Main application configuration:
  - High-level defaults (which stream, which pipeline, which SLAM profile)
  - Global toggles (e.g., `ray`, `prefilter`)
  - Hydra behavior (logging, run directory, output subdir)
  - Output behavior (result path, whether to save artifacts/SLAM map)

- `configs/pipeline/default.yaml`  
  Pipeline settings:
  - `instance`: fully qualified class for the annotation pipeline (e.g., `vipe.pipeline.default.DefaultAnnotationPipeline`)
  - `init`: camera model, intrinsics source (`geocalib`, `gt`, etc.), pipeline-specific parameters (e.g., `kf_gap_sec`, `phrases`, `add_sky`)
  - `post`: post-processing options (e.g., `depth_align_model`)

- `configs/slam/default.yaml`  
  SLAM-related options:
  - Depth priors and model selection (e.g., `keyframe_depth: unidepth-l`)
  - Geometry/optimization settings (e.g., `optimize_intrinsics` based on intrinsics source)
  - Other SLAM engine parameters (mapping, keyframes, optimization frequency)

- `configs/streams/raw_mp4_stream.yaml`  
  Video input configuration:
  - Stream backend and implementation (`instance` path used by `StreamList.make`)
  - Any arguments required to construct the video stream (e.g., file path, FPS, resolution)
  - Can be swapped for other backends (e.g., directory-based streams) via Hydra overrides.

Hydra overrides let you change behavior from the command line without editing files. For example:

```bash
vipe streams=raw_mp4_stream \
     pipeline=default \
     slam=default \
     output.path=custom_results/ \
     init.intrinsics=gt \
     slam.keyframe_depth=unidepth-l
```

Common patterns:

- Switch stream backend:
  - `streams=raw_mp4_stream`
- Change pipeline implementation:
  - `pipeline=some_other_pipeline`
  - or directly override the class: `pipeline.instance=vipe.pipeline.custom.CustomPipeline`
- Adjust detection/annotation behavior:
  - `pipeline.init.kf_gap_sec=1.0`
  - `pipeline.init.phrases='[person,car,bus]'`
- Control SLAM and depth:
  - `slam.keyframe_depth=unidepth-s`
  - `slam.optimize_intrinsics=false`
- Control outputs:
  - `output.path=/tmp/vipe_runs`
  - `output.save_slam_map=true`
  - `output.save_artifacts=true`

When embedding nv-tlabs__vipe in Python, you can use the same Hydra configs and `DictConfig` objects (as shown in the `@hydra.main` example above) and access/modify fields programmatically before passing them into your pipeline or stream constructors.

## Advanced Usage / Commands / API

Advanced usage builds on the basic `vipe` CLI by composing different Hydra configs and pipeline components. This section focuses on (1) pipeline/config selection, (2) CUDA-accelerated SLAM/geometry, and (3) depth priors such as Depth-Anything-3.

### CLI pipelines and config selection

At runtime, Hydra loads `configs/default.yaml` and its nested pipeline/stream definitions, then builds the pipeline via a factory such as `make_pipeline(args.pipeline)`.

Key concepts:

- **Top-level config**: `configs/default.yaml`  
  Controls global options and selects a sub-pipeline under `pipeline`.
- **Pipeline configs**: `configs/pipeline/*.yaml`  
  Define different processing graphs (e.g., default pose pipeline vs. SLAM-focused variants).
- **SLAM configs**: `configs/slam/*.yaml`  
  Configure SLAM-specific modules, backends, and geometry parameters.
- **Stream configs**: `configs/streams/*.yaml`  
  Configure how frames are read (e.g., raw MP4 vs. image directory).

You can switch pipelines in two ways:

1. **Via CLI `--pipeline` flag** (mapped to `args.pipeline`):

   ```bash
   vipe infer path/to/video.mp4 --pipeline default
   vipe infer path/to/video.mp4 --pipeline slam_dense
   ```

   These names must match the pipeline keys defined in `configs/pipeline/`.

2. **Via Hydra overrides** (when calling programmatically):

   ```python
   import hydra
   from omegaconf import DictConfig
   from vipe.pipeline import make_pipeline  # example import

   @hydra.main(version_base=None, config_path="configs", config_name="default")
   def run(cfg: DictConfig) -> None:
       pipeline = make_pipeline(cfg.pipeline)
       # choose stream backend based on cfg
       ...
       pipeline.run(video_stream)
   ```

   You can then override at launch time (e.g., `pipeline=slam_dense`, `streams=raw_mp4_stream`, etc.) using Hydra’s standard override syntax.

### Switching stream backends (MP4 vs. frame directory)

The CLI infers the stream backend from the arguments:

- **Raw MP4 video** (default):

  ```python
  video_stream = ProcessedVideoStream(
      RawMp4Stream(video), []
  ).cache(desc="Reading video stream")
  ```

- **Frame directory** (image sequence):

  ```python
  video_stream = ProcessedVideoStream(
      FrameDirStream(image_dir), []
  ).cache(desc="Reading image frames")
  ```

From the CLI:

- Use `video` positional arg for MP4 input.
- Use `--image-dir` to process a directory of frames instead of a video file.

You can define additional stream backends by creating new configs under `configs/streams/` and wiring them into your pipeline configs.

### CUDA-accelerated SLAM and geometry

nv-tlabs__vipe includes C++/CUDA extensions for SLAM and geometric operations. These are discovered and compiled via the extension spec:

```python
from vipe.ext.specs import get_sources, get_cpp_flags, get_cuda_flags
```

In configs under `configs/slam/` and `configs/pipeline/`, SLAM modules typically:

- Enable CUDA kernels when available (e.g., device set to `cuda`).
- Configure SLAM parameters (tracking window, keyframe selection, optimization settings).
- Optionally toggle CPU-only fallbacks for environments without GPUs.

To leverage CUDA:

- Select a SLAM-oriented pipeline (e.g., a pipeline config that imports `configs/slam/default.yaml` or similar).
- Ensure the SLAM module’s `device` or `use_cuda` setting is enabled in your config.
- If creating custom pipelines, reference the existing SLAM configs as templates and re-use their modules, only overriding high-level parameters.

For complex workflows, define your own `configs/pipeline/my_slam_experiment.yaml` that:

- Imports the stock SLAM config.
- Overrides only the parameters you care about (e.g., optimization iterations, visualization flags).

### Integrating Depth-Anything-3 and other depth priors

Depth-Anything-3 and similar depth/vision priors are integrated as optional components in the pipeline:

- The **optional dependency group** `dav3` in `pyproject.toml` pulls in the Depth-Anything-3 repo:

  ```toml
  [project.optional-dependencies]
  dav3 = [
      "depth-anything-3 @ git+https://github.com/heiwang1997/Depth-Anything-3.git@main"
  ]
  ```

- Pipeline configs can then enable a depth estimation module that wraps Depth-Anything-3 and feeds depth maps into downstream geometry/SLAM components.

To use Depth-Anything-3 in your own configs:

1. Install the `dav3` extra (see [Installation / Setup](#installation--setup) for the exact command).
2. Start from an existing depth-enabled pipeline under `configs/pipeline/` (if provided).
3. In your custom config (e.g., `configs/pipeline/depth_slam.yaml`):
   - Add/enable a depth node that uses the Depth-Anything-3 model.
   - Wire its outputs into SLAM or reconstruction modules (e.g., replacing a simple mono-depth prior).
   - Configure model resolution, device, and batching via Hydra parameters.

You can combine multiple priors (pose, depth, optical flow) by composing them in a single Hydra pipeline node graph, as long as each module’s inputs/outputs are consistent.

### Custom configs and power-user workflows

For more advanced setups:

- Create a new config under `configs/` (e.g., `configs/pipeline/my_custom_pipeline.yaml`).
- Use Hydra’s `defaults` and `override` mechanisms to:
  - Reuse shared components from `configs/pipeline/default.yaml`, `configs/slam/default.yaml`, and `configs/streams/*.yaml`.
  - Swap specific modules (e.g., replace the default depth prior with Depth-Anything-3).
  - Toggle visualization-only vs. full SLAM/geometry processing.

Since Hydra is the primary configuration mechanism, most advanced workflows involve composing or overriding YAML configs rather than modifying Python code. For examples of how existing pipelines are structured, inspect the YAML files under `configs/` and mirror their patterns in your own configs.

## Features

- Hydra-configurable video processing pipeline for SLAM and annotation, with modular components that can be swapped or extended via configuration.
- Command-line entry point (`vipe`) to run nv-tlabs__vipe end-to-end from the terminal, including optional visualization of intermediate results.
- CUDA-accelerated SLAM and geometry routines for efficient 3D reconstruction and camera tracking on modern GPUs.
- Integration with Depth-Anything-3 and other depth/vision priors to improve depth estimation and geometric understanding from video.
- Support for multiple video stream backends, including raw MP4 files and directories of image frames, enabling flexible input sources.

## Demo / Examples

Example configs and scripts are provided in the repository (for example under `configs/` and any `examples/` or `scripts/` directories). These cover typical end-to-end video processing and annotation pipelines, SLAM runs with CUDA-accelerated backends, and different video stream backends such as raw MP4 files and frame directories.

Refer to these example configs and entry points as a starting point, then adapt them by modifying the corresponding Hydra config files (e.g., `configs/pipeline/default.yaml`, `configs/slam/default.yaml`, or `configs/streams/raw_mp4_stream.yaml`) to match your own data and workflows.

## Contributing

Contributions to nv-tlabs__vipe are welcome. Before opening an issue or pull request, please read the contributing guidelines in this repository (see the CONTRIBUTING document in the project root).

You can help by:
- Reporting bugs and edge cases you encounter
- Proposing enhancements or new pipeline/stream configurations
- Submitting focused pull requests (e.g., CLI improvements, config updates, docs fixes)

When submitting a pull request, include a clear description, reference any related issues, and ensure existing tests or examples still run as described in the installation and quickstart sections.

## License

nv-tlabs__vipe is licensed under the MIT License. See the [LICENSE](LICENSE) file for the full license text and third-party notices.