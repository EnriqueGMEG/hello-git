# 🧠 CLAM Inference Pipeline for WSIs – Dockerized

This Docker container executes a full inference pipeline using an ensemble of 5 cross-validated CLAM models for the classification of Whole Slide Images (WSIs), following the methodology described by Kalimuthu et al.

The pipeline performs the following steps automatically:

1. Patch Extraction: Generates image patches from input WSIs.
2. Feature Extraction: Extracts high-dimensional features from these patches using a Vision Transformer (UNI model).
3. Ensemble Inference: Performs slide-level classification using an ensemble of 5 pre-trained CLAM models. The final prediction is determined by majority voting across the individual models, each applying its own optimal Youden's Index threshold.

---

## 🚀 Running the Docker Container

To run the container, you must:

1. Create two folders on your local machine:
    - **Input folder**: Local directory with the WSIs (in `.svs`, `.tif`, `.tiff` format)
    - **Output folder**: Directory where all intermediate and final results will be saved

2. Provide the `--patch_level` parameter at runtime:
    - **Patch level** (`--patch_level`): The OpenSlide pyramid level used to extract patches (default is `1`, usually corresponds to 20x)

### 🗃️ Required Parameters

| Parameter     | Description                                                                            |
|---------------|----------------------------------------------------------------------------------------|
| `-v /local/input:/mnt/storage/input`  | Maps your local WSI folder to the container’s input folder    |
| `-v /local/output:/mnt/storage/output`| Maps your local results folder to the container’s output folder |
| `--patch_level` | Level of downsampling (OpenSlide level)                             |

> ⚠️ The container reads from `DATA_INPUT`, writes to `DATA_OUTPUT`, and expects both to point to folders under `/mnt/storage/`, as defined by the host system.

### ✅ `--patch_level` Parameter

This parameter defines the resolution level of the WSI to use when extracting patches. It corresponds to the pyramid levels generated by OpenSlide.

- `0`: Maximum resolution (typically 40x).
- `1`: 2x downsampled (typically 20x) → **default and recommended**.

> Use `--patch_level 1` unless you're certain your images require a different scale.

### Example Inference Command

```bash
docker run --rm \
  -e DATA_INPUT=/mnt/storage/input \
  -e DATA_OUTPUT=/mnt/storage/output \
  -v /local/input:/mnt/storage/input \
  -v /local/output:/mnt/storage/output \
  kalimuthu_infer_image \
  --patch_level 1
```

> 🖼️ `kalimuthu_infer_image` is the name of the Docker image.

> 🔁 The --patch_level parameter is optional. If not specified, it will default to 1.


### 🧑‍💻 Running Docker Without Root Permissions (Recommended)

By default, Docker containers run as the `root` user. This means that any files created inside the container may be owned by `root` on your local machine, potentially causing **permission issues** when trying to edit or delete them later.

To avoid this, you can run the container as your **local user** using the `--user` option:

```bash
--user $(id -u):$(id -g)
```

This ensures that all files generated inside the mounted output folder are owned by your user.

#### Example: Run with Correct User Permissions

```bash
docker run --rm \
  --user $(id -u):$(id -g) \
  -e DATA_INPUT=/mnt/storage/input \
  -e DATA_OUTPUT=/mnt/storage/output \
  -v /local/input:/mnt/storage/input \
  -v /local/output:/mnt/storage/output \
  kalimuthu_infer_image \
  --patch_level 1
```

> ⚠️ This avoids `Permission Denied` errors and ensures portability to environments with restricted users.

---

### 🔍 How Paths and Environment Variables Work

#### 📁 Working Directory: `/opt/app`

The container uses `/opt/app` as its working directory (instead of `/app`) to ensure compatibility with cloud and cluster environments that do not allow writing in the root-level `/app` folder. This avoids permission errors when running as a non-root user.

#### 🧱 Environment Variables

The container defines three key environment variables internally:

| Variable              | Default Value         | Description                                        |
| --------------------- | --------------------- | -------------------------------------------------- |
| `DATA_INPUT`          | `/mnt/storage/input`  | Directory where WSIs are read from                 |
| `DATA_OUTPUT`         | `/mnt/storage/output` | Directory where results are written                |
| `DATA_INPUT_BASE_DIR` | `/mnt/storage`        | Base directory used to define temporary subfolders |

📅 These are **already set in the Docker image** — you don’t need to pass them manually in most cases.

#### 🔄 Overriding Environment Variables

If your execution platform requires different paths (e.g. `/mnt/storage/output/CNIO`), you can override the defaults by passing `-e` flags at runtime:

```bash
docker run --rm \
  -v /local/input:/mnt/storage/input \
  -v /local/output:/mnt/storage/output/CNIO \
  -e DATA_OUTPUT=/mnt/storage/output/CNIO \
  kalimuthu_infer_image \
  --patch_level 1
```

This overrides `DATA_OUTPUT` and ensures the algorithm writes to a location that your infrastructure can later access and export.

---

## 📁 Output Folder Structure

After running, the output directory will contain:

```
output/
├── patches/
│   ├── patches/       # Extracted image patches
│   ├── masks/         # Tissue masks
│   └── stitches/      # Optional tissue visualizations
├── features/          # Extracted feature files (.pt)
    ├── h5_files/      # Contains the HDF5 files with patch metadata (location, size, etc.)
    └── pt_files/      # Extracted feature tensors (.pt) used for model inference
├── process_slides.csv # CSV with processed slide IDs
├── inference_summary_predictions.csv # Final slide-level predictions
└── inference_detailed_predictions.csv # Slide-level predictions of each model
```

### 📄 Description of Output Files

- **`inference_summary_predictions.csv`**  
  Contains the final classification results for each slide. It includes the following columns:

  | Column                     | Description                                      |
  |----------------------------|--------------------------------------------------|
  | `slide_id`                 | ID of the processed slide                        |
  | `predicted_class`          | Predicted class label (0 or 1) resulting of majority voting  |

- **`inference_detailed_predictions.csv`**  
  Contains the detailed classification results for each slide. It includes the following columns:

  | Column                     | Description                                      |
  |----------------------------|--------------------------------------------------|
  | `slide_id`                 | ID of the processed slide                        |
  | `Model_X_Predicted_Probability_Class_1` | Predicted probability of the slide belonging to class 1, as determined by a specific cross-validation model (Model X) |
  | `Model_X_Optimal_Threshold` | Optimal Youden's Index threshold used by a specific cross-validation model (Model X) to make its binary prediction |
  | `Model_X_Individual_Binary_Prediction` | Prediction (0 or 1) made by a specific cross-validation model (Model X) for the slide, based on its predicted probability and optimal threshold |
  
  
- **`process_slides.csv`**  
  Keeps track of which slides have already been processed, to avoid repeating patch extraction and feature computation.

  - ✅ If you add a new image, it will be appended to this file and processed as expected.
  - ⚠️ If you want to **re-run the full pipeline** (patch extraction + feature extraction + inference) on a slide that’s already in the file, you **must manually delete that row** from `process_slides.csv`.

---

## ⚙️ GPU vs CPU Execution

This container runs with **GPU if available** and **falls back to CPU automatically** if not.

### 🧠 Note on Performance

- **GPU recommended** for fast feature extraction with Vision Transformer
- **CPU works**, but feature extraction will be significantly slower

### 🛠️ Requirements for GPU Execution

To run with GPU, your local machine must have:

- A compatible **NVIDIA GPU**
- **NVIDIA drivers** installed
- **Docker Engine** installed
- **NVIDIA Container Toolkit** installed

### Installing NVIDIA Container Toolkit

**Follow the official installation guide** from NVIDIA to install and configure the container toolkit:

See: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html

This includes:

- Adding the official NVIDIA repository
- Installing `nvidia-container-toolkit`
- Configuring Docker to use the `nvidia` runtime

> ⚠️ Without this, the container will **not be able to access the GPU**, even if one is present.

### Verify GPU Access from Docker

```bash
docker run --rm --gpus all nvidia/cuda:11.8.0-base nvidia-smi
```

If this command returns information about your GPU (model, memory, driver version, etc.), the system is correctly set up to use the GPU

> The inference script will automatically detect and use the GPU if available

---

## 🧪 Minimal Example

### 1. Create local folders:
```bash
mkdir input output
```

### 2. Copy your WSI slides (e.g. `.svs`) into `input/`.
```bash
cp your_slide.svs input/
```

### 3. Run the container:
```bash
docker run --rm \
  --user $(id -u):$(id -g) \
  -v local/input:/mnt/storage/input \
  -v local/output:/mnt/storage/output \
  kalimuthu_infer_image \
  --patch_level 1
```

### 4. View results in `output/inference_summary_predictions.csv`:
```bash
cat output/inference_summary_predictions.csv
```

---

## 📬 Questions?

If you encounter any problems, please check:

- That your input slides are supported (`.svs`, `.tif`, etc.)
- That Docker has the right permissions to access input/output folders
- That your system has enough RAM/VRAM for patching and feature extraction

If you have questions or encounter issues not covered here, please contact us:

📧 **Support contacts**:  
- promeroj@cnio.es  
- elopezl@cnio.es
- ssabroso@cnio.es

---

© 2025 – Kalimuthu WSI Dockerized Inference

