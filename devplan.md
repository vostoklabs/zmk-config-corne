## Development plan: Handwired Corne on ProMicro NRF52840 (nice!nano v2 compatible)

This plan adapts the stock ZMK Corne config to your handwired build using a ProMicro‑style NRF52840 board that is a pin‑compatible clone of `nice_nano_v2`. It covers: custom matrix pins for each half, OLED over I2C on P0.17/P0.20, deep sleep with wake on any key, split roles (left=central/master, right=peripheral/slave), and dual USB/BLE operation.

References (read alongside each step):
- New shield (hardware integration, pins, transforms): [zmk.dev › development › hardware-integration › new-shield](https://zmk.dev/docs/development/hardware-integration/new-shield)
- Power management, sleep/wake: [zmk.dev › config › power](https://zmk.dev/docs/config/power)
- Customization and keymaps: [zmk.dev › customization](https://zmk.dev/docs/customization)
- Nice!Nano board (compatible pins): [zmk.dev boards list](https://zmk.dev/docs/hardware)

Assumptions
- Your ProMicro NRF board uses the `nice_nano_v2` board definition (clone-compatible). Use Zephyr pins P0.xx / P1.xx as given below.
- OLEDs are SSD1306 128x32 at I2C address 0x3C on both halves, wired to SDA=P0.17 and SCL=P0.20.
- Typical Corne diode direction is COL2ROW. If your diodes are reversed, change to ROW2COL in the overlays (Step 4).

Hardware pin map (provided)
- Left half
  - rows: r0=P1.00, r1=P0.11, r2=P1.04, r3=P1.06
  - cols: c0=P0.09, c1=P0.10, c2=P1.11, c3=P1.13, c4=P1.15, c5=P0.02
- Right half
  - rows: r0=P0.29, r1=P1.13, r2=P0.10, r3=P0.09
  - cols: c0=P1.06, c1=P1.04, c2=P0.11, c3=P1.00, c4=P0.24, c5=P0.22
  
Note: Row 3 has only thumb keys (3 keys). We will use a matrix-transform to map the sparse bottom row into the Corne layout expected by your `*.keymap`.

---

### Step 1 — Keep board and toolchain
1) Continue targeting board `nice_nano_v2` in all builds. Your clone matches the pinout in the device tree, so no custom board is required.
2) Keep `config/west.yml` as-is to consume upstream ZMK (already present here).

---

### Step 2 — Create custom split shields
Why: The stock `corne_left/right` shields assume different pins. We’ll define two new shields so we can keep upstream ZMK and your keymap layers intact.

Create directories (names are suggestions you can adjust):
- `zmk-config/boards/shields/corne_hw_left/`
- `zmk-config/boards/shields/corne_hw_right/`

Each directory will contain at minimum:
- `corne_hw_left.overlay` or `corne_hw_right.overlay` (device tree pins, I2C, matrix-transform, wakeup)
- `corne_hw_left.keymap` or reuse the existing `config/corne.keymap` by not adding a keymap here
- Optional `Kconfig.defconfig` if you want per-half defaults (e.g., split role)

Doc: New shield structure and where to place files is covered in the new-shield guide.

---

### Step 3 — I2C/OLED on both halves
In each overlay, enable the NRF TWI (I2C) and SSD1306. SDA=P0.17, SCL=P0.20.

Example snippet to place in both overlays:
```dts
&i2c1 {
    status = "okay";
    sda-pin = <17>; /* P0.17 */
    scl-pin = <20>; /* P0.20 */
    clock-frequency = <I2C_BITRATE_STANDARD>;

    oled0: oled@3c {
        compatible = "solomon,ssd1306fb"; /* Zephyr SSD1306 */
        reg = <0x3C>;
        width = <128>;
        height = <32>;
        segment-offset = <0>;
        page-offset = <0>;
        label = "OLED";
    };
};
```

In `config/corne.conf` enable the display (uncomment):
```ini
CONFIG_ZMK_DISPLAY=y
```

Doc: New shield (I2C/display section) and customization docs.

---

### Step 4 — Matrix pins and diode direction per half
Use `zmk,kscan-gpio-matrix` with your exact rows/cols and add `wakeup-source;`. Start with `diode-direction = "col2row";`. If keys don’t scan, flip to `row2col`.

Left overlay core (put inside `/ { ... }`):
```dts
kscan0: kscan {
    compatible = "zmk,kscan-gpio-matrix";
    label = "KSCAN_LEFT";
    /* Rows: P1.00, P0.11, P1.04, P1.06 */
    row-gpios = <&gpio1 0 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 4 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 6 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
    /* Cols: P0.09, P0.10, P1.11, P1.13, P1.15, P0.02 */
    col-gpios = <&gpio0 9 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 10 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 13 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 15 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 2 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
    diode-direction = "col2row";
    debounce-press-ms = <5>;
    debounce-release-ms = <5>;
    wakeup-source; /* Allow wake from deep sleep on any key */
};
```

Right overlay core:
```dts
kscan0: kscan {
    compatible = "zmk,kscan-gpio-matrix";
    label = "KSCAN_RIGHT";
    /* Rows: P0.29, P1.13, P0.10, P0.09 */
    row-gpios = <&gpio0 29 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 13 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 10 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 9 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
    /* Cols: P1.06, P1.04, P0.11, P1.00, P0.24, P0.22 */
    col-gpios = <&gpio1 6 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 4 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio1 0 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 24 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                <&gpio0 22 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
    diode-direction = "col2row";
    debounce-press-ms = <5>;
    debounce-release-ms = <5>;
    wakeup-source;
};
```

Doc: Pin properties, kscan matrix, and diode direction are covered in the new‑shield guide.

---

### Step 5 — Matrix transform for Corne layout
Keep a 4×6 matrix but map the sparse thumb row to the expected logical positions. Reuse the stock Corne transform as a starting point and adjust only if your thumb key wiring yields mismatches.

Example (place alongside `kscan0` in each overlay):
```dts
matrix_transform {
    compatible = "zmk,matrix-transform";
    columns = <6>;
    rows = <4>;
    /* Map physical (row-major) to logical Corne order. Adjust if needed. */
    map = <0 1 2 3 4 5
           6 7 8 9 10 11
           12 13 14 15 16 17
           18 19 20 21 22 23>;
};
```

If some bottom-row positions are not populated, keep them mapped and leave the corresponding keymap bindings as `&none` or `&trans` depending on your preference.

Doc: Matrix transform section in the new‑shield guide.

---

### Step 6 — Split roles and transports
Left is the central (master), right is peripheral (slave).

Per‑half defconfig (optional) or `*.conf` per shield:
```ini
# Left (central)
CONFIG_ZMK_SPLIT=y
CONFIG_ZMK_SPLIT_BLE=y
CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y
CONFIG_ZMK_USB=y

# Right (peripheral)
CONFIG_ZMK_SPLIT=y
CONFIG_ZMK_SPLIT_BLE=y
CONFIG_ZMK_SPLIT_ROLE_PERIPHERAL=y
CONFIG_ZMK_USB=y
```

Notes
- With `CONFIG_ZMK_USB=y`, the left half will enumerate as a USB keyboard when plugged in; BLE stays available. This satisfies the “Bluetooth build but also work wired” requirement.
- Keep the right half flashable over USB for updates, but you normally won’t plug it during use.

Doc: Split details in the hardware‑integration guide; USB/BLE behavior in customization; power doc for USB vs BLE priority.

---

### Step 7 — Deep sleep and wake on key
Enable deep sleep and ensure the kscan is a wakeup source (already added in Step 4). Configure timeouts globally in `config/corne.conf` (or per shield conf):
```ini
CONFIG_ZMK_SLEEP=y
# Example: 10 minutes idle → sleep; tweak to taste
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=600000
```

If you previously observed a board that would not wake except on power, it was likely missing the `wakeup-source` on the kscan (or wired to non‑wakeup capable pins). The pins you selected on nRF52840 support wake via GPIOTE sensing.

Doc: Power management and wake sources page.

---

### Step 8 — Build matrix
Add your shields to `build.yaml` so CI (or local West) produces both halves:
```yaml
include:
  - board: nice_nano_v2
    shield: corne_hw_left
  - board: nice_nano_v2
    shield: corne_hw_right
```

Local builds (examples):
```bash
west build -p -b nice_nano_v2 -- -DSHIELD=corne_hw_left
west build -p -b nice_nano_v2 -- -DSHIELD=corne_hw_right
```

Doc: Customization → building and publishing.

---

### Step 9 — Flashing
1) Put each half into UF2 bootloader (double‑tap RST pads). If your clone lacks a button, briefly short the RST pad twice.
2) Copy the corresponding UF2 artifact to the mass‑storage device.
3) First pair the left (central) over BLE; the right follows as a split peripheral.

---

### Step 10 — Validation checklist
Run in this order to isolate failures:
- Matrix scan: Open a key tester over USB (left); verify each position. If a whole row/col is dead, re‑check pin numbers and diode direction.
- Sleep/wake: Let idle hit the timeout; press a key to wake. If it doesn’t wake, confirm `wakeup-source` exists on `kscan0` and pins support sensing.
- OLED: Confirm boot logo/status. If blank, scan I2C (address 0x3C) and verify `&i2c1` pins 17/20 aren’t swapped.
- Split over BLE: Power only the right; then power the left. Right should join after the left is up.
- Wired fallback: Plug left via USB; ensure keystrokes reach the host immediately (USB has priority when present).

---

### Appendix — Full overlay skeletons

Left (`boards/shields/corne_hw_left/corne_hw_left.overlay`):
```dts
/ {
    chosen { zmk,kscan = &kscan0; };

    kscan0: kscan {
        compatible = "zmk,kscan-gpio-matrix";
        label = "KSCAN_LEFT";
        row-gpios = <&gpio1 0 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 4 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 6 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
        col-gpios = <&gpio0 9 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 10 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 13 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 15 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 2 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
        diode-direction = "col2row";
        debounce-press-ms = <5>;
        debounce-release-ms = <5>;
        wakeup-source;
    };

    matrix_transform {
        compatible = "zmk,matrix-transform";
        columns = <6>;
        rows = <4>;
        map = <0 1 2 3 4 5  6 7 8 9 10 11  12 13 14 15 16 17  18 19 20 21 22 23>;
    };
};

&i2c1 {
    status = "okay";
    sda-pin = <17>;
    scl-pin = <20>;
    clock-frequency = <I2C_BITRATE_STANDARD>;

    oled0: oled@3c {
        compatible = "solomon,ssd1306fb";
        reg = <0x3C>;
        width = <128>;
        height = <32>;
        label = "OLED";
    };
};
```

Right (`boards/shields/corne_hw_right/corne_hw_right.overlay`):
```dts
/ {
    chosen { zmk,kscan = &kscan0; };

    kscan0: kscan {
        compatible = "zmk,kscan-gpio-matrix";
        label = "KSCAN_RIGHT";
        row-gpios = <&gpio0 29 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 13 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 10 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 9 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
        col-gpios = <&gpio1 6 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 4 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio1 0 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 24 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>,
                    <&gpio0 22 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
        diode-direction = "col2row";
        debounce-press-ms = <5>;
        debounce-release-ms = <5>;
        wakeup-source;
    };

    matrix_transform {
        compatible = "zmk,matrix-transform";
        columns = <6>;
        rows = <4>;
        map = <0 1 2 3 4 5  6 7 8 9 10 11  12 13 14 15 16 17  18 19 20 21 22 23>;
    };
};

&i2c1 {
    status = "okay";
    sda-pin = <17>;
    scl-pin = <20>;
    clock-frequency = <I2C_BITRATE_STANDARD>;

    oled0: oled@3c {
        compatible = "solomon,ssd1306fb";
        reg = <0x3C>;
        width = <128>;
        height = <32>;
        label = "OLED";
    };
};
```

You can keep a single shared `corne.keymap` (already in `config/`) so both halves use the same layers. If any positions differ, add per‑half keymaps in the shield directories.

---

### Step 11 — What to change in this repo (summary)
- Add the two new shield directories and overlays (Step 2, 3, 4, 5).
- Uncomment `CONFIG_ZMK_DISPLAY=y` and add sleep options in `config/corne.conf` (Step 3, 7).
- Update `build.yaml` to build `corne_hw_left` and `corne_hw_right` (Step 8).
- Keep using `nice_nano_v2` for both halves.

After these edits, build and flash both halves; test USB first (left), then BLE split, then sleep/wake and OLED.



---

### Addendum — Alternative kscan polarity/pulls (ACTIVE_HIGH + PULL_DOWN)
From your 3×3 numpad tests, scanning only worked when rows were configured as `GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN` and columns as `GPIO_ACTIVE_HIGH`, with `diode-direction = "col2row";` and `wakeup-source;`. If your Corne behaves the same, prefer this variant over the default `ACTIVE_LOW | PULL_UP` style used earlier.

Your working minimal example:
```dts
kscan0: kscan0 {
    compatible = "zmk,kscan-gpio-matrix";
    diode-direction = "col2row";
    wakeup-source;

    row-gpios = <&gpio0 29 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,
                <&gpio0 2  (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,
                <&gpio1 15 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>;

    col-gpios = <&gpio0 9  GPIO_ACTIVE_HIGH>,
                <&gpio1 11 GPIO_ACTIVE_HIGH>,
                <&gpio0 31 GPIO_ACTIVE_HIGH>;
};
```

Apply the same polarity/pulls to the full 4×6 matrices:

Left (ACTIVE_HIGH rows with PULL_DOWN):
```dts
kscan0: kscan {
    compatible = "zmk,kscan-gpio-matrix";
    label = "KSCAN_LEFT";
    row-gpios = <&gpio1 0  (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,  /* P1.00 */
                <&gpio0 11 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,  /* P0.11 */
                <&gpio1 4  (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,  /* P1.04 */
                <&gpio1 6  (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>;  /* P1.06 */
    col-gpios = <&gpio0 9  GPIO_ACTIVE_HIGH>,  /* P0.09 */
                <&gpio0 10 GPIO_ACTIVE_HIGH>,  /* P0.10 */
                <&gpio1 11 GPIO_ACTIVE_HIGH>,  /* P1.11 */
                <&gpio1 13 GPIO_ACTIVE_HIGH>,  /* P1.13 */
                <&gpio1 15 GPIO_ACTIVE_HIGH>,  /* P1.15 */
                <&gpio0 2  GPIO_ACTIVE_HIGH>;  /* P0.02 */
    diode-direction = "col2row";
    debounce-press-ms = <5>;
    debounce-release-ms = <5>;
    wakeup-source;
};
```

Right (ACTIVE_HIGH rows with PULL_DOWN):
```dts
kscan0: kscan {
    compatible = "zmk,kscan-gpio-matrix";
    label = "KSCAN_RIGHT";
    row-gpios = <&gpio0 29 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,  /* P0.29 */
                <&gpio1 13 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,  /* P1.13 */
                <&gpio0 10 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>,  /* P0.10 */
                <&gpio0 9  (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>;  /* P0.09 */
    col-gpios = <&gpio1 6  GPIO_ACTIVE_HIGH>,  /* P1.06 */
                <&gpio1 4  GPIO_ACTIVE_HIGH>,  /* P1.04 */
                <&gpio0 11 GPIO_ACTIVE_HIGH>,  /* P0.11 */
                <&gpio1 0  GPIO_ACTIVE_HIGH>,  /* P1.00 */
                <&gpio0 24 GPIO_ACTIVE_HIGH>,  /* P0.24 */
                <&gpio0 22 GPIO_ACTIVE_HIGH>;  /* P0.22 */
    diode-direction = "col2row";
    debounce-press-ms = <5>;
    debounce-release-ms = <5>;
    wakeup-source;
};
```

Notes
- Rows are inputs biased low; when a driven column goes high, current flows through the diode to the selected row, which reads high. This matches `col2row` and enables wake-on-key.
- Use one style consistently: either `ACTIVE_LOW | PULL_UP` (default) or this `ACTIVE_HIGH | PULL_DOWN` variant. Mixing styles per pin can prevent scanning.
- If columns float or you see chatter, optionally add `GPIO_PULL_DOWN` to columns or raise debounce values.

