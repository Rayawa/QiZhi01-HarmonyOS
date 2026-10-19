# QiZhi01-HarmonyOS

## 项目概述

本项目是一个运行在 **OpenHarmony 5.0.3** 上的 **X光胸片五类疾病检测图形化推理界面**，基于 DevEco Studio 开发，面向 **启智01开发者套件（昇腾310B）**。

项目集成了两套推理引擎：
- **CANN ACL 引擎**：通过 C++ NAPI 桥接，调用昇腾310B NPU（DaVinci V300）执行 YOLOv5s 硬件加速推理
- **MindSpore Lite 引擎**：通过 ArkTS 侧 `@kit.MindSporeLiteKit`，在 CPU（TAISHAN V200M）上执行推理，作为 ACL 引擎的补充/fallback 方案

检测目标为五类 X 光胸片疾病：细菌性肺炎、病毒性肺炎、病变、健康、结核病。

## 功能特性

- **双引擎推理架构**：NPU（ACL）与 CPU（MindSpore Lite）双路径，确保推理可用性
- **平板三栏布局**：35%-30%-35% 响应式布局，适配大屏设备
- **Canvas 检测框绘制**：多类别颜色区分，实时叠加绘制检测框和置信度标签
- **PhotoViewPicker 图片选择**：支持相册选择 X 光图片，Decoder 直接缩放至 640×640
- **类别统计面板**：实时显示各类别检测数量
- **检测记录网格**：Grid 组件展示历史检测记录

## 项目结构

```
QiZhi01-HarmonyOS/
├── build-profile.json5              # 顶层构建配置（签名、SDK版本、编译模式）
├── oh-package.json5                 # 顶层包配置
├── hvigorfile.ts                    # Hvigor 构建入口
├── code-linter.json5                # 代码检查规则
├── .clangd / .clang-tidy            # C++ 语言服务器与静态检查配置
│
├── AppScope/                        # 应用全局配置
│   └── app.json5                    # 全局清单：bundleName(top.rayawa.qizhi01)、版本、图标
│
├── hvigor/                          # Hvigor 构建系统配置
│   └── hvigor-config.json5
│
├── entry/                           # 主模块（entry）
│   ├── build-profile.json5          # 模块构建配置（ArkTS混淆等）
│   ├── hvigorfile.ts                # 模块级构建入口
│   ├── oh-package.json5             # 模块包配置
│   └── src/main/
│       ├── module.json5             # 模块清单：Ability声明、权限、路由
│       ├── syscap.json              # 系统能力声明：SystemCapability.AI.MindSporeLite
│       │
│       ├── ets/                     # ArkTS 源码
│       │   ├── entryability/
│       │   │   └── EntryAbility.ets # 应用入口：加载模型文件、初始化UI
│       │   ├── model/
│       │   │   ├── DetectionType.ets    # 检测类型定义与常量（5类标签+颜色）
│       │   │   ├── ModelBufferHolder.ets# 模型缓冲区持有者（绕过AppStorage序列化）
│       │   │   └── YoloModel.ets        # MindSpore Lite推理引擎封装
│       │   └── pages/
│       │       └── Index.ets            # 主页面：三栏布局、图片选择、Canvas绘图
│       │
│       ├── cpp/                     # C++ Native 源码
│       │   ├── CMakeLists.txt       # CMake编译配置（C++17, libentry.so）
│       │   ├── hello.cpp            # NAPI桥接层（initYoloEngine / runYoloInference）
│       │   ├── yolo_infer.h         # ACL推理引擎头文件（AclDl单例、YoloInference类）
│       │   ├── yolo_infer.cpp       # ACL推理引擎实现（dlopen动态加载、推理+后处理+NMS）
│       │   └── types/
│       │       └── libentry/
│       │           ├── index.d.ts   # Native模块类型声明
│       │           └── oh-package.json5
│       │
│       └── resources/              # 资源文件
│           ├── base/
│           │   ├── element/         # 颜色、字体、字符串资源
│           │   ├── media/           # 应用图标
│           │   └── profile/
│           │       └── main_pages.json  # 页面路由配置
│           ├── dark/element/        # 深色模式资源
│           └── rawfile/
│               └── best_kirin.ms    # MindSpore Lite 模型文件
│
└── .deveco/plans/                   # DevEco Studio 开发计划（项目迭代记录）
    ├── 1781611682601-hidden-mountain.md   # 初始计划：完整推理流程
    ├── 1781614817754-quick-harbor.md      # 中期优化：前端重写 + 平板布局
    ├── 1782307811819-jolly-cactus.md      # 最终完善：模型部署 + MindSpore Lite集成
    ├── 1782649956651-glowing-star.md      # 修复：图像归一化 + letterbox
    └── 1782726043523-glowing-wizard.md    # 修复：MS Lite internal slot错误
```

## 架构设计

```
                      +---------------------+
                      |   Index.ets (UI)    |
                      |  三栏布局 / Canvas   |
                      +----------+----------+
                                 |
              +------------------+------------------+
              |                                      |
    +---------v----------+              +-----------v-----------+
    |  YoloModel.ets      |              |   hello.cpp (NAPI)    |
    |  (MindSpore Lite)   |              |   (C++ 桥接层)        |
    |  CPU 推理            |              |   initYoloEngine      |
    |  preprocessRGBA      |              |   runYoloInference    |
    |  predict + 后处理     |              +-----------+-----------+
    +---------+-----------+                          |
              |                         +------------v-----------+
    +---------v-----------+             |  yolo_infer.cpp (ACL)  |
    |  best_kirin.ms       |             |  (CANN ACL 推理引擎)   |
    |  (MindSpore Lite模型) |             |  dlopen libacl_runtime |
    +---------------------+             |  aclmdlExecute         |
                                         |  PostProcess + NMS     |
                                         +------------------------+
                                                   |
                                         +---------v----------+
                                         |   昇腾310B NPU      |
                                         |   (DaVinci V300)    |
                                         +--------------------+
```

### 推理流程

1. **ArkTS 端**：用户选择图片 → `DecodingOptions.desiredSize={640,640}` 缩放 → `readPixelsToBuffer` 提取 RGBA
2. **C++ NAPI 端**：`hello.cpp` 接收 ArrayBuffer → 异步任务中调用 `YoloInference::DoInference` → RGBA→RGB 归一化 → NPU 推理 → 后处理 NMS
3. **结果返回**：检测框数组返回 ArkTS → `Canvas` 绘制彩色检测框和标签

### 关键常量

| 参数 | 值 | 说明 |
|------|-----|------|
| 模型输入尺寸 | 640×640 | YOLOv5s 标准输入 |
| 类别数 | 5 | 细菌性肺炎/病毒性肺炎/病变/健康/结核病 |
| 候选框数 | 25200 | 3 尺度 × 3 anchors |
| 每行输出值 | 10 | cx,cy,w,h,obj_conf + 5 个类别置信度 |
| 置信度阈值 | 0.25 | 过滤低置信度检测框 |
| NMS IoU 阈值 | 0.45 | 去除重叠框 |

## 构建与部署

### 环境要求

- DevEco Studio（支持 API 12 / OpenHarmony 5.0.3）
- 设备端已安装 CANN 8.2.RC1 推理环境（ACL 引擎需要）
- 设备端已配置 `/etc/profile.d/cann_env.sh` 环境变量

### 构建

在 DevEco Studio 中打开项目，执行：
```
Build → Build Hap(s) → entry-default-signed.hap
```

### 部署

```bash
# 通过 HDC 部署
hdc install entry-default-signed.hap

# 或通过 U 盘手动安装
bm install -p /mnt/sdcard/entry-default-signed.hap
```

### 模型文件说明

| 文件 | 格式 | 引擎 | 部署方式 |
|------|------|------|----------|
| `rawfile/best_kirin.ms` | MindSpore Lite | CPU 推理 | `EntryAbility` 自动从 rawfile 读取 |
| `best.om` | Cann ACL (昇腾离线模型) | NPU 推理 | 需手动推送至设备 `hdc file send best.om /data/local/tmp/` |

## 权限

| 权限 | 用途 |
|------|------|
| `ohos.permission.READ_MEDIA` | 读取相册中的 X 光图片 |
| `ohos.permission.WRITE_MEDIA` | 写入相册（预留） |
| `SystemCapability.AI.MindSporeLite` | MindSpore Lite 推理框架系统能力 |

## 开发迭代历史

| 阶段 | 计划文件 | 主要内容 |
|------|----------|----------|
| 初始实现 | hidden-mountain | YOLOv5s 完整推理流程、OpenCV预处理、NAPI集成 |
| 中期优化 | quick-harbor | 前端重写、平板分栏布局、模型文件名更新 |
| 最终完善 | jolly-cactus | MindSpore Lite 集成、三栏布局、类别统计面板 |
| Bug 修复 | glowing-star | 图像归一化（ImageNet mean/std）、letterbox预处理 |
| Bug 修复 | glowing-wizard | 修复 MS Lite internal slot 错误、ModelBufferHolder |

## 相关文档

- 项目配套详细技术文档位于 `../` 目录下的 `.docx` 系列文件
- CANN 推理引擎安装指南参考 `../QiZhi01-OpenHarmony/device_board_ascend/tests/cann/`
