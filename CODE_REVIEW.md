# Magpie Experimental 项目审查报告

> 审查日期：2026-06（静态审查）
> 审查人：AI 代码审查（pi 会话）
> 范围：全项目（渲染核心深挖 + 全局风险横扫 + UI 抽查 + 测试覆盖评估）

---

## 1. 架构概览

Magpie Experimental 是 C++/WinRT + D3D11/12 的窗口缩放工具，核心管线为「捕获 → 处理 → 呈现」，跨两个渲染线程：

```
┌─ 前端线程（UI 消息泵）────────────────────────────┐
│  ScalingWindow (单例, RunId 会话守护)              │
│   ├─ Renderer::_frontendResources (D3D11 设备)     │
│   ├─ PresenterBase (Adaptive/CompSwapchain/XeSSFG) │
│   ├─ OverlayDrawer (ImGui 工具栏/参数面板)          │
│   └─ CursorDrawer / PassThroughFrames (原画对比)    │
└──────────────┬───────────────────────────────────┘
               │ 共享纹理环 (≤4 槽, keyed mutex + 槽访问 mutex
               │ + 每槽原子元数据: key/sequence/generation/frameId)
               │ WM_FRONTEND_RENDER(_DLSSFG) 投递 + 槽可用事件
┌─ 后端线程（消息循环 + DispatcherQueue）─────────────┐
│  Renderer::_backendResources (独立 D3D11 设备)      │
│   ├─ FrameSourceBase (DDA/DWM 共享面/WGC/GDI)       │
│   ├─ EffectDrawer 链 (MagpieFX HLSL)                │
│   ├─ NativeEffectBackend (DLSS SR/FSR2/3/4/XeSS…)  │
│   ├─ DLSSFrameGenerator / FrameGuidanceService      │
│   └─ ReflexController (NVAPI 低延迟帧同步)          │
└───────────────────────────────────────────────────┘
```

- **会话生命周期**：`ScalingSessionLifetime`（RunId 比对）贯穿所有异步回调，防止旧会话的 dispatcher/回调触碰新会话。
- **帧同步**：四后端（None/FrontEdge/Reflex/XeLL-Async），`_appliedFrameSyncBackend` 由后端线程探测驱动可用性后切换，失败回落有日志。
- **DLSSFG 同步呈现**：后端通过 4 槽环形共享纹理 + `_pendingDLSSFGFrontendFrames` 计数 + 槽事件实现背压；前端按 FIFO 逐帧消费并归还槽位。

线程模型复杂度极高，但所有权注释（"只能由前台线程访问/后端线程访问/所有线程"）在 `Renderer.h` 中标注清晰，是本项目最突出的工程质量亮点之一。

---

## 2. 发现清单（按严重度排序）

严重度定义：**Critical** = 可稳定触发的内存安全/安全漏洞；**High** = 特定但现实条件下崩溃/数据损坏；**Medium** = 潜伏缺陷或高风险设计；**Low** = 质量问题/边角缺陷。

### [High-1] 特效源文件 ≤1 字节触发无符号下溢 → 越界读写/堆损坏

- **位置**：`src/Magpie.Core/EffectCompiler.cpp:66-118`（`RemoveComments`，关键行 71）
- **证据**：
  ```cpp
  // 编译入口只挡住了"完全为空"的文件（EffectCompiler.cpp:1691-1694）
  if (source.empty()) { return 1; }   // 1 字节文件不在此列
  ...
  // RemoveComments 内：
  if (source.back() != '\n') { source.push_back('\n'); }  // size 可为 1
  for (size_t i = 0, end = source.size() - 2; i < end; ++i) {  // size==1 → end==SIZE_MAX
  ```
- **影响**：用户在 effects 目录放置仅含 1 字节（如单个 `\n` 或单个空格）的 `.hlsl` 文件后，启用该特效即触发：`end` 下溢为 `SIZE_MAX`，循环对 `std::string` SSO 缓冲区之外的内存持续读 + 写（`source[j++] = source[i]`），直至段错误或堆元数据损坏。行注释分支 `while (source[i] != '\n') ++i;`（76-79 行）同样依赖越界读取寻找换行。
- **触发路径**：本地用户自建/损坏的特效文件（本项目明确支持用户自写 MagpieFX 特效，属现实输入）。
- **修复建议**：函数入口加 `if (source.size() < 2) { source.clear(); return 0; }`；行注释扫描改为有界循环。可为特效编译入口补一组 0/1/2 字节文件的边界单测。

### [High-2] `AppXReader::GetIcon` 在 `noexcept` 内构造未转义正则 → 第三方清单数据可致进程崩溃（已复核）

- **位置**：`src/Magpie/AppXReader.cpp:663`（构造点）；`src/Magpie/AppXReader.h:24-28`（`GetIcon(...) noexcept`）
- **证据**：
  ```cpp
  // iconName 来自包清单 Square44x44Logo 属性，未做正则元字符转义
  std::wregex regex(fmt::format(L"^{}\\.[^\\.]+\\{}$", iconName, extension), std::wregex::nosubs);
  ```
  同一 `noexcept` 作用域内还有多个可抛调用（`SoftwareBitmap` 构造、`fmt::format`、`LockBuffer`、`UISettings().GetColorValue`）。
- **影响**：任一已安装应用的包清单图标名含正则元字符（如 `icon(1).png`、`logo[s].png`、`a*b.png`，微软商店缩放图标的实际命名形态），`std::wregex` 构造抛 `std::regex_error` → `std::terminate` → 整个进程崩溃。触发数据来自第三方清单，不受用户控制。调用方为后台 `fire_and_forget` 协程（`RootPage::_LoadIcon`、`ProfileViewModel::_LoadIcon`），无 try/catch 兜底。
- **修复建议**：对 iconName 做正则元字符转义，或改用普通字符串前后缀匹配；同一 `noexcept` 域内的可抛操作统一包 try/catch 降级。

### [High-3] `ProfileViewModel::_data` 裸指针在删除/新增/移动后失去不变量 → 空解引用与悬垂读写（已复核）

- **位置**：`src/Magpie/ProfileViewModel.cpp:323`（`Delete()` 置空）与 :339/367/401/426/523/632 等大量无守卫 `_data->` 访问器；根因链：`ProfileService::RemoveProfile`（ProfileService.cpp:172-173）先 erase 后通知、`AddProfile`（:148-151）`emplace_back` 可重分配 vector
- **证据**：仅 `Name`/`RenameText`/`Rename`/`CanDrag`/`Launch` 检查 `_data`；`ScalingMode` getter 直接 `return _data->scalingMode + 1;`。仓库自身纪律（`ScalingModeItem.cpp:81-84` 的 `_IsRemoved()` 守卫，注释"WinUI may update bindings after removal"）未应用到此处。
- **现实触发场景**：① `ChangeExeForLaunching()`（:147-201）在后台线程跑 `IFileOpenDialog`，对话框打开期间删除 profile → 回调仅 `weakThis.get()` 检查存活、不检查 `_data` → 空解引用；② `AddProfile` 重分配后，存活于 Frame 返回栈中的 ProfilePage VM 继续持有旧存储指针。
- **修复建议**：所有 `_data` 访问器统一加 `if (!_data) return 默认值;`；服务层变更时重绑/失效 ProfilePage VM；对话框回调改 `if (auto strong = weakThis.get())` 并复检 `_data`。

### [Medium-1] 发布事务中 `_sharedFrameMetadata` 槽位被锁内连续写两次，第二次覆盖第一次

- **位置**：`src/Magpie.Core/Renderer.cpp:3757` 与 `:3780`（`_PublishBackendTexture`）
- **证据**：
  ```cpp
  _sharedFrameMetadata[sharedTextureSlot] = frameMetadata;      // 写1: valid=true, stage=Published/GeneratedOutput
  ... CopyResource / 原子元数据写入 ...
  _sharedFrameMetadata[sharedTextureSlot] = publicationMetadata; // 写2: 覆盖，SDR 下 valid=false、stage 为默认值
  ```
  `publicationMetadata` 在非 HDR 兼容路径是默认构造的 `HdrFrameMetadata{}`（仅回填 frameId/captureSequence/generated 等），不含写 1 设置的 `stage`/`valid`。
- **影响**：当前前端只消费 `.generated`（`Renderer.cpp:995`），未触发活性 bug；但任何未来依赖 `stage`/`valid` 的消费者（HDR 元数据链路已在扩张）会静默拿到无效值。代码注释自述"Frame identity belongs to every published image"——意图与实现已经分叉。
- **修复建议**：删除写 2（或让写 2 合并缺失字段）；补一条断言/单测锁定槽元数据的最终字段。

### [Medium-2] FSR 4 INT8 provider 支持覆写：进程内改写第三方 DLL 机器码

- **位置**：`src/Magpie.Core/FSR3Upscaler.cpp:105-185`（`ForceFsr4Int8ProviderSupport`），同构副本 `FSR3ZeroMVUpscaler.cpp:88-180`
- **证据**：RTTI TypeDescriptor 字符串扫描 → 定位 COL → 定位 vtable → `VirtualProtect` 后向 `IsSupported` 函数写入硬编码 x64 机器码 `{0xB8,0x01,0x00,0x00,0x00,0xC3}`（`mov eax,1; ret`），永不恢复。
- **缓解因素**（复核后确认）：整个函数位于 `#ifdef MP_ENABLE_FSR3_ZEROMV` 内，而 `src/Common.Post.props:28` 仅在 `'$(Platform)' == 'x64'` 时定义该宏 → ARM64/ClangCL 构建路径已被隔离；查找算法有 COL 自洽性校验（`selfRva == colRva`）、可执行段校验。
- **残余风险**：
  1. 依赖 MSVC x64 RTTI 布局与 vtable 槽位顺序（注释自认 "deleting destructor, CanProvide, IsSupported"）——FSR SDK 小版本更新即可能错补别的函数；
  2. 补丁假设目标函数 ≥6 字节且前 6 字节无跳转重定位——未校验；
  3. 若未来有人放宽平台条件（宏名不含 x64 语义），ARM64 上即崩溃。
- **修复建议**：至少 (a) 在宏定义处与函数注释双向标注"x64-only 机器码"；(b) 运行时加 `#if !defined(_M_X64)` `static_assert`；(c) 优先寻找 SDK 官方开关替代补丁。

### [Medium-3] DLSSNR 签名 snippet 兼容层：全局永久 IAT 钩子

- **位置**：`src/Magpie.Core/DLSSNRFilter.cpp:745-930`（`InstallSnippetCallerCompatibility` / `SnippetGetModuleFileNameW`）
- **证据**：通过 PE 导入表定位 snippet DLL 的 `GetModuleFileNameW` IAT 槽，`InterlockedExchangePointer` 替换为自有 stub，使该 DLL 认为自己被 `nvngx.dll` 调用；`g_snippetCallerModule`/`g_snippetOriginalGetModuleFileNameW` 为进程级全局，CAS 保证单会话所有，**但从不卸载**。
- **影响**：实现本身防御到位（RVA 边界检查、owner CAS、原函数转发、缓冲区参数校验）；风险在于钩子生命周期 = 进程生命周期，snippet DLL 卸载后其模块句柄仍被 stub 匹配（`GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT` 取的句柄），理论上可对后续加载到同地址的其他模块产生误匹配。
- **修复建议**：会话结束时评估恢复原 IAT 指针的可行性；至少在 `g_snippetCallerModule` 匹配前增加模块名复核。

### [Medium-4] 截图协程在会话结束时可能被遗弃 → 协程帧与资源泄漏

- **位置**：`src/Magpie.Core/Renderer.cpp:620-640`（`TakeScreenshot`）、`3920-4155`（`_TakeScreenshotImpl`）
- **证据**：`winrt::fire_and_forget` 成员协程；非"显示画面"截图 `co_await dispatcher`（后端 DispatcherQueue）挂起。`Renderer::~Renderer` 只 join 后端线程并退出其消息循环（`WM_QUIT`），**不排空 dispatcher 队列**——已挂起未恢复的协程帧（含 staging 纹理 com_ptr、设备引用）永不释放。
- **缓解因素**：设计者已刻意让 `CopyResource` 之后协程不再触碰 `this`（注释 "can outlive it"），截图全程持有 COM 引用，无 UAF；泄漏仅在"截图进行中恰逢会话关闭"的窗口发生，量级有限。
- **修复建议**：会话关闭时显式取消（如 `ScalingSessionLifetime` 挂接取消标志，协程在每个恢复点检查后提前 `co_return`）；或对 dispatcher 使用带超时的排空。

### [Medium-5] 更新链路（休眠代码）使用 MD5 完整性校验

- **位置**：`src/Magpie/UpdateService.cpp:19`（`UPDATE_CHECK_ENABLED = false` 整体禁用）、下载哈希校验段（BCrypt MD5）
- **影响**：当前为死代码，无现实风险。若未来重新启用：`version.json` 经 HTTPS（GitHub raw）获取、包校验用 MD5——对"完整性"尚可，对"抗碰撞"不足；且解压后未对新 `Magpie.exe` 做签名/哈希复核（Updater 直接搬移执行）。
- **修复建议**：重新启用前升级为 SHA-256，并在 Updater 搬移前验证目标文件 Authenticode 签名。

### [Low-1] 全透明图标平均亮度计算除零 → NaN

- **位置**：`src/Magpie/AppXReader.cpp:559` — `const float lumaAvg = lumaTotal / lumaCount;`
- **影响**：全透明（alpha 全 0）图片时 `lumaCount==0`，结果 NaN 向下游主题判定传播（比较均为 false，行为退化为默认值，不崩溃）。
- **修复建议**：`lumaCount ? lumaTotal / lumaCount : 0.0f`。

### [Low-2] WARP 回落失败日志记录了错误的 HRESULT

- **位置**：`src/Magpie.Core/DeviceResources.cpp:150-156` — `hr` 此处是 `EnumWarpAdapter` 的返回值，失败日志 "创建 WARP 设备失败" 复用它，误导排障。
- **修复建议**：单独记录 `_TryCreateD3DDevice` 失败上下文。

### [Low-3] `PassInclude::Open` 在 `noexcept` 上下文中执行可抛操作

- **位置**：`src/Magpie.Core/EffectCompiler.cpp:33-52` — `new char[file.size()]`、`std::string`/`std::wstring` 构造均在 `noexcept` 内，OOM 时直接 `std::terminate`（而非传播给编译错误路径）。
- **修复建议**：捕获后返回 `E_OUTOFMEMORY`。

### [Low-4] `_SubmitFrontendFrame` 重复赋值（编辑残留）

- **位置**：`src/Magpie.Core/Renderer.cpp:1094-1095` — `_frontendPresentedFrameMetadata = _frontendFrameMetadata;` 连写两遍。与 Medium-1 同源的复制粘贴残留，建议清理。

### [Low-5] 前端共享纹理打开路径硬编码 4 个槽锁

- **位置**：`src/Magpie.Core/Renderer.cpp:645-647` — `scoped_lock(m[0],m[1],m[2],m[3])` 与 `MAX_SHARED_TEXTURE_SLOTS` 常量无编译期绑定；改动常量时此处静默失配。
- **修复建议**：改用 `std::apply`/索引折叠展开数组，或加 `static_assert(MAX_SHARED_TEXTURE_SLOTS == 4)`。

### [Low-6] DesktopDuplication 对源矩形位于监视器内的校验仅存在于 assert

- **位置**：`src/Magpie.Core/DesktopDuplicationFrameSource.cpp:56-59` — release 构建下若前置窗口定位未达预期（`_InitialMoveSrcWindowInFullscreen` 时序异常），`_frameInMonitor` 越界传入 `CopySubresourceRegion`，结果为黑屏/静默丢弃而非明确报错。
- **修复建议**：release 下改为显式范围检查并返回 `FrameSourceState::Error`。

### UI 层补充发现（五模型调查返回 + 人工复核采纳）

### [Medium-6] `CandidateWindowItem::_ResolveWindow` 将裸 `this` 捕入延迟执行的 dispatcher lambda（已复核）

- **位置**：`src/Magpie/CandidateWindowItem.cpp:116-127`
- **证据**：`weakThis.get()` 仅在**入队前**检查存活；`TryEnqueue` lambda `[this, ...]` 执行时无保护，直接写 `_defaultProfileName`/`_aumid` 并触发 `PropertyChanged`。同文件图标半程（:148-171）用的是正确的 `weakThis.get()` 模式。
- **影响**：条目在入队与执行之间被销毁（对话框关闭、候选列表重建）→ UAF。窗口窄但真实。
- **修复建议**：lambda 改 `[weakThis, ...]`，执行时 `if (auto strong = weakThis.get())`。

### [Medium-7] `XamlHelper::UpdateThemeOfXamlPopups` 空子弹引用

- **位置**：`src/Magpie/XamlHelper.cpp:61-63` — `popup.Child().try_as<FrameworkElement>()` 后未判空即调 `child.RequestedTheme(theme)`；:71 处 `get_class_name(popup.Child())` 同理。主题切换路径（RootPage.cpp:443、ToastPage.cpp:301）可 AV。Popups 几乎总带 FrameworkElement 子元素，实际触发率低。
- **修复建议**：补 `if (!child) continue;`。

### [Medium-8] 疑似强引用环：自拥有集合的 `VectorChanged(auto_revoke, {this,…})` 与 VM↔参数互持（需运行时确认）

- **位置**：`src/Magpie/ScalingModesViewModel.cpp:27-28`、`src/Magpie/ScalingModeItem.cpp:64-65`（自拥有集合自订阅）；`src/Magpie/EffectParametersViewModel.cpp:137/152/165/179`（`PropertyChanged({this,…})` 裸订阅无 token/revoker，VM 又强持同批参数对象）
- **分析**：C++/WinRT 实现类 `this` 构造的委托默认 `get_strong()` 捕获 → 自引用环，`auto_revoke` 的析构器永不执行 → 疑似每次访问对应页面泄漏一簇对象。`ScalingModeItem::Detach()` 只在移除/重置路径断环，普通导航离开不断。若捕获是弱引用则同一代码反而是悬垂 this——两种情况都该修。
- **验证与修复**：析构断点/委托计数器确认后，统一改 `auto_revoke` + `get_weak()`。

### [Medium-9] 硬编码中文提示字符串（i18n 回归）

- **位置**：`src/Magpie/ScalingModeItem.cpp:241-250`（`EffectAddProblem()` 返回字面量，经 AutomationProperties/选择器详情面板露出）；`src/Magpie/ScalingModesPage.cpp:204-206`（选择器失败 toast）——其余均走 .resw（20 个资源文件），非中文用户看到中英混杂。
- **修复建议**：接入资源加载器。

### [Medium-10] `AppXReader` 缓存：永久负缓存 + 并发 TOCTOU

- **位置**：`src/Magpie/AppXReader.cpp:174-198` — 锁内先占位空 `AppxCacheData` 再解析；并发 `Initialize(aumid)` 读到空条目失败；占位后的任何失败（含瞬态 `ERROR_INSUFFICIENT_BUFFER`）被缓存到进程结束（`ClearCache()` 仅 RootPage.cpp:55 调用）。影响为图标/名称缺失直至重启，表层缺陷。

### [Medium-11] `ScalingService::EffectParametersChanged` 跨线程调用面待验证

- **位置**：`src/Magpie/EffectParametersViewModel.cpp:213-216` 在事件回调中直接改 XAML 绑定对象。静态追踪显示主要发起方已在 UI 线程（ScalingService.cpp:577-585 入队、:776 dispatcher 回调内），但重试完成/会话合并路径无法完全静态闭环。建议在两个发起点加一次性线程断言或显式 dispatcher 入队。

### [Medium-12] `ScalingModeItem` 予子项的裸 `Removed` 订阅无退订 + 无界 erase

- **位置**：`src/Magpie/ScalingModeItem.cpp:159-192` — `Removed(std::bind_front(..., this))` 无 revoker；`Detach()` 不覆盖这些子项句柄；`_ScalingModeEffectItem_Removed` 的 erase/GetAt 缺少同文件 :126-129 有的边界检查。当前同属一主不会泄漏，但容器回收复用 DataContext 的假设一旦破坏即悬垂分发。

### UI 层低级项（合入既有 Low 列表精神，从简）

- `AppXReader.cpp:83,90` `SHLoadIndirectString` 传 `size()+1` 越过缓冲区约定；`:568,595` `CreateReference().data()` 临时对象生命周期依赖 BitmapBuffer 锁；`:597` `(0xff << 24)` 符号位移 UB（MSVC 结果正确）；`:745-755` 二次 `FindPackagesByPackageFamily` 只查 HRESULT、成功但条目变少时复制未初始化指针，`:774` `_packagePath.back()` 空串 UB。
- `ProfileViewModel.cpp:196-198`、`HomeViewModel.cpp:399,425` `if (weakThis.get())` 临时强引用反模式（条件结束后即析构，应用 `if (auto strong = ...)`）。
- `CandidateWindowItem.cpp:132-135`、`ProfileViewModel.cpp:975-979` 在 `resume_background()` 后读 `CurrentDpi()`/`IsLightTheme()` 等 UI 亲和值。
- `ProfileViewModel.cpp:82,127`、`SettingsViewModel.cpp:110` `fire_and_forget ... noexcept` 协程内含未检查 WinRT 调用，异常逃逸即 terminate。
- `ScalingModesPage.cpp:52,70` `_parameterSliders` 以 `get_abi(slider)` 为键，Unloaded 未触发或地址复用时静默丢失 Ctrl 点按重置注册。

### 观察项（不计入缺陷）

- `_PublishBackendTexture` 中 `_fenceEvent.wait()` 无超时（Renderer.cpp:3843 附近）：设备移除场景下 D3D fence 会被置信号，理论可接受；如遇驱动挂死会卡死后端线程（析构函数的 PostThreadMessage 循环能容忍线程不退出吗——不能，`~Renderer` 将永久 join）。建议评估加长超时 + 设备丢失检测。
- `Renderer` 类约 570 行头文件、200+ 成员，前/后端/共享状态按注释分区但无类型系统强制。中期建议把槽环协议抽出独立的 `PresentationRing` 类（keyed mutex 事务 + 元数据 + 事件已自成体系），`Renderer` 只留编排。
- `DwmSharedSurfaceFrameSource` 使用未公开 API `DwmGetDxSharedSurface`（上游继承设计，负坐标监视器上初始化失败已显式报错，可接受）。

---

## 3. 测试覆盖评估

**现状**：`tests/` 含 14+ 个 PowerShell 驱动的 C++ 单测套件（帧同步策略/StepTimer、DLSS R2 资源、HDR 组件边界、Reflex 控制器、effect picker 等），测试方法相当讲究——通过 Python 从生产源码**抽取**被测单元（如 `prepare_frame_sync_timer_test.py` 提取 StepTimer）编译进测试，并包含负向变异回归（`negative_policy` 用退出码 42 验证拒绝预期突变）。

**缺口**：
1. **CI 不运行任何测试**：`.github/workflows/build.yml` 仅执行 `publish.py` 构建签名，PS1 套件未接入——测试存在但无强制力。
2. **EffectCompiler 解析器零覆盖**：本报告 High-1 即位于此；建议加 0/1/2 字节、未闭合块注释、超长 token 等边界用例（解析器是纯函数逻辑，最易测）。
3. **Renderer 共享纹理环/发布-消费协议无单测**（依赖 D3D 设备，可仿照现有"抽取生产代码"手法对纯逻辑部分（世代/序号判定、丢弃谓词）建测）。
4. 截图管线、帧源（硬件依赖，可接受）、OverlayDrawer 无测试。
5. **UI 层无系统化测试**：ViewModel/事件绑定逻辑依赖人工回归（scripts/tests/ 下的 Python 测试覆盖了 profile 参数焦点等近期特性，但未接入 CI）。

---

## 4. 亮点与做得好的地方

1. **并发所有权文档化**：`Renderer.h` 每组成员标注访问线程；`ReflexController.h` 顶部注释明确锁纪律（"Sleep never holds the configuration mutex"）——大型图形代码里罕见。
2. **世代/序号双重防陈旧机制**：发布-消费全链路用 `captureSequence` + `resourceGeneration` + key 序号交叉验证陈旧帧并显式丢弃（含日志），缩放恢复/resize 场景考虑周全。
3. **会话生命周期守护**：所有跨线程回调捕获 `session = _sessionLifetime` 并比对 `RunId`，异步回调不触碰死会话；`ScalingWindow::RequestStop(runId)` 同理。
4. **关键互操作边界 SEH 防护**：NGX 调用统一经 `NgxRuntimeGuard::Invoke` 包裹，驱动崩溃转可控错误（`NgxRestartRequired`）。
5. **测试方法论**：抽取生产代码编译 + 负向变异验证，杜绝"测试副本与实现漂移"。
6. 常规卫生良好：wil 资源包装全覆盖、`strcpy` 族仅 1 处且用 `wcscpy_s`、裸 new/delete 极少且均立即入 `unique_ptr`。
7. **AcquirePresentationTextures 对 keyed mutex 语义的精确处理**（`Renderer.cpp:61-80` 注释点明 `AcquireSync` 返回 `WAIT_TIMEOUT` 是非负值，`FAILED(hr)` 不构成判据）——细节功底扎实。
8. **UI 层事件生命周期纪律**：全部 ViewModel 使用 `auto_revoke` 智能退订（ProfileViewModel.cpp:44-67、HomeViewModel.cpp:27-41），无裸 token 泄漏模式。
9. **EffectCatalog 嵌入式资源解析健壮**（EffectCatalog.cpp）：资源缺失/JSON 损坏均优雅降级，含中英文资源回退链；`EffectCacheManager` 以完整 key 比对防御 64 位哈希碰撞（EffectCacheManager.cpp:145-148）。

---

## 5. 附录：假设与限制声明

1. **纯静态审查**：审查环境为 Linux，无法编译/运行此 MSVC/WinRT/D3D 项目；所有结论基于源码阅读，未经运行时验证。
2. **范围假设**：用户未明确范围（澄清问题超时），按"全项目总体审查"执行：`Magpie.Core` 深挖（Renderer/帧同步/帧源/缩放器/EffectCompiler）、`Effects`（HLSL 着色器本身未逐个审查——为数据资源）、`Magpie`（UI 抽查，含并行模型深查）、`Updater`/`UpdateService`、`Shared`（SmallVector 为 LLVM 移植代码，未审）。
3. **行号基准**：`7eb19987`（feat(effect-picker): localize catalog and guidance），工作区无未提交源码改动。
4. **严重度评级**：按"可触发路径 × 影响面"综合评定；本地单用户桌面软件语境下未发现可被远程/低权限攻击者利用的 Critical 问题。
5. **补充说明**：审查过程中另启动了两个五模型并行只读调查（Effects 编译管线、UI 层）。UI 层调查已返回，其发现经人工抽样复核（High-2/High-3/Medium-6 均实地验证属实）后并入第 2 节；未采纳项：与既有条目重复或经验证不成立的部分。Effects 编译管线调查返回后如有增量发现，同样按此流程增补。
