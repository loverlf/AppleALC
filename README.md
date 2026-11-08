# AppleALC（含 Lenovo Tianjiao 16 / ALC274 定制分支）

> 本仓库是 [acidanthera/AppleALC](https://github.com/acidanthera/AppleALC) 的 Fork。
> 在完全保留官方内容的前提下，新增了 **layout-id 29**、一个 `kern_alc.cpp` 补丁和一条 GitHub Actions 编译流水线，
> 用于让 **联想天骄 16（ALO，子系统 0x17AA3757，Realtek ALC274，单 3.5mm 一体孔）** 的耳机、耳机麦、
> 底座麦克风静音键都能正常工作。
> 官方更新后，在 GitHub 点一次 **Sync fork** → Actions 自动编译出新的 `AppleALC.kext`，不需要本地 Xcode。

---

## 一、适配机型对比（ALC274）

| Layout ID | 来源 | 官方适配机型 | 在本机（天骄 16 ALO）上的表现 |
|---|---|---|---|
| **11** | 官方 | Realtek ALC274 通用 | 能出声，一体孔功能不完整 |
| **21** | 官方 | Dell Inspiron 27-7777 AIO 系列 | 不适合笔记本一体孔 |
| **28** | 官方 | Tongfang 17 准系统（Maingear Element 3 / Eluktronics Max 17） | 耳机有声但**人声发空、发远**（用户实测），静音键会触发"假插孔"把输入切走 |
| **35** | 官方 | Microsoft Surface Pro 7 | 不适用 |
| **39** | 官方 | 机械革命 UmiPro3（Tongfang GM5MG0Y） | 不适用 |
| **29** | **本仓库新增** | **联想天骄 16（ALO，17AA3757）** | **耳机双声道正常、耳机麦可用、静音键可用（本 README 的默认 layout）** |

> 官方 5 个 ID（11 / 21 / 28 / 35 / 39）在本仓库中**一字未改**，与本机型无关的机器仍可继续用它们。

---

## 二、这个 layout（29）修了本机的什么问题

本机是 **ALC274 + 单个 3.5mm 一体孔（耳机/耳机麦共用）**，底座上另有一个**麦克风静音实体键**（挂在 codec 的 `0x1A` 检测脚上，是自锁开关）。官方 layout 在本机存在以下问题，layout 29 + 补丁逐一解决：

1. **耳机人声空洞、像在山洞里**
   官方 layout 下本机耳机只有"单声道/发空"的听感。layout 29 修正了耳机脚 `0x21` 的 pin default（`0x02211020`）、EAPD 与 PathMap，耳机恢复双声道正常输出；并给输入链路配了 DSP（`SignalProcessing`），人声更实。

2. **耳机麦（一体孔麦克风）不出现 / 不跟随**
   官方 layout 下，插着耳机开机时输出会切到耳机，但输入仍停在内建麦，必须拔插一次才认耳机麦。
   补丁里的 **`-alcpres`（默认开）** 在 `0x19` 的 pin sense 查询上做委托，把耳机脚 `0x21` 的 presence 结果回写，开机即跟随。

3. **一体孔麦克风偏置被 AppleHDA 抹掉**
   AppleHDA 会把 `0x19` 的 pin ctl 写回 `0x02`（撤掉 VREF），导致耳机麦没偏置。
   补丁里的 **`-alcvref`（默认开）** 把 `0x19` 的 pin ctl 恒定为 `0x24`（IN + VREF 80%）。

4. **底座麦克风静音键完全无效**
   本机按这个键时，codec 的 `0x1A` 检测脚会被短接/断开（自锁，状态保持），但在 layout 29 下 AppleHDA 从不接管该脚（`unsol` 恒为 `0x09`），所以按键没有任何反应。
   补丁里的 **`-alckey`（默认开）**：
   - 把 `0x1A` 的通知挂到 AppleHDA 已经在用的中断标签（事件驱动，**不做轮询**，只留 5 秒一次的兜底自愈）；
   - 静音时不碰引脚、不撤偏置、不关输入，而是把**输入选择器** `0x22`、`0x24` 切到一条未接线的空脚（`SET_CONNECT_SEL`），等效"音量 0"，端口不会消失；
   - 再按一次恢复原选择器；开机首次强制"不静音"；
   - 静音态下拦截 AppleHDA 对 `0x22`/`0x24` 的 `0x701` 写入，避免被改回去。

5. **默认不静音、开关可选**
   三段钩子只在 **layout-id = 29** 下生效（内部用 `getAudioLayout` 判定），不会影响其它机型/其它 layout。
   需要关闭时用 boot-arg：`-alc274off`（全关）、`-alc274novref` / `-alc274nopres` / `-alc274nokey`（单关）。

---

## 三、使用方法

1. 把本仓库 Actions 产出的 `AppleALC.kext` 放进 `EFI/OC/Kexts/`，`config.plist` 的 `Kernel → Add` 里启用（顺序：Lilu → AppleALC）。
2. 指定 layout：boot-args 加 `alcid=29`，或在 `DeviceProperties` 里给 HDEF 写 `layout-id = 1D000000`（即 29）。
3. 可选：加 `-alcverbs=1` 打开 HDA verb 出口，方便用 `alc-verb` 现场调寄存器；钩子的诊断值（`alc274key-muted` / `-senses` 等）会写进 ioreg，用 `ioreg -l | grep alc274key` 查看。
4. 需要退回官方行为：去掉 `alcid=29` 换成 28 等官方 ID 即可，本仓库没有改动官方 layout。

---

## 四、官方更新后一键编译

1. 在本仓库页面点 **Sync fork → Update branch**（把 acidanthera 的新提交合进来）。
2. 进入 **Actions → Build ALC274 (Tianjiao 16) → Run workflow**（往上游同步时也会自动触发）。
3. 跑完后在该次运行的 **Artifacts** 里下载 `AppleALC-kext-ALC274`，解压即得签名好的 `AppleALC.kext`。

流水线做的事：拉 `MacKernelSDK` + 最新 `Lilu`（DEBUG，取头文件）→ `git apply patches/alc274-tianjiao16.patch` → `xcodebuild` 编译 Release → ad-hoc 签名 → 上传产物。

> 如果上游改动了 `AppleALC/kern_alc.cpp`，打补丁这一步会**明确失败**（不会产出半成品）。这时按下面"重新生成补丁"处理即可。

---

## 五、本地编译

```bash
git clone <本仓库地址> AppleALC && cd AppleALC
git clone --depth 1 https://github.com/acidanthera/MacKernelSDK.git MacKernelSDK
# 下载最新 Lilu DEBUG 版并解压为 ./Lilu.kext
git apply patches/alc274-tianjiao16.patch
xcodebuild -project AppleALC.xcodeproj -target AppleALC -configuration Release -arch x86_64 \
  CODE_SIGNING_ALLOWED=NO CODE_SIGN_IDENTITY=""
codesign --force --sign - --timestamp=none build/Release/AppleALC.kext
```

产物：`build/Release/AppleALC.kext`。

---

## 六、重新生成补丁（上游改了 kern_alc.cpp）

```bash
git checkout master && git pull            # 或 Sync fork
git apply --3way patches/alc274-tianjiao16.patch   # 冲突就手动合并
# 合并完成后，重新导出补丁
git diff AppleALC/kern_alc.cpp > patches/alc274-tianjiao16.patch
```

改完记得把 `patches/` 一起提交，Actions 会用它。

---

## 七、资源文件说明

- `Resources/ALC274/layout29.xml` / `Platforms29.xml`：本机 layout 与 PathMap（由验证通过的 72 号配置重编号而来）。
- `Resources/ALC274/Info.plist`：注册 29，官方 11/21/28/35/39 原样保留。
- `Resources/PinConfigs.kext/Contents/Info.plist`：29 的 96 字节 ConfigData（与验证版一致）。
- `.zlib` / `.md5` 是构建时由 `ResourceConverter/generate.sh` 自动生成的，不需要手工维护。

---

## 八、致谢与免责

- 资源与补丁基于 [acidanthera/AppleALC](https://github.com/acidanthera/AppleALC)（vit9696 等）与 [acidanthera/Lilu](https://github.com/acidanthera/Lilu)。
- 本分支仅针对 **联想天骄 16 ALO（ALC274, 17AA3757）** 验证；换机型请自行验证或改用官方 layout。
- 修改 EFI 有风险，请先备份可启动的 EFI。

---

AppleALC
========

[![Build Status](https://github.com/acidanthera/AppleALC/actions/workflows/main.yml/badge.svg?branch=master)](https://github.com/acidanthera/AppleALC/actions) [![Scan Status](https://scan.coverity.com/projects/16166/badge.svg?flat=1)](https://scan.coverity.com/projects/16166)

An open source kernel extension enabling native macOS HD audio for not officially supported codecs without any filesystem modifications. AppleALCU can be used for systems with digital-only audio.

English (Current)  
[简体中文](https://github.com/acidanthera/AppleALC/blob/master/README_CN.md)  

#### Features
- Digital and analog audio support starting from the OS installation
- Recovery HD/macOS Installer audio support
- Automated codec detection
- Unsupported audio controller enabling (internal and external)
- Arbitrary kext patching
- Custom platform/layout injection
- Works with SIP / El Capitan+
- Currently compatible with 10.4-26*

\* _NOTE_: macOS 26 dropped AppleHDA.kext in DP2. AppleALC functioning on macOS 26 may require additional actions if AppleHDA.kext is necessary.

#### Credits
- [Apple](https://www.apple.com) for macOS  
- [Onyx The Black Cat](https://github.com/gdbinit/onyx-the-black-cat) by [fG!](https://reverse.put.as) for the base of the kernel patcher
- [capstone](https://github.com/aquynh/capstone) by [Nguyen Anh Quynh](https://github.com/aquynh) for the disassembler module
- [toleda](https://github.com/toleda), [Mirone](https://github.com/Mirone) and certain others for audio patches and layouts
- [Pike R. Alpha](https://github.com/Piker-Alpha) for [lzvn](https://github.com/Piker-Alpha/LZVN) decompression and certain HDMI patches
- [07151129](https://github.com/07151129) for some code parts and suggestions
- [roddy20](https://github.com/roddy20) for training and research dumps to a patching of codecs
- [vit9696](https://github.com/vit9696) for writing the software and maintaining it
- [Andrey1970AppleLife](https://github.com/Andrey1970AppleLife), [vandroiy2013](https://github.com/vandroiy2013) for maintaining the codec database

#### Installation
The minimal instruction is available on the [wiki](https://github.com/acidanthera/AppleALC/wiki).  
The prebuilt binaries are available on [releases](https://github.com/acidanthera/AppleALC/releases) page.

#### Contribution
To support more audio codecs in the binary packages you are asked to submit your configurations. Please read the [wiki](https://github.com/acidanthera/AppleALC/wiki) for more details. For the contributors with programming skills the headers are filled with AppleDOC comments.

#### Support and discussion
[InsanelyMac topic](http://www.insanelymac.com/forum/topic/311293-applealc-—-dynamic-applehda-patching/) in English  
[AppleLife topic](https://applelife.ru/threads/applealc-dinamicheskij-patching-applehda.1171672/) in Russian

#### Donations
Writing and supporting code is fun but it takes time. If you want to thank the author for his work consider contributing, bugreporting, or providing the support to other users.
