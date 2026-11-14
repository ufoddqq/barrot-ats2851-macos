# Barrot `33fa:0001` USB 蓝牙在 macOS 14 上的驱动记录

**结论：蓝牙音频（A2DP）可用；AirDrop 不可用。**

一颗 macOS 完全不认识的 Barrot / Actions USB 蓝牙棒（`33fa:0001`，产品名 `USB2.0-BT`），
在黑苹果（OpenCore）上的兼容性实测记录。

| 检查项 | 结果 |
|---|---|
| `system_profiler SPBluetoothDataType` | `Address: 04:7F:0E:xx:xx:xx`、`State: On` |
| Chipset | `THIRD_PARTY_DONGLE` |
| 固件 | `v137 c137`，`LMPv 10 (5.1)` |
| `launchctl print system/com.apple.bluetoothd` | `runs = 1`、`last exit code = (never exited)` |
| A2DP 播放 | 可用 —— SBC，44100 Hz，双声道，抖动缓冲 0 ms |
| AirDrop / Handoff / 通用剪贴板 | **不可用** —— 需要 BLE 广播，本控制器开不起来（见「限制」） |

测试环境：macOS **14.7.7（23H723）**、OpenCore、SMBIOS `iMac19,1`、台式机（**无 Wi-Fi 网卡**）。

本仓库是「兼容性报告 + 配置片段」，不包含任何自编译 kext。

---

## 一句话方案：三层，缺一不可

| # | 层 | 解决什么 |
|---|---|---|
| 1 | **BlueToolFixup** 2.7.2 | `bluetoothd` 的厂商白名单只放 `0x0A12 / 0x0A5C / 0x0B05`；补丁把这道门改成无条件跳转，并放开 chipset 检查，让控制器以 `THIRD_PARTY_DONGLE` 身份通过 |
| 2 | **NVRAM 三个键** | 没有它们，`bluetoothd` 根本不会扫描「外置 dongle」，而是去上电 SMBIOS 期望的内置控制器 → 约 16 秒超时 → 自杀 → 循环重启 |
| 3 | **IntelBTPatcher** 2.4.0 | 在第一条 `LE Set Scan Parameters` 之前补发 `LE Set Random Address`；否则 `bluetoothd` 死在 `STATUS 718`（HCI `0x12 Invalid HCI Command Parameters`） |

**只装第 1 层会比原来更糟**（16 秒一次崩溃循环）。顺序和完整性都要对。

---

## 一、原始症状

```
Bluetooth Controller:
    Address: NULL
    State: Off
    Chipset: BCM_4350C2
    Transport: UART
    Vendor ID: 0x004C (Apple)
```

外加 `bluetoothd` 每 15–16 秒重启一次，`/Library/Logs/DiagnosticReports/` 持续新增
`ExcUserFault_bluetoothd-*.ips`（`GUARD_TYPE_USER` / `os_fault_with_payload`）。

`BCM_4350C2 / UART / 0x004C` 是**幽灵设备**：它来自 SMBIOS 板型（`iMac19,1` 真机是
Broadcom 组合卡），不是 USB 总线上的东西。第三方 dongle 显示 `Transport: UART`
也是纯显示标签，实际走 USB。

此时日志里**一条 `[bm3_usb]` 都没有**——外置扫描压根没执行。

## 二、为什么会这样（因果链）

1. **厂商白名单**：`0x33FA` 不在允许列表 → 回落到内置控制器分支。
2. **补丁是逐 4 KB 页打的**：BlueToolFixup 挂在 `_cs_validate_page` 上，页首次换入时才 patch。
   扫描代码没执行 → 它的页没换入 → 厂商门那处 `74` 永远不会被改成 `EB`。
3. **BlueToolFixup ≥ 2.7.0 强制要求两个 NVRAM 变量**，否则不进「外置 dongle」分支。
4. **从未发过 `LE Set Random Address`**，于是带随机地址的 LE 扫描返回 `0x12`，`bluetoothd` 视为致命错误。

## 三、配置

### 3.1 Kext（`Kernel → Add`）

| 顺序 | 路径 | 版本 |
|---|---|---|
| 0 | `Lilu.kext` | 两者都依赖 |
| 1 | `BlueToolFixup.kext` | 2.7.2 |
| 2 | `IntelBTPatcher.kext` | 2.4.0（`com.zxystd.IntelBTPatcher`） |

`IntelBTPatcher` 名字带 Intel，但它的 hook **不判断厂商**，只匹配 HCI opcode `0x200B`，
所以对这颗 dongle 同样有效。OpenCore 条目见 `snippets/kernel-add.txt`。

### 3.2 NVRAM（GUID `7C436110-AB2A-4BBB-A880-FE41995C9F82`）

| 键 | 值 |
|---|---|
| `bluetoothExternalDongleFailed` | `<data>00</data>` |
| `bluetoothInternalControllerInfo` | `<data>` 14 个 0 字节 |
| `bluetoothHostControllerSwitchBehavior` | `always` |

同时在 `NVRAM → Delete` 里登记这三个键，保证每次启动都被重写成 `00`，
防止 `bluetoothd` 自己写 `01` 之后永久放弃。

可直接粘贴的 XML：`snippets/nvram-fragment.xml`。

### 3.3 调试（可选）

`boot-args` 加 `-btlfxdbg`，Lilu 会把每条命中的补丁位置写进内核日志；调通后建议去掉。

## 四、验收命令

```sh
system_profiler SPBluetoothDataType | head -15
launchctl print system/com.apple.bluetoothd | grep -E "runs =|last exit"
log show --last 3m --predicate 'eventMessage CONTAINS "bm3_usb"' | head
```

期望看到：

```
Address: 04:7F:0E:xx:xx:xx
State: On
Chipset: THIRD_PARTY_DONGLE
runs = 1
last exit code = (never exited)
[bm3_usb][Found_USB_Dongle] -- FoundBluetoothInterfaces() for deviceClass (224), idVendor = 0x33FA, idProduct = 0x0001 returned true
[bm3_usb][Found_USB_Dongle] -- setting gUseInternalController to FALSE
```

接着配对音箱，确认它成为默认输出且采样率 44100 Hz。

## 五、我们依次踩到的三个坑

| 现象 | 原因 | 解决 |
|---|---|---|
| `BCM_4350C2 / UART / NULL / Off`，16 秒崩溃一次，无 `[bm3_usb]` | 缺 NVRAM → 不扫外置 → 扫描页不换入 → 厂商门没打上 | 补三个 NVRAM 键 |
| dongle 被接管，但 `reason=718` 每 10–16 秒循环 | 扫描前没设 LE 随机地址 | `IntelBTPatcher` |
| 一切正常，但 iPhone 永远看不到这台 Mac | BLE 广播没启动（`LE_InitMultipleAdvertising` = `0x98`） | 本仓库**未解决**，见下 |

细节见 [`docs/troubleshooting.md`](docs/troubleshooting.md)，
原始日志见 [`docs/evidence.md`](docs/evidence.md)。

## 限制：AirDrop 为什么不行

AirDrop 有两个前提：

| 前提 | 我们的情况 |
|---|---|
| BLE **广播**（对方才能发现你） | ❌ `[Server.LE.Advertise] Stack is not started` |
| **AWDL**（真正的传输通道，Wi-Fi 直连） | ❌ 测试机没有 Wi-Fi 硬件 |

BLE **扫描**是正常的，所以蓝牙音频、LE 设备发现都不受影响。

上游已有相关进展：`acidanthera/BrcmPatchRAM` **PR #40**（`-btlfxadvsets`）
让第三方控制器走扩展广播路径，但截至目前其字节 pattern 只覆盖 macOS 26.x。
另外 AirDrop 还必须有 AWDL —— 即能被 macOS 驱动的 Wi-Fi 网卡
（原生 Broadcom，或 Intel + ForbiddenItlwm）。**没有 Wi-Fi 硬件的机器上，AirDrop 无解。**

## 致谢

| 项目 | 作者 | 许可 |
|---|---|---|
| [BrcmPatchRAM / BlueToolFixup](https://github.com/acidanthera/BrcmPatchRAM) | acidanthera | GPL-2.0 |
| [IntelBluetoothFirmware / IntelBTPatcher](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) | zxystd / OpenIntelWireless | GPL-2.0 |
| [Lilu](https://github.com/acidanthera/Lilu) | acidanthera | BSD-3-Clause |

本仓库不含任何修改过的二进制，只记录配置与实测行为。

## 许可

文档采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)；配置片段可随意使用，无需署名。
