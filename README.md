# QiZhi01-HarmonyOS

English | [简体中文](README_zh.md)


## Project overview

A **graphical inference interface for detecting five classes of chest X-ray conditions** on **OpenHarmony 5.0.3**, developed with DevEco Studio for the **QiZhi 01 Developer Kit (Ascend 310B)**.

The project integrates two inference engines:
- **CANN ACL engine**: Uses a C++ NAPI bridge to run hardware-accelerated YOLOv5s inference on the Ascend 310B NPU (DaVinci V300).
- **MindSpore Lite engine**: Runs inference on the CPU (TAISHAN V200M) through ArkTS `@kit.MindSporeLiteKit`, supplementing the ACL engine as a fallback.

The five chest X-ray detection classes are bacterial pneumonia, viral pneumonia, lesions, healthy, and tuberculosis.

## Features

- **Dual inference engines**: NPU (ACL) and CPU (MindSpore Lite) paths maintain inference availability.
- **Three-column tablet layout**: A responsive 35%-30%-35% layout for large displays.
- **Canvas detection boxes**: Class-specific colors, with detection boxes and confidence labels overlaid in real time.
- **PhotoViewPicker image selection**: Select X-rays from the gallery; the decoder resizes them directly to 640×640.
- **Class statistics panel**: Displays detection counts for each class in real time.
- **Detection history grid**: Uses the Grid component to display past detections.

## Project structure

```
QiZhi01-HarmonyOS/
├── build-profile.json5              # Top-level build configuration (signing, SDK version, build mode)
├── oh-package.json5                 # Top-level package configuration
├── hvigorfile.ts                    # Hvigor build entry point
├── code-linter.json5                # Lint rules
├── .clangd / .clang-tidy            # C++ language server and static analysis configuration
│
├── AppScope/                        # Global app configuration
│   └── app.json5                    # Global manifest: bundleName(top.rayawa.qizhi01), version, icon
│
├── hvigor/                          # Hvigor build system configuration
│   └── hvigor-config.json5
│
├── entry/                           # Main module (entry)
│   ├── build-profile.json5          # Module build configuration (ArkTS obfuscation, etc.)
│   ├── hvigorfile.ts                # Module build entry point
│   ├── oh-package.json5             # Module package configuration
│   └── src/main/
│       ├── module.json5             # Module manifest: Ability declarations, permissions, routes
│       ├── syscap.json              # System capability: SystemCapability.AI.MindSporeLite
│       │
│       ├── ets/                     # ArkTS source
│       │   ├── entryability/
│       │   │   └── EntryAbility.ets # App entry: load model files and initialize UI
│       │   ├── model/
│       │   │   ├── DetectionType.ets    # Detection types and constants (5 class labels + colors)
│       │   │   ├── ModelBufferHolder.ets# Model buffer holder (bypasses AppStorage serialization)
│       │   │   └── YoloModel.ets        # MindSpore Lite inference wrapper
│       │   └── pages/
│       │       └── Index.ets            # Main page: three columns, image selection, Canvas drawing
│       │
│       ├── cpp/                     # C++ native source
│       │   ├── CMakeLists.txt       # CMake build configuration (C++17, libentry.so)
│       │   ├── hello.cpp            # NAPI bridge (initYoloEngine / runYoloInference)
│       │   ├── yolo_infer.h         # ACL engine header (AclDl singleton, YoloInference class)
│       │   ├── yolo_infer.cpp       # ACL engine implementation (dlopen loading, inference, postprocessing, NMS)
│       │   └── types/
│       │       └── libentry/
│       │           ├── index.d.ts   # Native module type declarations
│       │           └── oh-package.json5
│       │
│       └── resources/              # Resource files
│           ├── base/
│           │   ├── element/         # Color, font, and string resources
│           │   ├── media/           # App icons
│           │   └── profile/
│           │       └── main_pages.json  # Page route configuration
│           ├── dark/element/        # Dark mode resources
│           └── rawfile/
│               └── best_kirin.ms    # MindSpore Lite model
│
└── .deveco/plans/                   # DevEco Studio development plans (iteration records)
    ├── 1781611682601-hidden-mountain.md   # Initial plan: complete inference flow
    ├── 1781614817754-quick-harbor.md      # Intermediate optimization: frontend rewrite + tablet layout
    ├── 1782307811819-jolly-cactus.md      # Final improvements: model deployment + MindSpore Lite integration
    ├── 1782649956651-glowing-star.md      # Fix: image normalization + letterbox
    └── 1782726043523-glowing-wizard.md    # Fix: MS Lite internal slot error
```

## Architecture

```
                      +---------------------+
                      |   Index.ets (UI)    |
                      | Three columns / Canvas |
                      +----------+----------+
                                 |
              +------------------+------------------+
              |                                      |
    +---------v----------+              +-----------v-----------+
    |  YoloModel.ets      |              |   hello.cpp (NAPI)    |
    |  (MindSpore Lite)   |              |   (C++ bridge)         |
    |  CPU inference     |              |   initYoloEngine       |
    |  preprocessRGBA      |              |   runYoloInference    |
    | predict + postprocess |           +-----------+-----------+
    +---------+-----------+                          |
              |                         +------------v-----------+
    +---------v-----------+             |  yolo_infer.cpp (ACL)  |
    |  best_kirin.ms       |             | (CANN ACL engine)     |
    | (MindSpore Lite model) |           | dlopen libacl_runtime |
    +---------------------+             |  aclmdlExecute         |
                                         |  PostProcess + NMS     |
                                         +------------------------+
                                                   |
                                         +---------v----------+
                                         | Ascend 310B NPU     |
                                         |   (DaVinci V300)    |
                                         +--------------------+
```

### Inference flow

1. **ArkTS**: Select an image → resize with `DecodingOptions.desiredSize={640,640}` → extract RGBA using `readPixelsToBuffer`.
2. **C++ NAPI**: `hello.cpp` receives the ArrayBuffer → an asynchronous task calls `YoloInference::DoInference` → RGBA-to-RGB normalization → NPU inference → NMS postprocessing.
3. **Results**: Return an array of detection boxes to ArkTS → draw colored boxes and labels with `Canvas`.

### Key constants

| Parameter | Value | Description |
|------|-----|------|
| Model input size | 640×640 | Standard YOLOv5s input |
| Classes | 5 | Bacterial pneumonia / viral pneumonia / lesions / healthy / tuberculosis |
| Candidate boxes | 25200 | 3 scales × 3 anchors |
| Output values per row | 10 | cx, cy, w, h, obj_conf + 5 class confidence scores |
| Confidence threshold | 0.25 | Filters low-confidence detection boxes |
| NMS IoU threshold | 0.45 | Removes overlapping boxes |

## Build and deployment

### Requirements

- DevEco Studio with API 12 / OpenHarmony 5.0.3 support
- CANN 8.2.RC1 inference environment installed on the device (required by the ACL engine)
- Environment variables configured in `/etc/profile.d/cann_env.sh` on the device

### Build

Open the project in DevEco Studio and run:
```
Build → Build Hap(s) → entry-default-signed.hap
```

### Deploy

```bash
# Deploy through HDC
hdc install entry-default-signed.hap

# Or install manually from a USB drive
bm install -p /mnt/sdcard/entry-default-signed.hap
```

### Model files

| File | Format | Engine | Deployment |
|------|------|------|----------|
| `rawfile/best_kirin.ms` | MindSpore Lite | CPU inference | `EntryAbility` reads it automatically from rawfile |
| `best.om` | CANN ACL (Ascend offline model) | NPU inference | Manually push it to the device using `hdc file send best.om /data/local/tmp/` |

## Permissions

| Permission | Purpose |
|------|------|
| `ohos.permission.READ_MEDIA` | Read chest X-rays from the gallery |
| `ohos.permission.WRITE_MEDIA` | Write to the gallery (reserved) |
| `SystemCapability.AI.MindSporeLite` | System capability for the MindSpore Lite inference framework |

## Development history

| Stage | Plan file | Main changes |
|------|----------|----------|
| Initial implementation | hidden-mountain | Complete YOLOv5s inference flow, OpenCV preprocessing, and NAPI integration |
| Intermediate optimization | quick-harbor | Frontend rewrite, tablet column layout, and updated model filenames |
| Final improvements | jolly-cactus | MindSpore Lite integration, three-column layout, and class statistics panel |
| Bug fixes | glowing-star | Image normalization (ImageNet mean/std) and letterbox preprocessing |
| Bug fixes | glowing-wizard | Fixed the MS Lite internal slot error; ModelBufferHolder |

## Related documentation

- Detailed technical documents accompanying the project are the `.docx` files in `../`.
- For CANN inference engine installation, see `../QiZhi01-OpenHarmony/device_board_ascend/tests/cann/`.
