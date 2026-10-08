# Magpie Experimental 整体项目审查计划

## 目标与产出

对 Magpie Experimental（Blinue/Magpie 的实验分支，C++20/WinUI 3，当前 0.6.7）做整体审查：**架构 + 代码质量 + 工程健康度**，覆盖 src 主要模块。

产出：
1. 审查报告：`docs/experimental/reviews/20260911-project-overview-REVIEW.md`（中文，沿用项目现有 review 文档风格；发现按 高/中/低 分级，每条带 file:line 证据与建议）
2. 聊天中输出 Top 发现摘要

约束：本机为 Linux，Windows 专属项目 → **纯静态审查**，不编译、不运行测试（报告中注明此限制）。审查过程只读，唯一写入是报告文件。

## 已完成的初步勘察（作为审查起点）

- 模块拓扑：`Magpie`(WinUI3 UI) / `Magpie.Core`(静态库·核心引擎) / `Effects`(HLSL) / `Shared` / `TouchHelper` / `Updater` / `RtxVideoBridge`(条件构建) / `_ConanDeps`
- 规模热点：Renderer.cpp 4164 行、ScalingWindow.cpp 2639 行、AppSettings.cpp 1637 行、DLSSFrameGenerator.cpp 994 行
- CI（build.yml）：MSVC/ClangCL × x64/ARM64 四矩阵**只构建、不跑任何测试**
- tests/ 为自研 harness：PS1 驱动 + Python 从源码提取/生成 cpp 测试 + cl.exe 现场编译运行
- 疑点：`certs/Magpie.pfx`（签名证书）通过 `certs/.gitignore` 的 `!*.pfx` 被纳入版本控制
- docs/experimental/ 有成熟的 review/todos/testing 文档文化与逐 beta 发布说明

## 执行步骤

### 第 1 步：架构梳理（读代码 + 画数据流）

- 依赖方向检查：从各 vcxproj 提取引用关系，确认 Core 不反向依赖 UI、Shared 无重依赖；RtxVideoBridge 的条件构建边界（`RtxVideoSdkDir` 探测）
- 核心数据流追踪：`ScalingService`(UI) → `ScalingRuntime`(专用线程 + 消息泵) → `ScalingWindow`(会话状态机) → `Renderer` → 4 种 `FrameSource`(DDA/DWM共享面/GDI/GraphicsCapture) → `EffectDrawer`(MagpieFX 链) + 原生后端(DLSS/FSR2/3/XeSS/RTXVideo, 经 `NativeEffectBackendFactory`) → Presenter(Adaptive/CompSwapchain/XeSSFG)
- 分支新增子系统分层评估：`FrameGuidanceService` + 光流提供方(Nvidia/Amd/Zero/HalfRes)、HDR 管线(Hdr* 约 15 文件 + GroupA/BHdrRoutes)、`ReflexController`、参数系统
- UI 层：MVVM/Service 组织、`AppSettings`/`ConfigPersistence`/`ConfigRecovery`/`ConfigLocations` 的配置迁移与崩溃恢复设计
- 线程模型：UI 线程 / 缩放线程 / D3D11-D3D12 互操作 fence 与 keyed mutex 的同步边界（重点读 `ScalingRuntime.cpp` 析构中的消息泵死锁规避模式）

### 第 2 步：代码质量抽样深读

深读文件（按核心度与规模）：
- `src/Magpie.Core/ScalingRuntime.cpp`（500 行，线程生命周期）
- `src/Magpie.Core/ScalingWindow.cpp`（2639 行，会话状态机）
- `src/Magpie.Core/Renderer.cpp`（4164 行，重点：fork 新增的 HDR/FG 逻辑是否过度挤入核心文件、可拆分性）
- `src/Magpie.Core/DLSSFrameGenerator.cpp` + `FrameGuidanceService.cpp`（D3D11/12 互操作与 fence 正确性）
- Upscaler 家族（FSR2/FSR2ZeroMV/FSR3/FSR3ZeroMV/XeSS/XeSSZeroMV/DLSSSR）——评估复制粘贴程度与可抽公共层
- `src/Magpie/AppSettings.cpp` + `ConfigPersistence.h`（配置版本迁移正确性）
- `src/Shared/`（Logger、SmallVector 等基础设施）

检查维度：HRESULT/错误处理纪律、com_ptr/wil 资源管理、同步原语正确性、注释（中英混杂度/准确性）、TODO/FIXME 密度、死代码与实验遗留、命名一致性、重复代码。

### 第 3 步：工程健康度

- 构建：`Directory.Build.props` 编译选项纪律（W4/SDL/no-RTTI/utf-8——初判良好）；条件 SDK 特性矩阵（EnableDLSSFrameGeneration/EnableAmdOpticalFlow/EnableRTXVideoDenoise）与本地 SDK 依赖的可复现性（Fetch-* 脚本 + conan-locks）
- CI：build.yml 无测试步骤的影响；release.yml 的签名与发布流程
- 测试：自研 harness 模式评估（提取式测试对源码路径的脆弱耦合、可维护性、能否进 CI）
- 文档与发布：docs 双语文档、逐 beta RELEASE_NOTES、version.json(0.6.7) 与 README/release 工作流的一致性
- 仓库卫生：`certs/Magpie.pfx` 被纳入版本控制的影响（上游遗留自签名测试证书 + CI secret 密码；对正式发布签名的含义）；第三方 DLL 不入库的边界执行情况
- 上游关系：与 Blinue/Magpie 的分歧规模与同步负担（定性）

### 第 4 步：撰写报告

- 写 `docs/experimental/reviews/20260911-project-overview-REVIEW.md`：
  - 概览与架构图（文字版数据流）
  - 发现清单：架构 / 代码质量 / 工程健康 三类，各条含严重级别、file:line 证据、建议
  - 明确标注「静态审查、未编译验证」的限制
- 聊天输出 Top 发现摘要

## 验证方式

- 报告中每条发现均回溯到具体 file:line（可抽查复核）
- 交叉核对 build.yml / release.yml / version.json / docs 描述与源码实际一致
- 无任何源码改动；除报告文件外无其他写入
