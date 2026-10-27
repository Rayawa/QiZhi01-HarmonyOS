# QiZhi01-HarmonyOS

[English](README.md) | 简体中文

结合 Ascend ACL 与 MindSpore Lite 路径的 OpenHarmony 胸片推理演示。

## 项目概述

界面加载胸片图片，运行五分类检测模型，绘制检测框、分数与类别数量。Ascend ACL 路径通过原生 N-API 桥接，MindSpore Lite 路径使用 CPU 推理。项目为实验性模型演示，输出标签不代表经过验证的诊断结果。

## 技术栈与依赖

ArkTS/ArkUI、C++ N-API、OpenHarmony 5.0.3（API 15）、Hvigor、OHPM。硬件推理需要启智01/昇腾310B、匹配的 CANN/ACL 运行时和模型文件；CPU 路径需要 MindSpore Lite。

## 文件结构

- `entry/src/main/ets/` — ArkTS 应用代码
- `entry/src/main/resources/` — 资源与路由
- `AppScope/` — 应用配置
- `build-profile.example.json5` — 构建配置模板
- `oh-package.json5` — 包依赖
- `entry/src/main/cpp/` — 原生 ACL 桥接
- `entry/src/main/resources/rawfile/` — 模型资源

## 快速上手

1. 使用 DevEco Studio 打开仓库根目录，并安装 `build-profile.example.json5` 指定的 SDK。
2. 将模板复制为 `build-profile.json5`，恢复 OHPM 依赖；如需真机安装，在本地配置签名。
3. 选择 `entry` 模块和兼容设备或模拟器，在 IDE 中构建运行。证书及机器路径不进入版本控制。

面向已配置的启智01系统构建，按已安装 Ascend 运行时配置原生头文件和库路径，并准备源码要求的模型资源。选图后先检查引擎状态，再运行推理。板级适配见 [QiZhi01-OpenHarmony](https://github.com/Rayawa/QiZhi01-OpenHarmony)。
