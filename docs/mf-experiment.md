# Sunshine × ThinkPad E531：MediaFoundation 硬编实验档案

> 日期：2026-09-27 ~ 09-29。机器：ThinkPad E531 (68854UC)，i7-3740QM + HD 4000（驱动 10.18.10.5161，**独占所有显示输出**）+ GT 740M / GK208M（`PCI\VEN_10DE&DEV_1292&SUBSYS_501917AA`，驱动 25.21.14.1891 = 418.91），muxless Optimus。
> 一句话结论：**MF 路线已打通**——白名单放行 + 进程启动时持有一个 MF 平台引用后，`Found H.264 encoder: h264_mf [mediafoundation]` 实测成立（两次复现，非提权）。根因：Intel QSV MFT 在每个 MFStartup 纪元内的**首次** `IMFTransform::SetOutputType()` 必败（`MF_E_INVALIDMEDIATYPE`，仅"打火"），同纪元内后续实例成功。

## 1. 四条硬编路径状态（每条都有本机实测或源码级证据）

| 路径 | 状态 | 证据 |
|---|---|---|
| NVENC (GT 740M) | 死 | notebook 驱动上限 425.31 → NVENC API 9.0；Sunshine/Apollo 要求 ≥11.0（456.71+）。`nvEncodeAPI64.dll!NvEncodeAPIGetMaxSupportedVersion` P/Invoke 实测 `RAW=0x00000090`；425.31-notebook INF 含 `DEV_1292&SUBSYS_501917AA`，461.09-notebook 不含 |
| QSV (HD 4000, oneVPL) | 死 | oneVPL 在 `CheckValidLibraries()` 过滤 gen7，`MFX_ERR -9`；libmfx 会话本身可建（API 1.27/1.11） |
| CUDA 通用编码 | 死 | 无 CUDA 编码器路径；GK208 是 sm_35（低于最低 sm_50） |
| **MediaFoundation（Intel QSV MFT）** | **✅ 已打通（2026-09-29）** | `Found H.264 encoder: h264_mf [mediafoundation]`，probe 两次复现，见 §4.3/§5 |

## 2. MF 路线改了什么（共两处源码补丁）

### 2.1 能力判定白名单（commit `f250175`，1 行）

`src/platform/windows/display_vram.cpp`，`is_codec_supported()` 的 Intel 分支：

```diff
-      if (!boost::algorithm::ends_with(name, "_qsv")) {
+      if (!boost::algorithm::ends_with(name, "_qsv") && !boost::algorithm::ends_with(name, "_mf")) {
         return false;
       }
```

原因：该函数对 Intel adapter（VendorId 0x8086）只放行 `*_qsv`，使 `h264_mf` 在能力判定阶段就被丢弃（日志特征：`Trying encoder [X]` 后**没有** `Creating encoder [...]`）。补丁后进入会话创建阶段。

### 2.2 持有 MF 平台引用（commit `0183d0bc`，初版 `b71b8336`）

`src/main.cpp` 的 `main()` 开头（Windows 专属）：

```cpp
#ifdef _WIN32
  // Hold a Media Foundation platform reference for the lifetime of the process.
  // Intel QSV H.264 encoder MFT fails the first IMFTransform::SetOutputType() call
  // of every MFStartup() epoch with MF_E_INVALIDMEDIATYPE; that call only arms the
  // driver. FFmpeg's mfenc wraps every encoder open in MFStartup()/MFShutdown(), so
  // every open would start a fresh epoch and always fail.
  {
    static bool mf_platform_held = false;
    if (!mf_platform_held) {
      mf_platform_held = true;
      if (auto mfplat = LoadLibraryW(L"mfplat.dll")) {
        using mf_startup_t = long(__stdcall *)(unsigned long, unsigned long);
        if (auto mf_startup = reinterpret_cast<mf_startup_t>(GetProcAddress(mfplat, "MFStartup"))) {
          mf_startup(0x20070 /* MF_VERSION */, 0 /* MFSTARTUP_FULL */);
        }
      }
    }
  }
#endif
```

要点：

- MFStartup/MFShutdown 是**引用计数**的。持有一个引用后，mfenc 每次打开编码器时的 MFStartup/MFShutdown 不会真正拆掉平台 → 所有打开共享同一纪元 → 从第二次打开起 `SetOutputType` 不再首败。
- `validate_encoder()` 对指定编码器本来就做**两次** `validate_config`（先 max-ref-frames、再 autoselect）：第一次消耗"首败"打火，第二次即通过——因此不需要额外的 dummy 预热。
- **编译坑（重要）**：不要在 boost/asio（会引 `winsock2.h`）之前 include `<windows.h>`，否则 `#error WinSock.h has already been included` 在 `-Werror` 下直接挂（初版 `b71b8336` 就栽在这；`0183d0bc` 删掉了多余 include）。`main()` 里用到的 `LoadLibraryW` 本来就由 `confighttp.h`→boost/asio 链传入的 windows.h 提供（基线同 TU 更高处已用 HWND 等类型，可证）。

## 3. CI 与下载的坑（全部踩过）

1. **调用 reusable workflow 的 job 必须显式写 `permissions: contents: read`**；省略则继承调用方 `permissions: {}`，被调方要权限直接 `startup_failure`。
2. **`ci.yml` 的 `release-setup` 在 fork 上必失败**；要构建就触发自己的包装工作流 `ci-windows-mf.yml`（push 到 `mf-mft` 即触发，矩阵固定 AMD64(gcc/ucrt64) + ARM64(clang/clangarm64)，无参数可裁剪；ARM64 只作编译体检，验证只看 AMD64，别等它）。
3. fine-grained PAT 不能写 `.github/workflows/`（403）；需 classic PAT 的 workflow scope 或网页 UI。
4. Actions artifact 匿名下载 404（公开仓库也要登录）；网页端链接是 `.../runs/<id>/artifacts/<aid>`（复数）。
5. Git Data API：`POST /git/trees` 的 `base_tree` 要 **tree 的 SHA**；`GITHUB_TOKEN` 产生的 commit 不触发 workflow（防递归）。
6. **artifact 下载限速（本机链路实测）**：单连接 ~25–30 KB/s（直连/代理皆然），且**限速按连接**——8 路并行线性叠加（≈0.26 MB/s），**16 路并发整体停滞（0 字节）**；代理健康检查通过后仍可能 TLS 失败（exit 35），批量下载前先做一次可用性探测并支持直连回退。
7. **只取所需成员**：`build-Windows-AMD64.zip`(227MB) 内 lite.zip 仅 38MB（另 156MB debuginfo、33MB MSI）。用 HTTP Range 读尾部 EOCD + 中央目录，再按 Range 拉单个成员并 inflate（`mf_fetch_lite.ps1`），下载量降到 1/6，2 分钟拿到包。注意 PowerShell 里 `0xFFFFFFFF -ge ...` 会被当 **int32 的 -1**，zip64 判断要用 `[uint32]::MaxValue`。

## 4. 本机探测方法论与实测结果

### 4.1 方法论（可复用）

- portable 版配置与日志都在 **`<exe 目录>\config\`**（`src/platform/windows/misc.cpp:149` 的 `appdata()`）。
- 无凭据时 `http::init()` 失败 → 打 fatal、睡 10s、`return -1` 自退出；而 `video::probe_encoders()` 在它**之前**执行 → **不需要凭据就能拿到探测日志**。
- 判据原文（`src/video.cpp`）：`Trying encoder [X]`(3072) / `Creating encoder [Y]`(2601) / `Encoder [X] is not supported on this GPU`(3095) / `Encoder [X] failed`(3074) / `Found H.264 encoder: Y [X]`(3448)。
- ffmpeg 侧 verbose 日志（`min_log_level = verbose`）会打印 mfenc 的 MFT 枚举与 `setting output type` 全属性。
- 脚本（本机 `D:\work\qoder01\mfwork\`）：`mf_probe.ps1`、`mf_probe_cs.ps1`（独立 C#/COM 探针矩阵）、`mf_watch_build.ps1`（监视 CI→下载→探测）、`mf_fetch_lite.ps1`（Range 取单成员）、`mf_mft_matrix.ps1`、`mf_intel_hidden.ps1`、`mf_pipeline.py`、`mf_dispatch.py`、`mf_run.ps1`。

### 4.2 三组实验（白名单补丁后）

1. **补丁生效性**：`Trying encoder [mediafoundation]` → `Creating encoder [h264_mf]` ×2 → `Encoder [mediafoundation] failed`（修复前的状态）。
2. **Intel MFT 接受集合**（临时改名 `C:\Windows\System32\nvEncMFTH264.dll` 逼 ffmpeg 选 Intel；测完改回）：D3D11 输入 + `hw_encoding=1 -rate_control cbr -scenario display_remoting` 下，1080p60 profile 66/77/100、1080p30 High、720p60 High **全部成功**。即硬件与媒体类型无罪。
3. **跨 adapter**（临时改名 `C:\Program Files\Intel\Media SDK\mfx_mft_h264ve_64.dll` 逼 Sunshine 选 NVIDIA MFT）：`Could not open codec [h264_mf]: Function not implemented`。跨 adapter 在 Sunshine 内是死的。

### 4.3 失败窗口与修复后对照（Sunshine 进程内，verbose）

修复前：

```
Creating encoder [h264_mf]
MFT name: 'Intel(R) Quick Sync Video H.264 Encoder MFT'   (VEN_8086, ff_MF_SA_D3D11_AWARE=1)
setting output type: ... MF_MT_SUBTYPE=MFVideoFormat_H264
Error: could not set output type (MF_E_INVALIDMEDIATYPE)
Encoder [mediafoundation] failed
```

修复后（2026-09-29，run `36445869005` 产物，`mf_probe.ps1` 实测）：

```
01:33:20.925  Trying encoder [mediafoundation]
01:33:21.040  Creating encoder [h264_mf]        <- 第 1 次打开：消耗"首败"
01:33:21.206    MFT name: 'Intel QSV H.264 Encoder MFT'
01:33:21.206  Error: could not set output type (MF_E_INVALIDMEDIATYPE)      （预期）
01:33:21.209  Creating encoder [h264_mf]        <- 第 2 次打开：同一被持有的 MF 纪元
01:33:21.356    MFT name: 'Intel QSV H.264 Encoder MFT'                     （通过）
01:33:23.050  Found H.264 encoder: h264_mf [mediafoundation]
```

### 4.4 独立 C#/COM 探针矩阵（根因证据）

`mf_probe_cs.ps1` 在隔离进程里复刻 Sunshine 的调用形态（枚举→激活→SetOutputType），跑参数矩阵：

- 同一实例上重试 SetOutputType 无效；**换新实例**（同进程、同纪元）则成功 → "首败打火"。
- MFShutdown 后再 MFStartup（新纪元）→ **再次首败**；失败与 MFStartup 纪元绑定。
- 进程启动时持有一个额外 MFStartup 引用，使 MFShutdown 无法把引用计数降到 0 → 热状态跨"纪元"幸存，后续打开直接成功（T6）。
- 对照：ffmpeg CLI 直接驱动同一 MFT 首调即成功（探针 0/16 vs CLI 100%）——CLI 进程内可能已有别的 MF 活动先"打火"（**未完全解释，见 §5 注记**）。

## 5. 结论与残留注记

**根因（已确证）**：Intel QSV H.264 Encoder MFT 在每个 MFStartup 纪元内的首次 `SetOutputType()` 返回 `MF_E_INVALIDMEDIATYPE`（仅"打火"）；mfenc 每次打开编码器都 MFStartup/MFShutdown，使每次打开都落在新纪元 → 永远首败。

**修复（已验证）**：进程启动时持有 MF 平台引用（§2.2）→ 所有打开共享纪元 → `validate_encoder` 的第二次 `validate_config` 成功 → 实测 `Found H.264 encoder: h264_mf [mediafoundation]`（两次复现，非提权）。

**残留注记（不影响结论）**：ffmpeg CLI 在同一机器上首调即成功的现象仍未逐位解释；最可能是 CLI 进程内在 mfenc 打开前已有其他 MF/MFT 活动"打火"。Sunshine 的修复不依赖该解释。

## 6. 资产清单

- 远端：fork `Ginsengyard/Sunshine` 分支 `mf-mft`；commits `f250175`(白名单)、`b034781`/`1aac10e1`(工作流)、`a2b97266`、`bf156936`、`b71b8336`(MF 持有初版，含编译坑)、**`0183d0bc`(当前 HEAD：编译修复)**；run `36445869005`（**双架构 success**）；artifact `10982606366`（build-Windows-AMD64）。
- 本机 `D:\work\qoder01\mfwork\`：`mfb_fix\Sunshine\`（**验证通过的 MF 版可运行包**）、`art_fix\Sunshine-Windows-AMD64-lite.zip`、`probe_result.txt`（VERDICT: SUCCESS）、`probe_result_prefix.txt`（修复前对照）、`mfb\Sunshine\`（旧包）、上述全部脚本、`Sunshine-master\`（源码树）。

## 7. 将来 fork 继续时的上手清单

1. `git cherry-pick f250175 0183d0bc`（或照 §2.1/§2.2 手改）；复制 `ci-windows-mf.yml` 并保留 job 级 `permissions: contents: read`。
2. 触发构建：push 到实验分支即触发 `CI-Windows-MF`（矩阵含 ARM64，验证时只等 AMD64）。
3. 下载：**别下整个 227MB**，用 `mf_fetch_lite.ps1 -ArtifactId <id>` 只取 lite 成员（Range + 中央目录），8 路并行、别超 12 路；解包后按 §4.1 探测。
4. 编译注意：`-Werror` 下勿在 boost/asio 之前 include `<windows.h>`（winsock 顺序坑）。
5. 想换 ffmpeg：`cmake/dependencies/ffmpeg.cmake` 支持 `FFMPEG_PREPARED_BINARIES` / `FFMPEG_ARCHIVE_NAME` 覆盖（预编译二进制来自 LizardByte/build-deps 的 release）。
6. 任何"改名 System32/Program Files 下 DLL"的实验都必须 finally 改回，并先确认无进程占用。