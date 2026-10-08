# Magpie Experimental 项目审查计划

## 假设（用户未明确的部分）
- **范围**：全项目总体审查（用户范围问题超时未答）。策略 = 核心渲染模块深挖 + 全局风险模式横扫 + UI 快速过一遍。
- **侧重**：均衡覆盖（正确性 / 架构 / 性能），按严重度排序（用户已确认）。
- **环境约束**：本机为 Linux，无法编译 MSVC/WinRT 项目 → 纯静态审查，不做构建验证。

## 项目概况（已探明）
Magpie Experimental：Blinue/Magpie 非官方分支，C++/WinRT + D3D11/12 窗口缩放工具。
- `src/Magpie.Core` 179 文件 — 渲染核心（帧源、缩放器、帧生成、帧同步），审查重心
- `src/Magpie` 168 文件 — WinUI 3 UI
- `src/Effects` 183 文件 — HLSL 特效
- `src/Shared`、`src/Updater`、`src/TouchHelper`、`src/RtxVideoBridge`
- ~3000 提交，年度 381 次，非常活跃；tests/ 以 PowerShell/Python 脚本化测试为主

## 审查步骤

### 第 1 步：架构梳理（只读）
- 读 `Renderer.h/.cpp`、`FrameSourceBase`、各 `*Presenter`、`DeviceResources`，梳理渲染管线与线程模型
- 产出报告中的「架构概览」一节（文字图：捕获 → 处理 → 呈现，标注线程边界）

### 第 2 步：核心模块深挖（正确性优先）
按风险从高到低：
1. **Renderer 并发模型** — 后端线程、`MAX_SHARED_TEXTURE_SLOTS` 原子阵列/互斥锁（Renderer.h:308-401）：竞态、关停顺序、纹理槽生命周期
2. **ReflexController / 帧同步状态机**（ReflexController.h:197-203）：锁范围、原子序（memory order）、状态跃迁遗漏
3. **DLSS/FSR/XeSS 帧生成与缩放器**（DLSSFrameGenerator、FSR3Upscaler、XeSS*）：SDK 互操作、HRESULT 检查、资源泄漏、设备丢失恢复
4. **帧源**（DesktopDuplication、DwmSharedSurface、GDI）：超时/错误处理、DXGI 重置路径
5. **EffectCompiler/EffectDrawer**：MagpieFX 自定义格式解析的健壮性

### 第 3 步：全局风险模式横扫（grep 模式清单）
- 裸 `new/delete`、`memcpy/strcpy`、`reinterpret_cast`、未检查 HRESULT
- 整数截断（`static_cast<int>` 于尺寸类值）、有符号/无符号混用
- WinRT 事件退订遗漏（UI 泄漏）、回调/DllMain 中抛异常
- 逐个命中点人工复核，过滤误报（如 SmallVector 为 LLVM 移植代码，低优先）

### 第 4 步：UI 层快速过
- ViewModel 生命周期、dispatcher 线程安全、`XamlHelper`/`AppXReader`（已见手写内存管理）
- 最近活跃提交涉及的 effect picker / profiles 参数面板逻辑

### 第 5 步：测试覆盖评估
- 盘点 tests/ 脚本化测试实际覆盖面，指出与核心风险区的缺口

## 产出物
写入 **`CODE_REVIEW.md`**（仓库根目录，中文），结构：
1. 架构概览
2. 发现清单 — 按严重度分级（Critical / High / Medium / Low），每条含 `文件:行号`、证据代码、影响、修复建议
3. 测试覆盖缺口
4. 亮点与做得好的地方
5. 附录：假设与审查范围声明（注明纯静态、Linux 环境限制）

## 验证方式
- 每条发现回读原文件二次确认行号与上下文，报告中仅收录经复核的条目
- 严重度按「可触发路径 × 影响面」评定，报告中说明定级理由
- 无法运行构建/测试（Windows 项目 + Linux 环境），在报告附录明示此限制
