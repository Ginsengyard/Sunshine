# Sunshine × ThinkPad E531：MediaFoundation 硬编实验档案

> 日期：2026-09-27 ~ 09-28。机器：ThinkPad E531 (68854UC)，i7-3740QM + HD 4000（驱动 10.18.10.5161，**独占所有显示输出**）+ GT 740M / GK208M（`PCI\VEN_10DE&DEV_1292&SUBSYS_501917AA`，驱动 25.21.14.1891 = 418.91），muxless Optimus。
> 一句话结论：**四条硬件编码路径全部实测关闭；MF 路线差最后一步——Intel QSV MFT 在 Sunshine 进程内对一组「独立验证可接受」的媒体类型返回 `MF_E_INVALIDMEDIATYPE`，根因在 Sunshine 进程内、mfenc 之外，需进程内调试才能继续。**

## 1. 四条硬编路径的死因（每条都有本机实测或源码级证据）

| 路径 | 死因 | 证据 |
|---|---|---|
| NVENC (GT 740M) | notebook 驱动上限 425.31 → NVENC API 9.0；Sunshine/Apollo 要求 ≥11.0（456.71+） | `nvEncodeAPI64.dll!NvEncodeAPIGetMaxSupportedVersion` P/Invoke 实测 `RAW=0x00000090`；425.31-notebook INF 含 `DEV_1292&SUBSYS_501917AA`，461.09-notebook 不含 |
| QSV (HD 4000, oneVPL) | oneVPL 在 `CheckValidLibraries()` 过滤 gen7，`MFX_ERR -9` | Sunshine 日志 `Error creating a MFX session: -9`；libmfx 会话本身可建（API 1.27/1.11） |
| CUDA 通用编码 | 不存在：ffmpeg 编码器无 CUDA 路径；Sunshine 唯一 `.cu` 是 Linux VRAM 捕获；arch 列表最低 sm_50，GK208 是 sm_35 | `cmake/compile_definitions/linux.cmake`、FFmpeg codec 列表 grep |
| MediaFoundation | 见 §4：白名单补丁生效后，Intel MFT 在 Sunshine 进程内拒收输出媒体类型 | `mf_window.txt`（本目录/仓库 docs 旁证） |

## 2. MF 路线改了什么

### 2.1 源码补丁（唯一一处，1 行）

`src/platform/windows/display_vram.cpp`，`is_codec_supported()` 的 Intel 分支（上游 master 约 L2097）：

```diff
-      if (!boost::algorithm::ends_with(name, "_qsv")) {
+      if (!boost::algorithm::ends_with(name, "_qsv") && !boost::algorithm::ends_with(name, "_mf")) {
         return false;
       }
```

原因：该函数对 Intel adapter（VendorId 0x8086）只放行 `*_qsv`，使 `h264_mf` 在能力判定阶段就被丢弃（日志特征：`Trying encoder [X]` 后**没有** `Creating encoder [...]`）。补丁后进入会话创建阶段。对应 commit：`f250175`（分支 `mf-mft`）。

### 2.2 CI 构建（14 行包装工作流）

`.github/workflows/ci-windows-mf.yml`（commit `b034781`）不复制上游 200 行构建逻辑，直接复用：

```yaml
jobs:
  windows:
    name: Windows
    permissions:
      contents: read        # 必不可少，见 §3.1
    uses: ./.github/workflows/ci-windows.yml
    with:
      publish_release: 'false'
      release_commit: ${{ github.sha }}
      release_version: 2026.927.133000   # 必须是 WiX 可接受的数字形状
```

触发：push 到 `mf-mft`，或用带 **Actions: write** 的令牌 `POST /repos/<o>/<r>/actions/workflows/ci.yml/dispatches {"ref":"mf-mft"}`（注意：dispatch `ci.yml` 会跑全平台且 `release-setup` 在 fork 上必失败，见 §3.2；要构建就 dispatch/触发自己的包装工作流）。成功 run：`36324333523`（AMD64+ARM64 全绿），artifact id `10933734851`。

## 3. CI 与凭据的坑（全部踩过）

1. **调用 reusable workflow 的 job 必须显式写 `permissions: contents: read`**。省略则继承调用方顶层 `permissions: {}`（零授权），被调方再要 `contents: read` 直接 `startup_failure`（无任何 job 日志）。
2. **`ci.yml` 的 `release-setup` 在 fork 上必失败**，`needs:` 链会跳过全部构建 job。别用「dispatch ci.yml」当构建入口。
3. **fine-grained PAT 不能写 `.github/workflows/`**：同令牌同 endpoint，写 `src/**` 返回 200、写 workflows 返回 `403 Resource not accessible by personal access token`。只有 classic PAT 的 `workflow` scope 或网页 UI 能过。
4. **Actions artifact 匿名下载 404**（公开仓库也要登录）。网页端链接是 `.../runs/<id>/artifacts/<aid>`（复数）。
5. Git Data API：`POST /git/trees` 的 `base_tree` 要 **tree 的 SHA**，传 commit SHA 会得到误导性 403。
6. 网页新建文件页 `/new/<branch>/<path>` 把 URL 末段当**目录**（要在 File name 框只填文件名）；提交对话框在 portal 里、accessibility 快照看不见，用 **Ctrl+Enter** 提交；同路径已被目录占用时报 "A file with the same name already exists"，删除直达页是 `/delete/<branch>/<path>`。
7. `GITHUB_TOKEN` 产生的 commit 不触发 workflow（防递归）；「CI 内改源码再 push 触发第二轮」天然走不通。

## 4. 本机探测方法论与实测结果

### 4.1 方法论（可复用）

- portable 版的配置与日志都在 **`<exe 目录>\config\`**（`src/platform/windows/misc.cpp:149` 的 `appdata()` = exe 目录 + `\config`；`src/config.cpp:883/888`）。与 Apollo/安装版完全隔离。
- 无凭据时 `http::init()` 失败 → 打 fatal、睡 10s、`return -1` 自退出（`src/httpcommon.cpp:66`、`src/main.cpp:464`）；而 `video::probe_encoders()` 在它**之前**执行 → **不需要凭据就能拿到探测日志**。
- 判据原文（`src/video.cpp`）：`Trying encoder [X]`(3072) / `Creating encoder [Y]`(2601) / `Encoder [X] is not supported on this GPU`(3095，能力判定阶段被拒) / `Encoder [X] failed`(3074，会话阶段被拒) / `Found H.264 encoder: Y [X]`(3448，成功)。
- ffmpeg 侧 verbose 日志（`min_log_level = verbose`）会打印 mfenc 的 MFT 枚举与 `setting output type` 全属性，是定位 MF 问题的唯一窗口。
- 脚本（本机 `D:\work\qoder01\mfwork\`）：`mf_probe.ps1`（-Elevated / -ExtraConfig）、`mf_mft_matrix.ps1`、`mf_intel_hidden.ps1`（两个改名实验，finally/catch 双保险改回）、`mf_pipeline.py`、`mf_dispatch.py`、`mf_run.ps1`（-Clipboard / -Dispatch / -Fetch）。

### 4.2 三组实验结果

1. **补丁生效性**（提权探测）：`Trying encoder [mediafoundation]` → `Creating encoder [h264_mf]` ×2 → `Encoder [mediafoundation] failed`。白名单这关已过。
2. **Intel MFT 接受集合**（临时改名 `C:\Windows\System32\nvEncMFTH264.dll` 逼 ffmpeg 选 Intel；测完改回）：D3D11 输入 + `hw_encoding=1 -rate_control cbr -scenario display_remoting` 下，**1080p60 profile 66/77/100、1080p30 High、720p60 High 全部成功**。即硬件与媒体类型无罪。
3. **跨 adapter**（临时改名 `C:\Program Files\Intel\Media SDK\mfx_mft_h264ve_64.dll` 逼 Sunshine 选 NVIDIA MFT，采集仍是 Intel）：`Could not open codec [h264_mf]: Function not implemented`。跨 adapter 在 Sunshine 内是死的。

### 4.3 失败窗口（Sunshine 进程内，verbose）

```
Trying encoder [mediafoundation]
Creating encoder [h264_mf]
activate MFT 0 / activate MFT 1
MFT name: 'Intel(R) Quick Sync Video H.264 Encoder MFT'   (VEN_8086, ff_MF_SA_D3D11_AWARE=1)
setting output type:
   MF_MT_FRAME_SIZE=1920x1080  MF_MT_FRAME_RATE=60:1  MF_MT_MPEG2_PROFILE=100
   MF_MT_AVG_BITRATE=1000000   MF_MT_INTERLACE_MODE=2  MF_MT_SUBTYPE=MFVideoFormat_H264
Error: could not set output type (MF_E_INVALIDMEDIATYPE)
Error: Could not open codec [h264_mf]: Generic error in an external library
Encoder [mediafoundation] failed
```

同一属性集合在 ffmpeg CLI（BtbN master）+ D3D11 输入下**成功**，且该次日志 `MF_MT_MPEG2_PROFILE=66`。

## 5. 未决问题与嫌疑清单

未决：同一 MFT、同一属性、同样带 `D3D11_CREATE_DEVICE_VIDEO_SUPPORT` 的设备，为何 CLI 成功而 Sunshine 失败。

**已排除**（各有一条反证）：profile 取值（实验 2 证明 Intel 收 High）；设备创建标志（`display_vram.cpp:866` 已带 VIDEO_SUPPORT）；COM 套间（全仓仅 `audio.cpp:289` 一处 CoInitializeEx 且为 MTA，Qt 托盘在探测之后才起）；ffmpeg 版本/mfenc 源码（pinned = FFmpeg `release/9.0` @ `bf1b838f`，经 build-deps `a1fe2841`；与 master 的 `libavcodec/mfenc.c` diff 仅 `#if CONFIG_D3D11VA` 编译开关）；MFT 选择（选中的就是 Intel）。

**继续挖的前提**：进程内调试（挂调试器或写独立 MF 客户端复刻 Sunshine 的 pre-state）。本机无 MSVC/MSYS2 工具链，做不了；无假设的 CI 重试没有靶子。另一条产出路径：把 §4 的证据链提给 LizardByte/Sunshine 上游 issue。

## 6. 资产清单

- 远端：fork `Ginsengyard/Sunshine` 分支 `mf-mft`；commits `f250175`(补丁)、`b034781`/`1aac10e1`(工作流)、`a2b97266`(删残留)、`bf156936`(探针标记文件 `docs/mf-write-probe.md`，可删)；run `36324333523`；artifact `10933734851`。
- 本机 `D:\work\qoder01\mfwork\`：`mfb\Sunshine\`（MF 版可运行包）、`build-Windows-AMD64.zip`(227MB)、`probe_result.txt`、`mf_window.txt`、`mft_matrix.txt`、`intel_hidden_probe.txt`、`mfenc_pinned.c`/`mfenc_master.c`、上述全部脚本、`Sunshine-master\`（fork 源码树）。

## 7. 将来 fork 继续时的上手清单

1. `git cherry-pick f250175`（或照 §2.1 手改一行）；复制 `ci-windows-mf.yml` 并保留 job 级 `permissions: contents: read`。
2. 触发构建：push 到实验分支，或带 Actions:write 的令牌 dispatch 自己的工作流（别 dispatch `ci.yml`）。
3. 下载 artifact 需登录态；解出 `*-lite.zip` 后按 §4.1 探测。
4. 想换 ffmpeg：`cmake/dependencies/ffmpeg.cmake` 支持 `FFMPEG_PREPARED_BINARIES` / `FFMPEG_ARCHIVE_NAME` 覆盖（预编译二进制来自 LizardByte/build-deps 的 release，tag = build-deps submodule 的 tag）。
5. 想改 profile：入口是 `src/video.cpp:2032` 的 `select_h264_profile()`（默认返回 `AV_PROFILE_H264_HIGH`）——但实验 2 已证明它与本故障无关，别在这上面花时间。
6. 任何「改名 System32/Program Files 下 DLL」的实验都必须 finally 改回，并先确认无进程占用。
