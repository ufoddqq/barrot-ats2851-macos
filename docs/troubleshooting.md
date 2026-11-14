# Troubleshooting

A symptom → cause → fix table for this dongle, plus the diagnostics that produced it.

---

## Quick diagnosis

```sh
# 1. What does the dongle report?
ioreg -p IOUSB -l -w0 | grep -A3 -i "USB2.0-BT"

# 2. Does bluetoothd see it as an external dongle?
log show --last 5m --predicate 'eventMessage CONTAINS "bm3_usb"'

# 3. Is it looping?
launchctl print system/com.apple.bluetoothd | grep -E "runs =|last exit"
ls -lt /Library/Logs/DiagnosticReports | grep -i bluetoothd | head -3

# 4. What does macOS think it is?
system_profiler SPBluetoothDataType | head -15
```

| `bm3_usb` present? | `State` | Meaning |
|---|---|---|
| no | `Off`, chipset `BCM_4350C2` | external-dongle scan never ran → NVRAM keys missing (or BlueToolFixup not loaded) |
| yes | `Off`, `reason=718` in logs | scan started but LE random address was never set → `IntelBTPatcher` missing |
| yes | `On`, `THIRD_PARTY_DONGLE` | working |
| yes | `On`, but no AirDrop | BLE advertising off → not covered by this repo |

---

## Symptom table

### 1. `BCM_4350C2` / `Transport: UART` / `Vendor ID: 0x004C`

**Not your dongle.** SMBIOS says the board has a Broadcom combo card (`iMac19,1`), so
`bluetoothd` tries to power on that internal controller, fails after ~15 s, and kills itself.
`Transport: UART` is a cosmetic label that Sonoma prints for third-party dongles; the actual
transport is USB.

Fix: the three NVRAM keys. Nothing else changes this.

### 2. `bluetoothd` restarts every 15–16 s, `.ips` files accumulate

Same cause as above; it is `os_fault_with_payload` after the internal-controller timeout.
Check with:

```sh
launchctl print system/com.apple.bluetoothd | grep -E "runs =|last exit"
```

If `runs` grows by one every ~15 s, you are in the loop.

### 3. No `[bm3_usb]` lines at all

The external-dongle scan never executed. Two possible reasons:

| Reason | Check |
|---|---|
| NVRAM keys missing | `nvram -p \| grep -i bluetooth` — expect all three keys |
| BlueToolFixup not loaded / not first after Lilu | check `Kernel → Add` order, `Lilu` index 0, `BlueToolFixup` index 1 |

**Side note on why the vendor gate seems "not patched".** BlueToolFixup hooks
`_cs_validate_page` and patches each 4 KB page the first time it is paged in. Pages that are
never executed are never patched. A useful confirmation that the patching engine is alive:

```sh
codesign --verify --verbose /usr/sbin/bluetoothd
# => invalid signature   (patches were written into the file page cache this boot)
```

`invalid signature` here is expected and harmless: it only means the in-memory pages differ
from the sealed ones, and it resets on reboot.

### 4. `Reset scan in LE_Scan: STATUS 718` / `restarting { reason=718 }`

`718` is HCI `0x12 Invalid HCI Command Parameters`. `bluetoothd` starts an LE scan with
`Own_Address_Type = random` without ever sending `LE Set Random Address`. Controllers that
follow the spec answer `0x12`, and `bluetoothd` treats that as fatal.

Fix: `IntelBTPatcher.kext` 2.4.0. It hooks `IOUSBHostDevice::deviceRequest`, watches for
`HCI_OP_LE_SET_SCAN_PARAM` (`0x200B`), and injects `LE Set Random Address` (`0x2005`) first.
The hook does not check VID/PID, so it works for non-Intel controllers.

### 5. `LE_InitMultipleAdvertising failed ... STATUS 152` and `Stack is not started`

`152` = `0x98`. On `THIRD_PARTY_DONGLE`, `bluetoothd` never enables Bluetooth-spec advertising
sets. Consequences: no BLE advertising → iPhone/iPad will not discover this Mac over BLE.

This is **not** a crash (it is non-fatal once the scan path is fixed), but it blocks AirDrop
discovery and Handoff.

Upstream work: `acidanthera/BrcmPatchRAM` PR #40 adds `-btlfxadvsets`, which makes
`bluetoothd` use extended advertising on third-party controllers. Its byte patterns currently
cover macOS 26.x only.

### 6. Everything looks fine but the speaker stutters

Not a driver issue in our case. Try:

- move the dongle from a hub to a rear USB 2.0 port (2.4 GHz coexistence),
- keep USB 3.0 devices away from the dongle,
- check `JtBf` (jitter buffer) in `GetControllerInfo` — it should stay near 0 ms.

### 7. Dongle works, Wi-Fi interferes

The chip family is reported by Linux as Actions Semiconductor (its `btusb` quirks flag
`BROKEN_EXT_SCAN`, `BROKEN_SET_RPA_TIMEOUT`, `BROKEN_READ_TRANSMIT_POWER`,
`BROKEN_ERR_DATA_REPORTING`, `BROKEN_READ_ENC_KEY_SIZE` for ATS2851-based devices). We did not
confirm the exact silicon, but the symptom profile matches. Practical consequence: extended
scanning may misbehave, so prefer configurations that keep legacy LE scanning.

---

## Things that did **not** help

| Attempt | Result |
|---|---|
| Spoofing the VID/PID via `AppleUSBHostMergeProperties` | `bluetoothd` still keys off the controller, not the property; no change |
| `BrcmPatchRAM` + `BrcmFirmwareData` (Broadcom firmware upload) | irrelevant — this is not a Broadcom controller |
| BlueToolFixup **without** the NVRAM keys | worse: the chipset patches land, `bluetoothd` powers the phantom internal controller, 16 s crash loop |
| Editing `/usr/sbin/bluetoothd` on disk | impossible on a sealed volume; BlueToolFixup patches pages in memory only |

## Rollback

Keep `config.plist` copies before each change. To go back to a Bluetooth-free system:

1. disable `BlueToolFixup.kext` and `IntelBTPatcher.kext` in `Kernel → Add`,
2. remove the three NVRAM keys from both `Add` and `Delete`,
3. reboot.

The dongle then simply does nothing, and macOS stops trying to power a controller that does
not exist — which is also a stable state.

## Getting more data

If `reason=` values keep changing, capture HCI traffic:

```sh
sudo cp /var/root/Library/Logs/Bluetooth/bluetoothd-hci-*.pklg /tmp/
```

Open the `.pklg` in PacketLogger (Additional Tools for Xcode) and look at the last commands
before the failure. That is how the missing `LE Set Random Address` and the `0x98`
advertising failure are visible at the HCI layer.
