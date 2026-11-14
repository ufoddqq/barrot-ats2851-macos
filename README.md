# Barrot `33fa:0001` USB Bluetooth on macOS 14

**Status: works for Bluetooth audio (A2DP). Does not enable AirDrop.**

A cheap, always-plugged Barrot / Actions USB Bluetooth dongle (`33fa:0001`, product string
`USB2.0-BT`) that macOS does not know about, running on a Hackintosh with OpenCore.

| Check | Result |
|---|---|
| `system_profiler SPBluetoothDataType` | `Address: 04:7F:0E:xx:xx:xx`, `State: On` |
| Chipset | `THIRD_PARTY_DONGLE` |
| Firmware | `v137 c137`, `LMPv 10 (5.1)` |
| `launchctl print system/com.apple.bluetoothd` | `runs = 1`, `last exit code = (never exited)` |
| A2DP to a speaker | works — SBC, 44100 Hz, 2 ch, jitter buffer 0 ms |
| AirDrop / Handoff / Universal Clipboard | **no** — needs BLE advertising, which stays off (see [Limitations](#limitations)) |

Tested on macOS **14.7.7 (23H723)**, OpenCore, SMBIOS `iMac19,1`, desktop (no Wi-Fi card).

This repository is a compatibility report plus the exact configuration. No new kext is
shipped here; both kexts are upstream projects, linked below.

---

## TL;DR — three layers, all three required

| # | Layer | What it fixes |
|---|---|---|
| 1 | **BlueToolFixup** 2.7.2 | `bluetoothd`'s vendor allow-list rejects `0x33FA`; the patch turns that gate into an unconditional jump, and relaxes the chipset check so the controller is accepted as `THIRD_PARTY_DONGLE` |
| 2 | **NVRAM keys** | Without them `bluetoothd` never scans for an *external* dongle at all — it goes straight for the internal controller your SMBIOS implies, times out after ~16 s and kills itself in a loop |
| 3 | **IntelBTPatcher** 2.4.0 | Injects `LE Set Random Address` before the first `LE Set Scan Parameters`; without it `bluetoothd` dies on `STATUS 718` (HCI `0x12 Invalid HCI Command Parameters`) |

Installing only #1 makes things **worse** (crash loop). Order and completeness matter.

---

## 1. Symptom on stock macOS

```
Bluetooth Controller:
    Address: NULL
    State: Off
    Chipset: BCM_4350C2
    Transport: UART
    Vendor ID: 0x004C (Apple)
```

plus `bluetoothd` restarting every 15–16 seconds and `ExcUserFault_bluetoothd-*.ips` reports
(`GUARD_TYPE_USER`, `os_fault_with_payload`).

`BCM_4350C2 / UART / 0x004C` is a **phantom**: it is what `bluetoothd` expects from the
SMBIOS board-id (`iMac19,1` ships a Broadcom combo card), not what is on the USB bus.
`Transport: UART` on a third-party dongle is also cosmetic — the real transport is USB.

The dongle itself never appears in the log at all. There are **no** `[bm3_usb]` lines, because
the external-dongle scan never runs.

## 2. Why it fails

1. **Vendor allow-list.** `bluetoothd` accepts `0x0A12` (CSR), `0x0A5C` (Broadcom),
   `0x0B05:0x190E` (ASUS). `0x33FA` is rejected → falls back to the internal path.
2. **Patches are applied per 4 KB page.** BlueToolFixup hooks `_cs_validate_page` and patches
   each page when it is first paged in. If the scan code never executes, its pages are never
   patched — which is exactly what happens without the NVRAM keys.
3. **Since v2.7.0, BlueToolFixup requires two NVRAM variables** on every macOS version.
   Without them `bluetoothd` takes the "internal controller" branch.
4. **`LE Set Random Address` is never sent**, so the LE scan with a random own address returns
   `0x12`, which `bluetoothd` treats as fatal.

## 3. Configuration

### 3.1 Kexts

| Order | Bundle path | Version |
|---|---|---|
| 0 | `Lilu.kext` | required by both |
| 1 | `BlueToolFixup.kext` | 2.7.2 (from BrcmPatchRAM 2.7.2 release) |
| 2 | `IntelBTPatcher.kext` | 2.4.0 (`com.zxystd.IntelBTPatcher`) |

`IntelBTPatcher` is named after Intel, but its hook is **vendor-agnostic**: it only looks at the
HCI opcode (`0x200B`) and does not check VID/PID. That is why it fixes this dongle too.
See `snippets/kernel-add.txt` for the OpenCore entries.

### 3.2 NVRAM

Under GUID `7C436110-AB2A-4BBB-A880-FE41995C9F82`:

| Key | Value |
|---|---|
| `bluetoothExternalDongleFailed` | `<data>00</data>` |
| `bluetoothInternalControllerInfo` | `<data>` 14 zero bytes |
| `bluetoothHostControllerSwitchBehavior` | `always` |

Also add the same three keys to `NVRAM → Delete` for that GUID, so they are rewritten to `00`
on every boot (otherwise `bluetoothd` may write `01` and permanently give up).

Ready-to-paste XML: `snippets/nvram-fragment.xml`.

### 3.3 Debug (optional)

`boot-args` += `-btlfxdbg` while bringing the dongle up. Lilu then logs every patch site it
hits. Remove it once everything works.

## 4. Verify

```sh
system_profiler SPBluetoothDataType | head -15
launchctl print system/com.apple.bluetoothd | grep -E "runs =|last exit"
log show --last 3m --predicate 'eventMessage CONTAINS "bm3_usb"' | head
```

Expected:

```
Address: 04:7F:0E:xx:xx:xx
State: On
Chipset: THIRD_PARTY_DONGLE
Transport: UART            # cosmetic label, ignore
runs = 1
last exit code = (never exited)
[bm3_usb][Found_USB_Dongle] -- Found USB Device :
[bm3_usb][Found_USB_Dongle] -- FoundBluetoothInterfaces() for deviceClass (224), idVendor = 0x33FA, idProduct = 0x0001 returned true
[bm3_usb][Found_USB_Dongle] -- setting gUseInternalController to FALSE
```

Then pair a speaker and check that it becomes the default output at 44100 Hz.

## 5. Failure modes we hit, in order

| Symptom | Cause | Fix |
|---|---|---|
| `BCM_4350C2 / UART / NULL / Off`, crash every 16 s, no `[bm3_usb]` logs | NVRAM keys missing → no external scan → scan pages never paged in → vendor gate never patched | add the three NVRAM keys |
| Dongle accepted, but `Bluetooth error - restarting { reason=718 }` every 10–16 s | no `LE Set Random Address` before LE scan | `IntelBTPatcher` |
| Everything works, but iPhone never sees the Mac | BLE advertising never starts (`LE_InitMultipleAdvertising` = `0x98`, `Stack is not started`) | not solved here — see below |

More detail: [`docs/troubleshooting.md`](docs/troubleshooting.md).
Raw evidence: [`docs/evidence.md`](docs/evidence.md).

## Limitations

<a id="limitations"></a>

**No AirDrop, no Handoff, no Universal Clipboard.** Those need BLE *advertising*, and on a
`THIRD_PARTY_DONGLE` `bluetoothd` never starts it:

```
[Server.LE.Advertise] Stack is not started, updateAdvertisement returning
```

BLE *scanning* works fine, so Bluetooth audio and LE device discovery do work.

There is active upstream work on this: `acidanthera/BrcmPatchRAM` **PR #40**
(`-btlfxadvsets`) makes `bluetoothd` take the extended-advertising path on third-party
controllers. As of this writing its byte patterns only cover macOS 26.x, and AirDrop also
needs **AWDL**, i.e. a Wi-Fi card macOS can drive (native Broadcom, or Intel + ForbiddenItlwm).
On a machine with no Wi-Fi hardware there is nothing to patch — AirDrop is impossible.

## Credits

| Project | Author | License |
|---|---|---|
| [BrcmPatchRAM / BlueToolFixup](https://github.com/acidanthera/BrcmPatchRAM) | acidanthera | GPL-2.0 |
| [IntelBluetoothFirmware / IntelBTPatcher](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) | zxystd, OpenIntelWireless | GPL-2.0 |
| [Lilu](https://github.com/acidanthera/Lilu) | acidanthera | BSD-3-Clause |

Nothing here is a fork or a modified binary; this repo only records the configuration and the
observed behaviour.

## License

Documentation: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Configuration snippets may be used freely without attribution.
