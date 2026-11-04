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
| OpenCore | 1.0.8（**DEBUG 版**，`OpenCore.efi` 体积 612 KB） |
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
-v keepsyms=1 agdpmod=pikera debug=0x100 -wegnoigpu e1000=0 swd_panic=1 npci=0x2000 -lilubetaall revcpu=1
```

| 参数 | 含义 |
| --- | --- |
| `-v` | 啰嗦模式（调试期保留） |
| `keepsyms=1` | panic 时保留符号 |
| `debug=0x100` | 禁止看门狗自动重启，便于看清 panic |
| `agdpmod=pikera` | 修复 RX 6000 系开机黑屏 |
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

`NVRAM → Delete` 中，`4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102` 下列了 `rtc-blacklist` / `revcpu` / `revcpuname` / `revpatch`，每次启动先删除再按 Add 重建，避免旧值残留。

**Misc / 安全**

| 项目 | 值 | 说明 |
| --- | --- | --- |
| `ScanPolicy` | `2687747` | 仅扫描指定类型分区 |
| `SecureBootModel` | `Disabled` | 未启用 Apple 安全启动 |
| `DmgLoading` | `Signed` | 仅加载已签名的 DMG |
| `Vault` | `Optional` | 未启用 vault 校验 |
| `BlacklistAppleUpdate` | `true` | 阻止 Apple 固件自动更新 |
| `ExposeSensitiveData` | `15` | NVRAM 敏感数据向 macOS 暴露 |
| `HideAuxiliary` | `true` | 隐藏辅助启动项 |
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
- **调试配置较重**：`-v` 啰嗦模式 + DEBUG 版 OpenCore + 文件日志持续写入 EFI 分区。日常使用可改用 RELEASE 版、去掉 `-v`、把 `Target` 降为 `0`。
- **`SMCRadeonGPU` 已于 2026-11-04 禁用**：`Kexts/` 里仍保留 0.3.3，需要 Radeon GPU 传感器数据时把 `Kernel → Add` 中该项 `Enabled` 改回 `true` 即可。
- **`Kernel → Emulate → Cpuid1Data` 设为 `0x000A0655`**（Raptor Lake 特征值），与 12900K 的 Alder Lake CPUID 不同，属沿用配置，如无特殊需求可置空。

## 故障恢复

| 现象 | 处理 |
| --- | --- |
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
