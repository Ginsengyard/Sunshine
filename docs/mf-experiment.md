# Sunshine × ThinkPad E531：MediaFoundation 硬编实验档案

> 日期：2026-09-27 ~ 09-29。机器：ThinkPad E531 (68854UC)，i7-3740QM + HD 4000（驱动 10.18.10.5161，**独占所有显示输出**）+ GT 740M / GK208M（`PCI\VEN_10DE&DEV_1292&SUBSYS_501917AA`，驱动 25.21.14.1891 = 418.91），muxless Optimus。
> 一句话结论：**MF 路线已打通并端到端验证**——白名单放行 + `main()` 持有 MF 平台引用后，Intel QSV MFT 在真实 Moonlight 会话中以 h264_mf 硬编出帧（720p / 1080p 实测成功，非提权与提权皆可）。根因是**双层**的：① 同纪元内前几次 `SetOutputType` 必败（驱动"打火"，由持有引用 + 内置重试吸收）；② 编码器有**硬件能力上限**，客户端请求超出（如 2400x1080@60）会被永久拒收（连败 199 次），需靠客户端分辨率/码率约束或后续降级补丁规避。

## 1. 四条硬编路径状态

| 路径 | 状态 | 证据 |
|---|---|---|
| NVENC (GT 740M) | 死 | notebook 驱动上限 425.31 → NVENC API 9.0；Sunshine 要求 ≥11.0（456.71+）。`NvEncodeAPIGetMaxSupportedVersion` P/Invoke 实测 `RAW=0x00000090`；425.31-notebook INF 含 `DEV_1292&SUBSYS_501917AA`，461.09-notebook 不含 |
| QSV (HD 4000, oneVPL) | 死 | oneVPL `CheckValidLibraries()` 过滤 gen7，`MFX_ERR -9`；libmfx 会话可建（API 1.27/1.11）但现代 ffmpeg 已走 libvpl |
| CUDA 通用编码 | 死 | 无 CUDA 编码器路径；GK208 = sm_35 < sm_50 |
| **MediaFoundation（Intel QSV MFT）** | **✅ 端到端打通（09-29）** | 720p / 1080p Moonlight 会话实测成功，走 `h264_mf` + Intel QSV MFT；见 §4.3 / §5 |

## 2. MF 路线改了什么（两处补丁）

### 2.1 能力判定白名单（commit `f250175`，1 行）

`src/platform/windows/display_vram.cpp`，`is_codec_supported()` 的 Intel 分支：

```diff
-      if (!boost::algorithm::ends_with(name, "_qsv")) {
+      if (!boost::algorithm::ends_with(name, "_qsv") && !boost::algorithm::ends_with(name, "_mf")) {
         return false;
       }
```

原因：该函数对 Intel adapter（VendorId 0x8086）只放行 `*_qsv`，`h264_mf` 在能力判定阶段即被丢弃（特征：`Trying encoder [X]` 后**没有** `Creating encoder`）。

### 2.2 持有 MF 平台引用（commit `0183d0bc`，初版 `b71b8336`）

`src/main.cpp` 的 `main()` 开头（Windows 专属）：

```cpp
#ifdef _WIN32
  // Hold a Media Foundation platform reference for the lifetime of the process.
  // The Intel QSV H.264 encoder MFT fails the first IMFTransform::SetOutputType()
  // calls of every MFStartup() epoch with MF_E_INVALIDMEDIATYPE; those calls only
  // arm the driver.  FFmpeg's mfenc wraps every encoder open in
  // MFStartup()/MFShutdown(), so every open would otherwise start a fresh epoch.
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

- MFStartup/MFShutdown 引用计数；持有一个引用后所有编码器打开共享同一纪元，"打火"状态不丢失。
- Sunshine 的会话编码器创建带**重试循环**：720p 会话实测第 1~3 次失败、第 4 次成功；1080p 会话同样在前几次失败后成功。
- **编译坑**：`-Werror` 下勿在 boost/asio（引入 winsock2.h）之前 include `<windows.h>`——`#error WinSock.h has already been included`（初版 `b71b8336` 栽在这，`0183d0bc` 修复）。

## 3. CI 与下载的坑（全部踩过）

1. 调用 reusable workflow 的 job 必须显式 `permissions: contents: read`，否则 `startup_failure`。
2. `ci.yml` 的 release-setup 在 fork 上必失败；用包装工作流 `ci-windows-mf.yml`（push `mf-mft` 即触发；矩阵固定 AMD64 + ARM64，验证只看 AMD64）。
3. fine-grained PAT 不能写 `.github/workflows/`（403）；`GITHUB_TOKEN` 产生的 commit 不触发 workflow。
4. Actions artifact 匿名下载 404（公开仓库也要登录）。
5. **下载限速按连接**：单连接 ~25–30 KB/s（直连/代理皆然）；8 路并行线性叠加（≈0.26 MB/s），**16 路并发整体停滞（0 字节）**；代理健康检查通过后仍可能 TLS 失败（exit 35）→ 下载前先探测可用性、支持直连回退。
6. **只取所需成员**：227MB 的 `build-Windows-AMD64.zip` 里 lite.zip 仅 38MB（另 156MB 是 debuginfo、33MB 是 MSI）。用 HTTP Range 读 EOCD + 中央目录，只拉目标成员并 inflate（`mf_fetch_lite.ps1`），数据量降到 1/6，2 分钟拿到包。PowerShell 里 `0xFFFFFFFF -ge` 是 **int32 的 -1**，zip64 判断要用 `[uint32]::MaxValue`。

## 4. 本机探测方法学与实测

### 4.1 方法学

- portable 版配置/日志在 `<exe 目录>\config\`（`src/platform/windows/misc.cpp:149`）。
- 无凭据时 `http::init()` 失败自退出；`video::probe_encoders()` 在其之前 → 不需要凭据即可拿到探测日志。
- 判据行（`src/video.cpp`）：`Trying encoder [X]` / `Creating encoder [Y]` / `Encoder [X] is not supported on this GPU` / `Encoder [X] failed` / `Found H.264 encoder: Y [X]`。
- **verbose 是定位拒收字段的关键武器**：`min_log_level = verbose` 后 mfenc 打印每次 `setting output type` 的全部 `MF_MT_*` 属性。
- 脚本（`D:\work\qoder01\mfwork\`）：`mf_probe.ps1`（-Elevated）、`mf_probe_cs.ps1`（C#/COM 探针矩阵）、`mf_watch_build.ps1`、`mf_fetch_lite.ps1`、`mf_mft_matrix.ps1`、`mf_intel_hidden.ps1`、`mf_pipeline.py`、`mf_dispatch.py`、`mf_run.ps1`。

### 4.2 硬件可接受集合（ffmpeg CLI 直驱）

D3D11 输入 + `hw_encoding=1 -rate_control cbr -scenario display_remoting`：1080p60 profile 66/77/100、1080p30 High、720p60 High **全部成功**（临时改名 `nvEncMFTH264.dll` 逼选 Intel，测完改回）。

### 4.3 端到端会话实测（09-29，Sunshine + Moonlight + verbose）

| 客户端请求 | 结果 | 日志证据 |
|---|---|---|
| 1280x720 / 7.3 Mbps | ✅ 成功（两次连接均成功） | 第 1~3 次 `SetOutputType` 失败 → 第 4 次成功；`MFT name: Intel QSV`；全程无 libx264 回退 |
| 1920x1080 / 20 Mbps | ✅ 成功 | 编码器 input/output 均 1920x1080（NV12）；桌面 1366x768 被 GPU 视频处理器**放大**后再编码（真 1080p 码流、细节仍 768p 源） |
| 2400x1080 / 47 Mbps | ❌ 永久失败（连败 199 次直到客户端断开） | `setting output type: 2400x1080 @ 46,988,000` → `MF_E_INVALIDMEDIATYPE`；另有 `Client requested reference frame limit, but encoder doesn't support it!` |

结论：

- 会话**首次**连接也可能先失败 1~3 次（驱动打火），重试即成功——预期行为。
- **超出编码器能力的分辨率会永久失败**：2400x1080@60 宏块吞吐 612,000 MB/s 超出 H.264 Level 5.0（589,824 MB/s），需 Level 5.1；HD 4000 QSV 直接拒收。1920x1080 是已验证可用上限（2400 宽不行）。
- 客户端请求高于桌面分辨率时 Sunshine 会放大后再编码；建议客户端直接用原生 1366x768 或 1280x720，省码率与负载。

### 4.4 独立 C#/COM 探针矩阵（根因证据）

- 同实例重试无效；**换新实例**（同进程）在若干次后成功 → "打火"。
- `MFShutdown` 后再 `MFStartup`（新纪元）→ 再次首败；失败与纪元绑定。
- 进程启动时持有一个额外 MFStartup 引用 → 热状态跨纪元幸存（T6）。

## 5. 结论与残留注记

**双层根因**：① 驱动级"打火"（每纪元前几次 SetOutputType 必败）——由 §2.2 持有引用 + Sunshine 内置重试吸收；② 硬件能力上限——请求超出能力（2400x1080@60）时永久拒收，需客户端分辨率约束（或后续的自动降级补丁）。

**残留注记**：ffmpeg CLI 首调即成功的现象仍未逐位解释；不影响上述结论。

## 6. 资产清单

- 远端：fork `Ginsengyard/Sunshine` 分支 `mf-mft`；commits `f250175`(白名单)、`b034781`/`1aac10e1`(工作流)、`a2b97266`、`bf156936`、`b71b8336`(MF 持有初版)、`0183d0bc`(编译修复)、`f96941ec`(文档 v2)、本次文档 v3；run `36445869005`（**双架构 success**）；artifact `10982606366`。
- 本机 `D:\work\qoder01\mfwork\`：`mfb_fix\Sunshine\`（**验证通过的 MF 版可运行包**，配置含 `min_log_level = verbose`，实测日志在 `config\sunshine.log`）、`art_fix\Sunshine-Windows-AMD64-lite.zip`、`probe_result.txt`（SUCCESS）、`probe_result_success_nonelev.txt`、`sunshine_nonelev.log`（修复前对照）、上述全部脚本、`Sunshine-master\`（源码树）。

## 7. 将来继续时的上手清单

1. `git cherry-pick f250175 0183d0bc`；复制 `ci-windows-mf.yml` 并保留 job 级 `permissions: contents: read`。
2. push `mf-mft` 触发 `CI-Windows-MF`；验证只等 AMD64（ARM64 只是编译体检）。
3. 下载用 `mf_fetch_lite.ps1 -ArtifactId <id>`（8 路并行，别超 12 路）。
4. 排障先开 `min_log_level = verbose`，看每次 `setting output type` 的属性与拒收错误。
5. **客户端分辨率不要超过编码器能力**（本机实测上限 1920x1080；2400x1080 会被永久拒收）。
6. 编译注意：`-Werror` 下勿在 boost/asio 之前 include `<windows.h>`。
7. 任何"改名 System32/Program Files 下 DLL"的实验都必须 finally 改回，并先确认无进程占用。