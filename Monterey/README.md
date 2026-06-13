# Monterey

OS Version: 12.7.6 (21H1320)

## Hardware

### Device with normal driver

- Intel HD Graphics 530
- Built-in display (Control Center adjustable brightness)
- SATA SSD x2
- USB 3.0 x2, USB 2.0 x1
- Intel Wireless network card (WIFI)
- RTL wired network card
- Microphone
- Touchpad (left click, right click)
- Camera (USB protocol)
- Battery
- Intel Power Management
- Intel Bluetooth

### Unresolved device

- Touchpad (gesture)
- RTL Audio (speeakers, microphone)
- Sleep (USB devices still powered on)

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

### About Audio Layout-ID

#### Pristine (original) environment

| Layout ID | Result - has sound | Result - microphone picking up |
| --------- | ------------------ | ------------------------------ |
| 3         | ❌                 | UnTested                       |
| 27        | ❌                 | ✔                              |
| 28        | ❌                 | UnTested                       |
| 30        | ✔                  | ❌                             |
| 13        | ❌                 | ❌                             |
| 11        | ❌                 | ❌                             |
| 21        | ❌                 | ❌                             |
| 99        | ❌                 | ❌                             |

#### apply HEPT、IRQ、RTC、TIMR patches and Virtual Sound Card

> full-patches.diff

| Layout ID | Result - has sound | Result - microphone picking up |
| --------- | ------------------ | ------------------------------ |
| 27        | ❌                 | ✔                              |
| 30        | ❌                 | ❌                             |

### only apply HEPT、IRQ、RTC、TIMR patches

| Layout ID | Result - has sound | Result - microphone picking up |
| --------- | ------------------ | ------------------------------ |
| 30        | ❌                 | ❌                             |
| 27        | ❌                 | ✔                              |
| 66        | ❌                 | ❌                             |
| 21        | ❌                 | ❌                             |

### Force enable audio speeakers

```bash
sudo ./alc-verb 0x20 0x500 0x45
sudo ./alc-verb 0x20 0x400 0x5089
sudo ./alc-verb 0x20 0x500 0x10
sudo ./alc-verb 0x20 0x400 0x0020
```

### Why `layout-id=27` is the perfect match? (Hardware & Source Code Analysis)

Through strict cross-referencing between the motherboard's codec dump and the AppleALC source code (`Platforms27.xml`), `layout-id=27` is the only layout that perfectly matches the physical routing of the ASUS X550VQ.

**1. Hex to Decimal Node Mapping from Codec Dump:**

- **Speaker (内放喇叭):** Node `0x14` (Dec **20**) -> Mixer `0x0c` (12) -> DAC `0x02` (2)
- **Headphone (耳机孔):** Node `0x21` (Dec **33**) -> Mixer `0x0d` (13) -> DAC `0x03` (3)
- **Internal Mic (内置麦克风):** Node `0x1b` (Dec **27**) -> Mixer `0x22` (34) -> ADC `0x09` (9)
- **Headset Mic (耳麦麦克风):** Node `0x19` (Dec **25**) -> Mixer `0x23` (35) -> ADC `0x08` (8)

**2. DSP Routing in `Platforms27.xml` (PathMapID 255):**

- **Output 1:** `20 -> 12 -> 2` (Perfectly matches Speaker)
- **Output 2:** `33 -> 13 -> 3` (Perfectly matches Headphone)
- **Input 1:** `8 -> 35 -> 25` (Perfectly matches Headset Mic)
- **Input 2:** `9 -> 34 -> 27` (Perfectly matches Internal Mic)

By contrast, other layouts like `layout-id=30` point the microphone path to `Node 18` (0x12), which is an unconnected (`[N/A] Line Out`) dead node on this specific machine, resulting in a non-functional microphone.

---

### Archive Record: The Index Mismatch Anomaly in Layout 27

Although `layout-id=27` works perfectly in practice, there is a known indexing anomaly in the AppleALC source code for this layout that is worth documenting:

In `layout27.xml`, the outputs are declared as:

1. `Headphone` (Index 0)
2. `IntSpeaker` (Index 1)

However, in `Platforms27.xml`, the DSP paths are ordered as:

1. Node `20` (Speaker)
2. Node `33` (Headphone)

This creates an "Index Mismatch" where the XML definition ties the Headphone name to the Speaker node, and vice versa.
**Why it still works:** macOS CoreAudio is smart enough to rely on the injected `PinConfigs` (`ConfigData`) rather than strictly following the XML naming strings. Since the `PinConfigs` correctly define Node 20 as an internal speaker and Node 33 as a headphone, the system resolves the routing dynamically.
