# Monterey

OS Version: 12.7.6 (21H1320)

## Hardware

### Device with normal driver

- Intel HD Graphics 530
- Built-in display (Control Center adjustable brightness)
- SATA SSD x2
- USB 3.0 x1 (Port5 & 21), USB 2.0 x1 
	- The USB 3.0 ports near the 3.5mm headphone jack on my computer are broken (ports 6 and 22)
- Intel Wireless network card (WIFI)
- RTL wired network card
- RTL Audio (Internal Speakers & Microphone)
- Touchpad (left click, right click)
- Camera (USB protocol)
- Battery
- Intel Power Management
- Intel Bluetooth

### Unresolved device

- Touchpad (gesture)
- No sound from the speakers after waking up from sleep

### UnTested device

- HDMI、VGA interface
- SD card reader
- 3.5mm audio jack

### Device that can't be driven

- NVIDIA GeForce 940MX

## Fn key

| Fn key              | Function                                | Status                                                   |
| ------------------- | --------------------------------------- | -------------------------------------------------------- |
| Fn + F1             | Sleep                                   | ✔                                                        |
| Fn + F2             | Switch to airplane mode                 | ❌                                                       |
| Fn + F5             | Turn down the screen brightness         | ❌                                                       |
| Fn + F6             | Turn up the screen brightness           | ❌                                                       |
| Fn + F7             | Turn off the screen                     | ❌                                                       |
| Fn + F8             | Switch display mode (Switch monitor)    | ❌                                                       |
| Fn + F9             | Turn the touchpad on or off             | ❌                                                       |
| Fn + F10            | Turn mute on or off                     | ✔                                                        |
| Fn + F11            | Turn down the volume                    | ✔                                                        |
| Fn + F12            | Turn up the volume                      | ✔                                                        |
| Fn + Up             | Stop the music playback                 | ❌                                                       |
| Fn + Down           | Pause or play the music playback        | ❌                                                       |
| Fn + Left           | Previous song or rewind                 | ❌                                                       |
| Fn + Right          | Next song or fast forward               | ❌                                                       |
| Fn + Num lock       | Switch the numeric keypad or arrow keys | unexpected turn down screen brightness (Fn + / (numpad)) |
| Fn + delete         | `insert` key                            | ❌                                                       |
| Fn + enter (numpad) | open Calculator                         | ❌                                                       |

## Note

### About Audio Layout-ID & Permanent Speaker Fix

Through strict cross-referencing between the motherboard's codec dump and the AppleALC source code (`Platforms27.xml`), **`layout-id=27`** is the only layout that perfectly matches the physical routing of the ASUS X550VQ.

#### 1. Why `layout-id=27` is the perfect match?

*   **Hex to Decimal Node Mapping from Codec Dump:**
    *   **Speaker (内放喇叭):** Node `0x14` (Dec **20**) -> Mixer `0x0c` (12) -> DAC `0x02` (2)
    *   **Headphone (耳机孔):** Node `0x21` (Dec **33**) -> Mixer `0x0d` (13) -> DAC `0x03` (3)
    *   **Internal Mic (内置麦克风):** Node `0x1b` (Dec **27**) -> Mixer `0x22` (34) -> ADC `0x09` (9)
    *   **Headset Mic (耳麦麦克风):** Node `0x19` (Dec **25**) -> Mixer `0x23` (35) -> ADC `0x08` (8)

*   **DSP Routing in `Platforms27.xml` (PathMapID 255):**
    *   **Output 1:** `20 -> 12 -> 2` (Perfectly matches Speaker)
    *   **Output 2:** `33 -> 13 -> 3` (Perfectly matches Headphone)
    *   **Input 1:** `8 -> 35 -> 25` (Perfectly matches Headset Mic)
    *   **Input 2:** `9 -> 34 -> 27` (Perfectly matches Internal Mic)

By contrast, other layouts like `layout-id=30` point the microphone path to `Node 18` (0x12), which is an unconnected (`[N/A] Line Out`) dead node on this machine, resulting in a non-functional microphone.

---

#### 2. The Root Cause of Silent Speakers (ALC255/3234 Deep Sleep)

Although Layout 27 has the perfect hardware routing paths, the Realtek ALC255 Class-D speaker amplifier (Node `0x14`) and clock gate (Node `0x10`) default to a deep-sleep/mute state on cold boot (and are aggressively cut by the Windows driver on shutdown).

Under the default/silent state of Layout 27, querying the active coefficients on NID `0x20` yields:
*   **Index `0x45` (Amp Control) = `0xd089`** (Bit 15 is `1` = Speaker Amplifier hardware-muted)
*   **Index `0x10` (Clock Gate) = `0x0220`** (Bit 9 is `1` = Audio stream clock muted/cut)

To wake up the physical amplifier, the following values must be forced into the codec:
*   Clear Bit 15 on **Index `0x45`** to `0` (set data to **`0x5089`**)
*   Clear Bit 9 on **Index `0x10`** to `0` (set data to **`0x0020`**)
*   Enable EAPD on **Node `0x14`** (send EAPD enable command **`01470C02`**)

```bash
sudo ./alc-verb 0x20 0x500 0x45
sudo ./alc-verb 0x20 0x400 0x5089
sudo ./alc-verb 0x20 0x500 0x10
sudo ./alc-verb 0x20 0x400 0x0020
```

*(Note: Layout 30 natively initializes these registers, which is why it had sound, but physically points the microphone to a dead node. Layout 27 has the correct routing but lacks the default initialization verbs).*

---

#### 3. The Elegant Solution (Native AppleALC Patching)

Instead of using background daemons (like ALCPlugFix) or manual `alc-verb` terminal commands, we can inject these custom initialization machines code verbs directly into the pre-compiled `AppleALC.kext`'s configuration block.

**Step-by-step Patching:**
1. Open `AppleALC.kext/Contents/Info.plist` with a plist editor.
2. Search for `283902549` (Decimal CodecID for ALC255) under `IOKitPersonalities -> as.vit9696.AppleALC -> HDAConfigDefault`.
3. Locate the dictionary block containing `<key>LayoutID</key> <integer>27</integer>`.
4. Replace the entire `<dict>` configuration block with the following patched XML:

```xml
<dict>
	<key>AFGLowPowerState</key>
	<data>
	AwAAAA==
	</data>
	<key>Codec</key>
	<string>ALC255 for Asus X556UA m-dudarev</string>
	<key>CodecID</key>
	<integer>283902549</integer>
	<key>ConfigData</key>
	<data>
	AUccEAFHHQEBRx4XAUcfkAGXHCABlx0QAZce
	gQGXHwQCFxwgAhcdEAIXHiECFx8EAbccMAG3
	HQEBtx6gAbcfkAFHDAgCBQAQAgQAIAgFAEUC
	BFCJAUcMAg==
	</data>
	<key>FuncGroup</key>
	<integer>1</integer>
	<key>LayoutID</key>
	<integer>27</integer>
	<key>WakeConfigData</key>
	<data>
	AgUAEAIEACACACACBQBFAgRQiQFHDAI=
	</data>
	<key>WakeVerbReinit</key>
	<true/>
</dict>
```

5. Save the plist, replace the kext in your OpenCore EFI/OC/Kexts, and Reset NVRAM on your next boot.

This natively forces AppleALC to clear the deep-sleep register blocks at the kernel level upon boot and wake, ensuring 100% stable, out-of-the-box native audio and microphone on macOS Monterey without any auxiliary software.