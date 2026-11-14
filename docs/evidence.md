# Evidence

Raw observations from the test machine. MAC addresses of the dongle and the speaker are
partially masked; everything else is verbatim from `log show` / `system_profiler`.

## Environment

| Item | Value |
|---|---|
| macOS | 14.7.7, build 23H723 (`sw_vers`, `sysctl -n kern.osversion`) |
| Bootloader | OpenCore |
| SMBIOS | `iMac19,1` |
| Mainboard | desktop H310 board, no Wi-Fi slot populated |
| Network | Ethernet only — `networksetup -listallhardwareports` shows no 802.11 interface |
| Dongle | Barrot `33fa:0001`, product string `USB2.0-BT`, permanently plugged |
| Bluetooth firmware | `v137 c137`, `LMPv 10 (5.1)` |

## Before: phantom controller and crash loop

`system_profiler SPBluetoothDataType`:

```
Bluetooth Controller:
    Address: NULL
    State: Off
    Chipset: BCM_4350C2
    Discoverable: Off
    Firmware Version: v0 c0
    Supported services: 0x392039 < HFP AVRCP A2DP HID Braille LEA AACP GATT SerialPort >
    Transport: UART
    Vendor ID: 0x004C (Apple)
```

`launchctl print system/com.apple.bluetoothd`:

```
state = spawn scheduled
runs = 178
last exit code = 1
```

`/Library/Logs/DiagnosticReports/` contained 77 accumulated `ExcUserFault_bluetoothd-*.ips`,
all with the same shape:

```
exception: GUARD_TYPE_USER namespc 18 reason_code 0x0000000000000002
0 os_fault_with_payload
1 ... bluetoothd
...
7 _dispatch_call_block_and_release
```

Nothing from the USB dongle appears anywhere in the log — no `[bm3_usb]`, no `0x33fa`.

## After: external dongle accepted

`log show --predicate 'process == "bluetoothd" AND eventMessage CONTAINS "bm3_usb"'`
(first boot with BlueToolFixup + NVRAM keys):

```
[b m3_usb][Found_USB_Dongle] -- Found USB Device :
[b m3_usb][Found_USB_Dongle] -- FoundBluetoothInterfaces() for deviceClass (224), idVendor = 0x33FA, idProduct = 0x0001 returned true
[b m3_usb][Found_USB_Dongle] -- setting gUseInternalController to FALSE
[b m3_usb][Found_USB_Dongle] -- returned TRUE
[b m3_usb][Initialize] -- Before calling pthread_cond_wait()
[b m3_usb][IOThreadFunc] -- USB -- calling GetProductAndVendorID()
```

(the leading `[b m3_usb]` is a rendering artefact of the log filter; the real tag is `[bm3_usb]`)

Controller info once powered:

```
GetControllerInfo: BlSt PoweredOn, Chip THIRD_PARTY_DONGLE, FrmV v137 c137,
                   HWAd 04:7F:0E:xx:xx:xx, LMPv 10 (5.1)
```

## Failure between layer 2 and layer 3

With BlueToolFixup + NVRAM but **without** `IntelBTPatcher`:

```
LE_InitMultipleAdvertising failed with status  STATUS 152 (status=65535)
Reset scan in LE_Scan:  STATUS 718 for location 3 (status=65535)
Bluetooth error - restarting { build=release, reason=718, context="" }
```

`719`'s meaning is HCI `0x12 Invalid HCI Command Parameters`; OpenIntelWireless documents
that `0x12` is reported as `OI_STATUS 718` in `bluetoothd` logs. Restart interval was
10–16 s, with `runs` incrementing every time.

After adding `IntelBTPatcher.kext` 2.4.0:

```
state = running
runs = 1
last exit code = (never exited)
pid = 156
```

and no new crash report was written after that boot.

## Final state

`system_profiler SPBluetoothDataType`:

```
Bluetooth Controller:
    Address: 04:7F:0E:xx:xx:xx
    State: On
    Chipset: THIRD_PARTY_DONGLE
    Discoverable: Off
    Firmware Version: v137 c137
    Supported services: 0x392039 < HFP AVRCP A2DP HID Braille LEA AACP GATT SerialPort >
    Transport: UART
    Vendor ID: 0x004C (Apple)
```

`ioreg -l -w0 -r -c IOBluetoothHCIController`:

```
"BluetoothTransportConnected" = Yes
"BTAddress" = <047f0eXXXXXX>
"IOPowerManagement" = {"CapabilityFlags"=2,"MaxPowerState"=2,"CurrentPowerState"=2}
```

A2DP playback to a Philips CN-BT55 speaker:

```
Connected:
    Philips CN-BT55:
        Minor Type: Speaker
        RSSI: -60
        Services: 0x800019 < HFP AVRCP A2DP ACL >
```

```
AuLQ [{BtRt 0 Kbps, Codc SBC, DvNm 'Philips CN-BT55', JtBf 0 ms, NsFl 0, RtmR 0%, ...}]
Default Output Device: Yes
Output Channels: 2
Current SampleRate: 44100
```

## What still fails

BLE scanning is fine:

```
[CBStackBLEScanner] Scan start: rescan, DsFl ... < NearbyInfo ScreenOff BLEAdvertisementData RSSI Ensemble Identify Leech >, DsTy <AirDrop, MacMigrate>, ...
[Server.LE.Scan] Scanning stopped successfully.  6 Advertising Events, 0 dropped pkts
```

BLE advertising is not:

```
[Server.LE.Advertise] Stack is not started, updateAdvertisement returning
```

repeating every few seconds for the whole session. This is the `0x98` path: on a
`THIRD_PARTY_DONGLE`, `bluetoothd` never enables Bluetooth-spec advertising sets.

No Wi-Fi hardware is present, so there is no `awdl0`:

```
$ ifconfig -l
lo0 gif0 stf0 XHC20 en0 utun0 utun1 utun2 utun3
```

`en0` is Ethernet. Consequently AirDrop cannot work on this machine even if BLE advertising
were fixed.
