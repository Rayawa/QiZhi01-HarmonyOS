# QiZhi01-HarmonyOS

English | [简体中文](README_zh.md)

An OpenHarmony chest-X-ray inference demo with Ascend ACL and MindSpore Lite execution paths.

## Overview

The interface loads an X-ray image, runs a five-class detection model and draws boxes, scores and class counts. A native N-API bridge provides the Ascend ACL path; a MindSpore Lite path provides CPU inference. This is an experimental model demonstration; labels are model outputs, not a validated diagnostic result.

## Technology and Dependencies

ArkTS/ArkUI, C++ N-API, OpenHarmony 5.0.3 (API 15), Hvigor and OHPM. Hardware execution requires the QiZhi01/Ascend 310B environment, a matching CANN/ACL runtime and compatible model files; the CPU path requires MindSpore Lite.

## Repository Structure

- `entry/src/main/ets/` — ArkTS application code
- `entry/src/main/resources/` — resources and routes
- `AppScope/` — application metadata
- `build-profile.example.json5` — build configuration template
- `oh-package.json5` — package dependencies
- `entry/src/main/cpp/` — native ACL bridge
- `entry/src/main/resources/rawfile/` — model resources

## Quick Start

1. Open the repository root in DevEco Studio with the SDK required by `build-profile.example.json5`.
2. Copy that template to `build-profile.json5`, restore OHPM dependencies and configure signing locally if installing on a device.
3. Select the `entry` module and a compatible device/emulator, then build and run from the IDE. Keep certificates and machine-specific paths outside version control.

Build for the prepared QiZhi01 system, configure the native include/library paths for the installed Ascend runtime, and provide the model resources expected by the code. Choose an image and inspect engine availability before running inference. Board integration is maintained in [QiZhi01-OpenHarmony](https://github.com/Rayawa/QiZhi01-OpenHarmony).
