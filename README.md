# ASUS PRIME Z790-P WIFI D5 · i9-12900K · RX 6600 — OpenCore EFI

macOS **Sequoia 15.7.9 (24G830)** · OpenCore **1.0.8** · SMBIOS **MacPro7,1**

本仓库为实机在用的 EFI，配置项与 `EFI/OC/config.plist` 一一对应，非模板拼装。

> ### ⚠️ 使用前必读：SMBIOS 已脱敏
>
> 仓库内 `EFI/OC/config.plist` 的 `SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM` 全部替换为占位值：
>
> ```
> SystemSerialNumber : C02XXXXXXXXX
> MLB                : C02XXXXXXXXXXXXXX
> SystemUUID         : 00000000-0000-0000-0000-000000000000
> ROM                : 00 00 00 00 00 00
> ```
>
> **直接拿这份 EFI 启动可以进系统，但 iCloud / iMessage / FaceTime / App Store 会不可用**（Apple 服务需要有效且未被占用的 SMBIOS）。
>
> 必须自己生成一套：用 [OCAuxiliaryTools](https://github.com/ic005k/OCAuxiliaryTools) 或 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)，机型选 **MacPro7,1**，生成后填回 `PlatformInfo → Generic` 的四个字段。**不要**直接沿用别人仓库里的序列号——重复的序列号会导致双方都被 Apple 拒绝。

---

## 硬件

| 部件 | 型号 |
| --- | --- |
| 主板 | ASUS PRIME Z790-P WIFI D5 |
| CPU | Intel Core i9-12900K（16C/24T，Alder Lake） |
| 内存 | Kingston FURY DDR5 32 GB ×2 = 64 GB（KF552C40-32，SPD 报 4800 MT/s） |
| 独显 | Sapphire AMD Radeon RX 6600 |
| 核显 | UHD 770 —— 不驱动（`-wegnoigpu`） |
| 存储 | NVMe SSD（启用 NVMeFix） |
| 有线网卡 | Realtek RTL8125 2.5GbE（另配 I225 / I219 / Atheros 驱动以兼容不同网卡） |
| 无线 / 蓝牙 | 板载 Intel（蓝牙走 IntelBluetoothFirmware；Wi-Fi 驱动见下方「已知情况」） |
| 声卡 | 板载，`layout-id = 11` |

## 软件版本

| 项目 | 值 |
| --- | --- |
| macOS | 15.7.9（Build 24G830） |
| OpenCore | 1.0.8（**RELEASE 版**，`OpenCore.efi` 626,688 B ≈ 612 KB，与官方发布包 MD5 一致） |
| SMBIOS 机型 | MacPro7,1 |
| 引导界面 | OpenCanopy 图形界面，主题 `chris1111\Flavours-OSX26`（`PickerAttributes = 145`，启用 content flavour） |
| 引导超时 | 3 秒，显示辅助项（`HideAuxiliary = false`） |
| SIP | 部分放行，`csr-active-config = 0x0803` |
| SecureBootModel | Disabled（未启用 Apple 安全启动） |
| 处理器显示名 | `Intel(R) Core(TM) i9-12900K`（`ProcessorType = 3841` + RestrictEvents，见「CPU 名称显示」） |

## 目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
├── APPLE/                      # Apple 固件更新器（MultiUpdater / DpUtil），非必需
└── OC/
    ├── OpenCore.efi
    ├── config.plist            # 唯一配置，OpenCore.efi 实际加载的就是它
    ├── ACPI/                   # 7 个 SSDT
    ├── Kexts/                  # 34 个 kext（19 启用 / 5 禁用 / 10 未被引用）
    ├── Drivers/                # 54 个 .efi 驱动（config 中只登记 8 个）
    ├── Tools/                  # CleanNvram / OpenShell / ResetSystem
    └── Resources/              # 主题图标（Acidanthera 三套 + chris1111\Flavours-OSX26）、字体、开机音
```

## ACPI（7 个，全部启用）

| 文件 | 作用 |
| --- | --- |
| `SSDT-RTCAWAC.aml` | 屏蔽 AWAC 实时时钟，避免与 RTC 冲突 |
| `SSDT-EC-USBX.aml` | 伪造 EC 设备 + USB 供电属性 |
| `SSDT-PLUG-ALT.aml` | 将现代 CPU 设备重定义为 legacy Processor 并给首个 CPU 注入 `plugin-type = 1` |
| `SSDT-RHUB.aml` | 重置 USB 端口，修复 XHCI 枚举 |
| `SSDT-SBUS-MCHC.aml` | 补全 SBUS / MCHC，保证 AppleSMBus 正常 |
| `SSDT-GPRW.aml` | 配合 GPRW→XPRW 改名，修复睡眠唤醒（改名补丁当前未启用，见「已知情况」） |
| `FixShutdown-USB-SSDT.aml` | 修复关机后 USB 供电未断的问题 |

## Kexts（19 个启用）

**核心**

| Kext | 版本 |
| --- | --- |
| Lilu | 1.7.2 |
| VirtualSMC | 1.3.8 |
| SMCProcessor | 1.3.8 |
| SMCSuperIO | 1.3.8 |
| WhateverGreen | 1.7.1 |

**音频 / CPU 电源**

| Kext | 版本 | 说明 |
| --- | --- | --- |
| AppleALC | 1.9.8 | 配合 `layout-id = 11` |
| CPUFriend | 1.3.0 | 自定义 CPU 电源管理 |
| CPUFriendDataProvider | 1.0.0 | CPUFriend 的数据源 |
| RestrictEvents | 1.1.6 | 配合 `revcpu` / `revcpuname` 显示正确 CPU 名称 |

**USB**

| Kext | 版本 | 说明 |
| --- | --- | --- |
| USBToolBox | 1.1.1 | USB 端口映射框架 |
| UTBMap | 1.1 | 本机端口映射（`UTBMap.kext`） |
| Z790-XHCI-unsupported | 0.9.2 | 放行 Z790 XHCI 控制器 |

**网络**

| Kext | 版本 | 对应网卡 |
| --- | --- | --- |
| LucyRTL8125Ethernet | 1.1.0 | Realtek RTL8125 2.5GbE |
| AppleIGC | 1.4d1 | Intel I225-V |
| IntelMausiEthernet | 2.3.0 | Intel I219 |
| AtherosE2200Ethernet | 2.2.2 | Atheros Killer E2200 系列 |

**蓝牙 / 存储**

| Kext | 版本 | 说明 |
| --- | --- | --- |
| IntelBluetoothFirmware | 2.4.0 | Intel 蓝牙固件 |
| BlueToolFixup | 2.7.2 | macOS 12+ 蓝牙修复 |
| NVMeFix | 1.1.3 | NVMe 电源管理与兼容性 |

### 未启用的 kext（15 个）

`Kexts/` 目录共 34 个 kext：19 个启用、5 个在 config 中显式 `Enabled = false`、10 个完全未被 config 引用（`IO80211FamilyLegacy` 另带 1 个同样禁用的嵌套插件 `AirPortBrcmNIC`）。

| Kext | 状态 | 用途 |
| --- | --- | --- |
| `AirportItlwm-Sonoma.kext` | 未引用 | Intel 无线网卡驱动（Sonoma 分支）。**当前 config 未加载** |
| `AirportBrcmFixup.kext` | 未引用 | Broadcom 无线卡补丁 |
| `BrcmFirmwareData` / `BrcmPatchRAM3` / `BrcmBluetoothInjector` | 未引用 | Broadcom 蓝牙固件上传 |
| `BluetoothFileExchange.kext` | 未引用 | 蓝牙文件交换 |
| `CtlnaAHCIPort.kext` | 未引用 | 第三方 SATA 控制器 |
| `UTBMap-2port front.kext` | 未引用 | 前面板 2 口的 USB 映射备选 |
| `USBWakeFixup.kext` | 未引用 | USB 唤醒修复 |
| `IntelBTPatcher-2.4.kext` | 未引用 | 与 `IntelBTPatcher.kext` 重复的备份副本（两者都未加载） |
| `SMCRadeonGPU.kext` | `false` | Radeon GPU 传感器，2026-11-04 起禁用 |
| `CpuTopologyRebuild.kext` | `false` | 大小核拓扑重建 |
| `IntelBTPatcher.kext` | `false` | 2.4.0 与 15.7.9 不兼容，已禁用（见「已知情况」） |
| `IOSkywalkFamily` / `IO80211FamilyLegacy`（+ `AirPortBrcmNIC`） | `false` | Broadcom 网卡降级方案 |

## UEFI 驱动（54 个文件，启用 8 个）

| 驱动 | LoadEarly | 作用 |
| --- | --- | --- |
| `OpenVariableRuntimeDxe.efi` | ✅ | 变量运行时服务（配合 `Misc → Security → Vault = Optional`） |
| `OpenRuntime.efi` | ✅ | OpenCore 运行时服务，必需 |
| `HfsPlus.efi` | — | 读取 HFS+ 分区 |
| `OpenCanopy.efi` | — | 图形引导界面 |
| `ResetNvramEntry.efi` | — | 引导菜单里的 Reset NVRAM 项 |
| `AudioDxe.efi` | — | 开机音 |
| `CrScreenshotDxe.efi` | — | 引导界面截图（F10） |
| `ToggleSipEntry.efi` | — | 引导菜单里切换 SIP |

`Drivers/` 里其余 46 个 .efi（Virtio 系列、各类网络栈、Ext4Dxe、OpenLinuxBoot 等）为 OpenCore 发布包原样保留，**均未在 config 中加载**。

## 关键配置

**boot-args**

```
-v keepsyms=1 agdpmod=pikera gfxrst=4 debug=0x100 -wegnoigpu e1000=0 swd_panic=1 npci=0x2000 -lilubetaall revcpu=1
```

| 参数 | 含义 |
| --- | --- |
| `-v` | 啰嗦模式（调试期保留） |
| `keepsyms=1` | panic 时保留符号 |
| `debug=0x100` | 禁止看门狗自动重启，便于看清 panic |
| `agdpmod=pikera` | 修复 RX 6000 系开机黑屏 |
| `gfxrst=4` | 关闭第二启动阶段的 framebuffer 初始化交互（WhateverGreen），用于修复「到登录界面黑屏」——见「间歇性开机黑屏」 |
| `-wegnoigpu` | 禁用核显输出 |
| `e1000=0` | 避免 Intel 网卡驱动抢占 |
| `swd_panic=1` | 看门狗触发时输出 panic |
| `npci=0x2000` | PCI 配置空间兼容 |
| `-lilubetaall` | 让 Lilu 及插件在非原生支持的 macOS 版本上加载 |
| `revcpu=1` | 允许 RestrictEvents 改写 CPU 品牌串（名称本身走 NVRAM，见下节） |

**DeviceProperties**

| 设备路径 | 内容 |
| --- | --- |
| `PciRoot(0x0)/Pci(0x1F,0x3)` | `layout-id = 0B000000`（即 11） |
| `PciRoot(0x0)/Pci(0x14,0x0)` | USB 控制器标识（ASUS PRIME Z790-P） |
| `PciRoot(0x0)/Pci(0x1,0x0)/Pci(0x0,0x0)/Pci(0x0,0x0)/Pci(0x0,0x1)` | 显卡 HDMI 音频控制器（`model = Audio Device Controller (HDMI)`） |

**NVRAM**

| GUID | 键 | 值 |
| --- | --- | --- |
| `4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102` | `revcpu` | `1`（Number） |
| `4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102` | `revcpuname` | `Intel(R) Core(TM) i9-12900K`（String） |
| `7C436110-AB2A-4BBB-A880-FE41995C9F82` | `boot-args` | 见上 |
| `7C436110-AB2A-4BBB-A880-FE41995C9F82` | `csr-active-config` | `03080000`（即 `0x0803`） |
| `7C436110-AB2A-4BBB-A880-FE41995C9F82` | `prev-lang:kbd` | `en-US:0` |
| `7C436110-AB2A-4BBB-A880-FE41995C9F82` | `run-efi-updater` | `No` |

`NVRAM → Delete` 分三段列出「每次启动先删除、再按 Add 重建」的键：

- `4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102`：`rtc-blacklist` / `revcpu` / `revcpuname` / `revpatch`，避免旧值残留。
- `7C436110-AB2A-4BBB-A880-FE41995C9F82`：**只剩 `ForceDisplayRotationInEFI`**。原本还列有 `boot-args`，已于 2026-11-05 移除——`Add` 每次启动本来就重写同一串值，这条 Delete 只会把「NVRAM 写入失败」放大成「参数被删空」（见「间歇性开机黑屏」）。
- `4D1EDE05-38C7-4A6A-9CC6-4BCCA8B38C14`：`DefaultBackgroundColor`。

**Misc / 安全**

| 项目 | 值 | 说明 |
| --- | --- | --- |
| `ScanPolicy` | `2687747` | 仅扫描指定类型分区 |
| `SecureBootModel` | `Disabled` | 未启用 Apple 安全启动 |
| `DmgLoading` | `Signed` | 仅加载已签名的 DMG |
| `Vault` | `Optional` | 未启用 vault 校验 |
| `BlacklistAppleUpdate` | `true` | 阻止 Apple 固件自动更新 |
| `ExposeSensitiveData` | `15` | NVRAM 敏感数据向 macOS 暴露 |
| `HideAuxiliary` | `false` | 显示辅助启动项 |
| `PickerMode` | `External` | 使用 OpenCanopy 图形界面 |

**调试**

`AppleDebug = true`、`ApplePanic = true`、`DisableWatchDog = true`、`Debug → Target = 67`（`0x43` = 控制台 + DataHub + 文件日志）。日志写入 EFI 分区根目录的 `opencore-YYYY-MM-DD-HHMMSS.txt`。

## CPU 名称显示

macOS 对未被原生支持的 CPU 一律显示「未知」。本机通过三处配合把「关于本机」的处理器一行改成 `Intel(R) Core(TM) i9-12900K`（已在 15.7.9 实机验证）：

| 位置 | 设置 |
| --- | --- |
| `PlatformInfo → Generic → ProcessorType` | `3841`（`0x0F01`） |
| `Kernel → Add` | `RestrictEvents.kext` 1.1.6，`Enabled = true`，排在 `Lilu.kext` 之后 |
| `NVRAM → Add → 4D1FDA02-…` | `revcpu = 1`（Number）、`revcpuname = Intel(R) Core(TM) i9-12900K`（String） |

**为什么 `ProcessorType` 必须是 3841**

`revcpuname` 并不是把名字直接写进系统，它的做法是在 dyld 共享缓存的字符串表里找到一条**已存在**的 CPU 名称串再替换掉。而这条串能否命中，取决于 `ProcessorType` + 核心数的查表结果：

- 8 核及以上 → `0x0F01` = **3841**（Xeon W 族，表里有 24 核条目）
- 1 / 2 / 4 / 6 核 → `0x0601` = **1537**

`ProcessorType = 0` 时按核数查表直接落空，显示的仍是「未知」，RestrictEvents 也就没有可替换的字符串，`revcpuname` 写了不生效。这正是本机 2026-11-04 之前一直显示「未知」的原因。

**为什么名称走 NVRAM 而不是 boot-args**

`boot-args` 以空格分词，`revcpuname` 的值本身含空格，写进去会被截断。RestrictEvents 支持同名 NVRAM 变量，效果与 boot-args 等价、优先级更低。当前 `boot-args` 里也有 `revcpu=1`，与 NVRAM 的 `revcpu` 重复但值相同，不冲突。

**影响范围**

`3841` 是 Xeon W 族的 Apple CPU identifier，属**显示层伪装**：不改 CPUID、不改核心/线程拓扑、不影响性能。系统报告的物理/逻辑核心数仍按真实 16C/24T 给出。取值为 `0` 即可完全还原。

## 使用方法

1. **先生成自己的 SMBIOS**（见顶部警告），填回 `PlatformInfo → Generic`。
2. 把 `EFI/` 整个目录拷到 ESP 分区的 `EFI/` 下（与 Windows 的 `Microsoft/` 同级共存即可）。
3. BIOS 中关闭 Secure Boot、关闭 Fast Boot，CSM 关闭（UEFI Only）。
4. 首次启动若异常，在引导菜单选 **CleanNvram** 或 **ResetSystem**。

## 已知情况

- **IntelBTPatcher 已禁用**：2.4.0（官方最新）在 macOS 15.7.9 上与 IOBluetoothFamily 不兼容——蓝牙控制器枚举后 bluetoothd 进入 `EXC_GUARD` 崩溃循环，登录后整机冻结。当前保留 `IntelBluetoothFirmware 2.4.0` + `BlueToolFixup 2.7.2`，蓝牙功能待实测；如系统稳定但蓝牙不可用，再评估新版 IntelBTPatcher。
- **`SSDT-GPRW.aml` 已加载，但配套的 ACPI 改名补丁（`GPRW` → `XPRW`）在 config 中 `Enabled = false`**。睡眠唤醒若异常，先检查这里。`ACPI → Patch` 三条（`_PTS`→`ZPTS`、`ADBG`→`XDBG`、`GPRW`→`XPRW`）**全部未启用**。
- **Wi-Fi**：`AirportItlwm-Sonoma.kext` 存在于 `Kexts/` 但**没有**写进 config，当前 EFI 不加载任何 Intel 无线驱动。需要用 Wi-Fi 时自行添加该 kext 条目。
- **以太网驱动有冗余**：同时启用了 RTL8125 / I225 / I219 / Atheros 四个驱动，属于兼容性覆盖，可按实际网卡精简。
- **调试配置较重**：`-v` 啰嗦模式 + 文件日志持续写入 EFI 分区。引导器本身已是 RELEASE 版（因此日志中**没有**任何 `OC:` 行，只剩 `AAPL:` 那一段），日常使用可去掉 `-v`、把 `Target` 降为 `0`。
- **`SMCRadeonGPU` 已于 2026-11-04 禁用**：`Kexts/` 里仍保留 0.3.3，需要 Radeon GPU 传感器数据时把 `Kernel → Add` 中该项 `Enabled` 改回 `true` 即可。
- **`Kernel → Emulate → Cpuid1Data` 设为 `0x000A0655`**（Raptor Lake 特征值），与 12900K 的 Alder Lake CPUID 不同，属沿用配置，如无特殊需求可置空。

## 间歇性开机黑屏（未解决）

**现象**（2026-11 起，间歇性；冷启动与热重启均出现过）：

1. 开机到登录界面时显示器失联、**黑屏**。判断依据：盲打回车能听到 macOS 的密码错误提示音——说明 `loginwindow` 已在运行、键盘与音频子系统正常，唯独显示输出丢失。
2. 偶尔 `-v` 白字刷过之后直接黑屏，不再出画面。
3. **2026-11-05 追加观测**：应用 `gfxrst=4` 并重启后**仍偶发**。
4. **2026-11-05 关键观测**：**只接 HD270（2560×1440@59.97Hz）一台显示器时，同样会偶发黑屏、进不去系统。** 单屏也犯 ⇒ 与「双屏把 display pipe 占满」无关，故障不依赖显示器数量。这一条把排查方向从「多屏资源竞争」拉回到「单路 framebuffer / connector 初始化」。

**已排除**（均经实机核对，非推测）：

| 排除项 | 依据 |
| --- | --- |
| 内核 panic / 驱动崩溃 | `/Library/Logs/DiagnosticReports` 与 `~/Library/Logs/DiagnosticReports` 下**无任何 `.panic` 文件**，NVRAM 无 `panic-info` |
| 引导器故障 | ESP 根目录 6 份 `opencore-2026-11-04-*.txt` 全部正常结束于 `#[EB\|LOG:EXITBS:START]`，`BootKernelExtensions.kc` 五次校验值一致 |
| 引导组件损坏或混版本 | `BOOTx64.efi` + `OpenCore.efi` + 54 个 `Drivers/*.efi` 与官方 1.0.8 RELEASE 包 **MD5 全部一致** |
| 配置语法 | 官方 `ocvalidate`（1.0.8）：`No issues found.` |
| DisplayPort 链路训练 / 热插拔检测 | **黑屏时拔插 DP 线无任何反应**，画面不恢复 |
| 双屏占满 4 条 display pipe | **只接 HD270 单屏时同样偶发黑屏、进不去系统**，故障与显示器数量无关 |
| 第二阶段 framebuffer 交接（`gfxrst`） | 已应用 `gfxrst=4` 后仍偶发，单靠此项无法解释 |
| 双 ESP 引导冲突 | `disk1s1`（Windows ESP）无 `EFI/OC`，BIOS 不可能从另一套 OpenCore 启动 |

**硬件背景**（2026-11-05 实读 `system_profiler` / `ioreg` / EDID）：

| 项 | 实况 |
| --- | --- |
| dGPU | RX 6600（Navi 23，`1002:73ff`），Slot-2，Metal 3，8 GB |
| iGPU | `8086:4660`（UHD 770）存在但 `No Kext Loaded`，macOS 15 无驱动，属正常 |
| **未知第二块 VGA** | **`4c54:5006`（class `0x030000`，Slot-1 x16，另带音频 function `4c54:5010`），`No Kext Loaded`** |
| HD270 | 厂商 **HKC**、产品码 `0x2700`、**2015 年**、EDID 1.4、60×34 cm；原生 DTD **2560×1440@59.97Hz**（pclk 241.5 MHz，htotal 2720 × vtotal 1481）；系统按 **30-bit ARGB2101010** 输出 |
| DELL U2725QE | 厂商 DEL、产品码 `0x4362`、**2026 年**、EDID 1.4；当前 3840×2160@60Hz（界面 HiDPI 缩放到 1920×1080）；30-bit |
| display pipe | Navi 23 共 **4 条**；双屏时被占满，但单屏也犯病 ⇒ 非瓶颈因素 |

> 那台 `4c54:5006` 是独立于 RX 6600 与 iGPU 的第三个 VGA 设备，`system_profiler` 里显示为 Slot-1 的 `No Kext Loaded`。它没有驱动、不输出画面，但**会参与 macOS 的显示设备枚举**；若某次开机它被选为 primary display，就可能出现「系统在跑、屏上没画面」。这条属低概率怀疑项，需在下次黑屏时用 `ioreg` 复核。

**根因假设**（按可能性排序，随观测进展调整）：

1. **单路 connector 的 modeset 随机失败（与显示器数量无关）**。单屏独显也犯病，说明不是 pipe/资源耗尽，而是开机窗口内某一路 connector 的初始化偶尔没完成。HD270 是「双屏」与「单屏」两套拓扑中**唯一都出现**的显示器，因此它是最值得单独验证的变量。
2. **HD270 自身的 EDID / 时序 / 色深**。HKC 2015 年 QHD 屏，EDID 1.4 只提供一条 DTD（2560×1440@59.97，pclk 241.5 MHz），而系统按 **30-bit ARGB2101010** 输出——30bpp 比 24bpp 多 25% 链路带宽，若 DP 链路协商落在 HBR（8.64 Gbps 有效）而非 HBR2，余量会明显变薄。2015 年的低价屏被误判为 10-bit 是常见情况。
3. **`Booter → Quirks → DevirtualiseMmio = true` 配空的 `MmioWhitelist`**。OpenCore 1.0.8 官方文档明言该组合「without additional measures, it is not necessarily compatible with the target board」，且「on certain firmware, a list of addresses that need virtual addresses for proper NVRAM and hibernation functionality may be required」。本机 **18:08:03 那份日志即为物证**：那次启动 `boot-args` 与 `csr-active-config` 全部丢失（`MBA:NV` 为空、`MBA:OUT` 空串、`CSR:IN 0x00000040`、`BRD:SMBIOS` 回落、3 条 `GFWF` 读取失败），等于以「无 `agdpmod=pikera`、无 `-wegnoigpu`」的状态裸奔——必然黑屏。
4. **`NVRAM → Delete` 中的 `boot-args`**：把「写入失败」放大成「参数全丢」。`Add` 每次启动本来就重写同一串值，这条 Delete 没有存在必要。**已于 2026-11-05 移除。**
5. **未知第二块 VGA（`4c54:5006`，Slot-1，`No Kext Loaded`）干扰 primary display 判定**：低概率，但代价极低可验证。

> **已降级的假设**：
> - ~~4 条 display pipe 分配竞态~~ —— 只接 HD270 单屏时同样犯病，与显示器数量无关。
> - ~~第二阶段 framebuffer 交接失败~~ —— 应用 `gfxrst=4` 后仍偶发，该层不足以单独解释。

**待验证的实验与修改**（逐项单独验证，A/B 两个零成本对照优先）：

| 顺序 | 实验 / 修改 | 状态 | 风险 | 目的 |
| --- | --- | --- | --- | --- |
| **A** | **只接 HD270 单屏**，开关机 8～10 次 | **✅ 已观测：仍偶发黑屏、进不去系统** | 零 | 确认故障与显示器数量无关 |
| **B** | **只接 DELL 单屏**（拔掉 HD270），同样 8～10 次 | **下一步（首选）** | 零 | 区分「HD270 专属」还是「全局」：不犯 ⇒ 锁定 HD270；仍犯 ⇒ 与屏无关，直接做 E |
| C | HD270 换到 RX 6600 的另一路 DP，或改用 HDMI，并换线 | 待定 | 零 | 排除 connector 与线材 |
| D | 降低 HD270 链路负载：分辨率降 1920×1080@60，或 `boot-args` 加 `-rad24` 强制 24-bit 色深 | 待定 | 低 | 验证 30-bit（30bpp）/ 链路带宽是否为诱因 |
| E | `DevirtualiseMmio` 改 `false` | 待定 | 中 | 验证 NVRAM 写入是否转稳 |
| F | `boot-args` 追加 `gfxrst=4` | **✅ 2026-11-05 已应用（无效）** | 极低 | WhateverGreen：*Disable framebuffer init interaction during 2nd boot stage* |
| G | 从 `NVRAM → Delete` 移除 `boot-args` | **✅ 2026-11-05 已应用** | 零 | 关闭「参数全丢」窗口 |

`gfxrst=4` 无效后仍可试 `gfxrst=1`（*Prefer drawing Apple logo at 2nd boot stage instead of framebuffer copying*）；`-rad24` 已并入 D。

**应急处理**：

| 场景 | 操作 |
| --- | --- |
| 黑屏但系统在跑 | 盲打密码回车；登录后仍无画面，则 SSH 进去执行 `sudo launchctl kickstart -k system/com.apple.WindowServer` |
| 怀疑是 `gfxrst=4` 引起的 | 回滚到改前备份：`cp /Volumes/EFI/EFI/OC/config.plist.bak-20261105-040421 /Volumes/EFI/EFI/OC/config.plist`（该备份为「已移除 Delete、未加 `gfxrst`」状态；`config.plist.bak-20261105-040314` 是最初原始状态） |
| 现场取证 | 记下 `-v` 最后一行停在何处——`EXITBS:START` 之后的内核输出**不会**写进 OpenCore 日志，事后翻日志看不到 |

## 故障恢复

| 现象 | 处理 |
| --- | --- |
| 开机到登录界面黑屏，但能听到密码错误提示音 | 系统已启动，仅显示输出丢失——见「间歇性开机黑屏」一节 |
| 卡 `OCB: StartImage failed` / 引导项错乱 | 引导菜单 → **CleanNvram** |
| 反复重启、进不去系统 | 引导菜单 → **ResetSystem**，或进 BIOS 清 CMOS |
| 需要临时关闭 SIP | 引导菜单 → **ToggleSipEntry** |
| 需要抓取日志 | 按 F10 截图（CrScreenshotDxe），或到 EFI 根目录找 `opencore-*.txt` |

## 参考

- 原始配置参考视频：https://www.youtube.com/watch?v=8SD0_6MaToo
- OpenCore 官方文档与配置手册：https://github.com/acidanthera/OpenCorePkg
- Dortania 桌面平台指南：https://dortania.github.io/OpenCore-Install-Guide/

## 许可

GPL-3.0，见 [LICENSE](LICENSE)。

---

*SMBIOS 信息已脱敏。本仓库只提供一套可参考的实机配置，硬件相同不代表可直接套用——请按自身硬件重新生成 SMBIOS 并自行验证。*
