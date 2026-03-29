<div align="center">
    <h1>BommieToolkit</h1>
    <a href="https://github.com/BommieToolkit/BommieToolkit"><img src="https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black" /></a>
    <a href="https://github.com/BommieToolkit/BommieToolkit"><img src="https://img.shields.io/badge/macOS%20ARM-000000?logo=apple&logoColor=white" /></a>
    <br />
</div>

<p align="center">
    <a href="https://scholar.google.com/citations?user=SDtnGogAAAAJ&hl=en"><strong>Alejandro Fontan</strong></a>
    ·
    <a href="https://scholar.google.com/citations?user=MNrMUPMAAAAJ&hl=en"><strong>Emilio Olivastri</strong></a>
</p>

## End-to-End Pipeline: Videos → Gaussian Splatting

The full pipeline (from videos to a nerfstudio-ready `transforms.json`) can be run with a single command:

```bash
pixi run -e colmap reconstruct
```

This chains the following steps automatically (using pixi task dependencies):

1. Extract frames from left & right videos
2. Synchronise image pairs by timestamp
3. COLMAP feature extraction
4. COLMAP rig configuration
5. COLMAP sequential matching
6. COLMAP mapping
7. COLMAP model conversion (TXT + PLY)
8. `colmap2nerf` → `monkey_output/transforms.json`

Default video paths are `videos/monkey_left.MP4` and `videos/monkey_right.MP4`.
Override them with environment variables:

```bash
VIDEO_LEFT=path/to/left.MP4 VIDEO_RIGHT=path/to/right.MP4 RESOLUTION=high OUTPUT_DIR=my_output pixi run -e colmap reconstruct
```

| Variable | Default | Description |
|---|---|---|
| `VIDEO_LEFT` | `videos/monkey_left.MP4` | Path to the left camera video |
| `VIDEO_RIGHT` | `videos/monkey_right.MP4` | Path to the right camera video |
| `RESOLUTION` | `medium` | Frame extraction resolution (`lowest`, `low`, `medium`, `high`) |
| `OUTPUT_DIR` | `monkey_output` | Root output directory for all pipeline artefacts |

After the pipeline finishes, run nerfstudio separately (see [GS Reconstruction](#gs-reconstruction-with-nerfstudio) below).

---

## Video Recording

## Rig Calibration
Build Kalibr
```bash
pixi run -e kalibr build
```

Extract images from calibration videos
```bash
# Target pixel counts used for scaling based on the '--resolution' preset
#     Presets: extra-low (~320x240), low (~640x480), medium (~1920x1080), high (~3840x2160)
# 'skip' specifies how many seconds to skip from the beginning of the video
pixi run extract_images --video videos/cal_left.MP4 --output calibration_output/cam0 --gray --resolution medium (--skip 2.0)
pixi run extract_images --video videos/cal_right.MP4 --output calibration_output/cam1 --gray --resolution medium (--skip 2.0)
```
(OPTIONAL!! But makes life easier)
Synch image pairs using timestamps
```bash
pixi run match_images_by_ns \
  --images_folder_left calibration_output/cam0 \
  --images_folder_right calibration_output/cam1 \
  --colmap_folder_left calibration_output/syncd/cam0 \
  --colmap_folder_right calibration_output/syncd/cam1 \
  --threshold-ns 5000000 \
  --sample_step 5
```

Run calibration

```bash
pixi run -e kalibr kalibr-calibrate-stereo-rig \
  images_folder_left=calibration_output/cam0 \
  images_folder_right=calibration_output/cam1 \
  output_folder=calibration_output \
  target=files/april_10x6.yaml \
  freq=30
```
Added optional flags: verbose, and create_bag.
- verbose: The default value is 0, deactivating the visualization which makes the calibration fail when there is no screen avaialable.
- create_bag: The default value is 1, which makes you create the rosbag every time you run the calibration script. If happy with the first bag, just setting create_bag=0, will avoid this extra step.

Added the following flag to handle cases in which the calibration fails due to bad initialization. The scritp is going to wait for an input by the user to initialize the focal length. Typical good value is the half of the height of the image.

```bash
export KALIBR_MANUAL_FOCAL_LENGTH_INIT=1
```

Calibration with extra flags

```bash
pixi run -e kalibr kalibr-calibrate-stereo-rig \
  images_folder_left=calibration_output/cam0 \
  images_folder_right=calibration_output/cam1 \
  output_folder=calibration_output \
  verbose=1 create_bag=0 \
  target=files/april_10x6.yaml \
  freq=30
```

Generate .json file with rig configuration
```bash
pixi run get_rig_config_json calibration_output/calibration-camchain.yaml calibration_output/rig_config.json
```

## COLMAP Reconstruction

The individual COLMAP steps are available as pixi tasks in the `colmap` environment.
Run `pixi run -e colmap reconstruct` for the full automated pipeline (see above), or execute individual steps:

Extract images from videos

```bash
pixi run -e colmap extract-left
pixi run -e colmap extract-right
# Override defaults with env vars: VIDEO_LEFT=... VIDEO_RIGHT=... RESOLUTION=...
```

Synchronise image pairs using timestamps

```bash
pixi run -e colmap sync-images
```

(OPTIONAL) Get masks for the images using sam3, you have to have a hugging face account and login to download the weights
```bash
pixi run -e sam hf auth login
pixi run -e sam create_masks
```
To access the app click navigate on the following link:
```bash
http://0.0.0.0:7997
```

Execute COLMAP steps individually

```bash
pixi run -e colmap feature-extractor
pixi run -e colmap rig-configurator
pixi run -e colmap sequential-matcher
pixi run -e colmap mapper
pixi run -e colmap model-converter-txt
pixi run -e colmap model-converter-ply
```

Visualize reconstruction
```bash
pixi run -e colmap colmap gui \
  --database_path monkey_output/database.db  \
  --image_path monkey_output/colmap_images \
  --import_path monkey_output/sparse/0
```

Convert COLMAP output for nerfstudio

```bash
pixi run -e colmap to-nerf
```

## GS Reconstruction with nerfstudio

```bash
git clone https://github.com/nerfstudio-project/nerfstudio.git
cd nerfstudio
pixi run post-install
pixi shell
```

```bash
ns-train splatfacto --data /home/alejandro/BommieToolkit/monkey_output
```

```bash
ns-viewer --load-config outputs/monkey_output/splatfacto/2025-11-19_105617/config.yml
```

"ply_file_path" : "/home/alejandro/BommieToolkit/monkey_output/sparse/0/mesh.ply",

## BommieToolkit Roadmap

- [ ] Make Kalibr a Conda package
- [x] Implement one end-to-end command, from videos to GS.
- [ ] Documentation for the intermediate outputs
- [ ] Documentation on recording calibration/reconstruction data
- [ ] Documentation on gopro settings
- [ ] How to build an underwater calibration pattern
- [ ] Refraction Removal
