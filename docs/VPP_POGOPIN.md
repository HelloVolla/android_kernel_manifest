# Pogo Pin Module — Hardware & Driver Overview

**Project:** Volla Phone Plinius Plus (`ansuz`)
**Driver:** `drivers/misc/mediatek/extcon/mtk-pogo-usb.c` (`CONFIG_MTK_POGO_USB`)

---

## 1. What the pogo pin module is

In addition to the standard USB Type-C port, this device carries a set of **pogo pins** — spring-loaded contacts on the chassis used to dock the phone to an accessory (cradle, keyboard base, etc.). The pogo connector can deliver **power** and carry **USB data**, so at any moment the phone may be talking to *two* physical interfaces at once: the Type-C port and the pogo dock.

The whole point of this module is **arbitration**: deciding, for every combination of "what's plugged into Type-C" and "what's on the pogo connector," which interface owns the USB data path, which one supplies/consumes power, and what data role (host/device) the controller should run in. When the two interfaces conflict (e.g. a charger on Type-C *and* a host accessory on pogo), the driver hands the decision to userspace via a sysfs node.

### Signal routing

```
                 ┌──────────────┐
   Type-C  ──────┤              │
   port         │  usb_sw mux  ├────── USB controller (role switch)
   Pogo    ──────┤  (GPIO 34)   │
   pins         └──────────────┘
                       │
            pogo_otg (GPIO 38)  → enables pogo OTG / power path
            pogo_otg_int (GPIO 4) → pogo attach/detach interrupt
```

- **`usb_sw` (GPIO 34)** — the USB data-line switch. `0` routes data to the Type-C path, `1` routes it to the pogo path.
- **`pogo_otg` (GPIO 38)** — enables the pogo OTG/power output path.
- **`pogo_otg_int` (GPIO 4)** — input line that fires an IRQ when something is attached to / removed from the pogo connector. Active-low (`0` = pogo source attached). Configured as a wake source.

### Board-variant detection (board_id)

Not every board in the family has the pogo hardware, so the build ships a single image and selects the right driver at runtime using an **ADC board-ID resistor**:

- In `extcon-mtk-usb.c`, the probe reads the PMIC AUXADC channel named `board-channel` (`AUXADC_VIN3`).
- If the processed voltage falls in **280–320 mV**, the board is identified as the pogo variant and the global `board_id` is set to `1` (exported via `EXPORT_SYMBOL_GPL`).
- On that variant the *standard* `extcon-mtk-usb` driver deliberately bails out of probe, leaving the device-tree node (which now lists both `mediatek,extcon-usb` and `mediatek,pogo-usb`) to be claimed by the pogo driver instead.

`board_id` is also consumed in `mtk_chg_type_det.c` to suppress the normal Type-C charger attach/detach notifications on the pogo board, so charger-type handling doesn't fight the pogo state machine.

---

## 2. The driver: `mtk-pogo-usb.c`

It's a `platform_driver` matching `mediatek,pogo-usb`, registered with `late_initcall` (it depends on the charger and TCPC classes being up). It is built as a module (`CONFIG_MTK_POGO_USB=m`).

### Probe sequence

1. Allocate driver context (`struct mtk_pogo_extcon_info`) and initialise default state (no devices attached, role `USB_ROLE_NONE`).
2. Allocate and register an **extcon device** advertising `EXTCON_USB` / `EXTCON_USB_HOST`.
3. Grab the **charger power-supply** (`charger` phandle) used for BC1.2 detection; defer probe if it isn't ready yet.
4. Get the **USB role switch** and the VBUS regulator.
5. Start the **state-machine kthread** (`pogo_machine_thread`).
6. Initialise GPIOs and request the pogo IRQ (`mtk_pogo_usb_gpio_init`).
7. Register the **BC1.2 power-supply notifier**.
8. Register the **TCPC notifier** for Type-C events.
9. Create the **`/sys/usb/pogo_state`** sysfs node.
10. Sample the current Type-C attach state so a cable already inserted at boot is handled.

### Key data structures

`struct usb_state` tracks the resolved decision:

- `data_state` — `DATA_STATE_NONE`, `DATA_STATE_TYPEC_DEVICE`, or `DATA_STATE_HUB_HOST`
- `pwr_state` — `PWR_STATE_NONE`, `PWR_STATE_TYPEC`, or `PWR_STATE_POGO`
- `case_state` — `0` (no conflict, auto-resolve), `1`, or `2` (conflict, ask userspace)
- `port_states` / `state_changed` — change-detection bookkeeping

Physical inputs are tracked as bits in `info->inputs`:

| Bit | Meaning |
|-----|---------|
| `ATTACHED_TYPEC_SRC` | Type-C source attached (host accessory on Type-C) |
| `ATTACHED_TYPEC_SNK` | Type-C sink attached (charger/host upstream on Type-C) |
| `ATTACHED_POGO_SRC`  | Pogo source attached (dock present) |

---

## 3. The arbitration state machine

A dedicated kernel thread (`mtk_pogo_usb_state_machine_thread`) sleeps on a wait queue and runs whenever `machine_run` is set — which happens from the Type-C (TCPC) notifier and from the pogo IRQ handler. On wake it calls `mtk_pogo_usb_get_next_state()` to map the current input bits to a desired `usb_state`.

### Decision table (`mtk_pogo_usb_get_next_state`)

| Inputs present | case_state | Data state | Power state | user_flag | Notes |
|----------------|:---------:|------------|-------------|:---------:|-------|
| Type-C SRC **and** Pogo SRC | 1 | HUB_HOST | NONE | 3 | Two host sources → conflict, ask user |
| Type-C SNK **and** Pogo SRC | 2 | HUB_HOST | TYPEC | 1 or 2 | Charger on Type-C + dock → conflict, ask user |
| Type-C SNK only | 0 | TYPEC_DEVICE | TYPEC | 6 | Phone is a USB device to upstream |
| Type-C SRC only | 0 | HUB_HOST | NONE | 4 | Type-C OTG host |
| Pogo SRC only | 0 | HUB_HOST | NONE | 5 | Pogo OTG host |
| Nothing | 0 | NONE | NONE | 0 | Idle |

When `case_state == 0` the machine resolves automatically via `mtk_pogo_usb_set_route_by_state()`. When `case_state != 0` (a genuine conflict) it instead waits up to **10 seconds** for userspace to write a choice, then applies `mtk_pogo_usb_set_route_by_state_chosen()`.

The `typec_fast` / `pogo_fast` flags record *which* interface arrived second (set in the pogo IRQ and TCPC notifier), so the conflict can be described to userspace correctly.

### Applying a route

Both apply-paths drive the same three knobs in combination:

- `usb_role_switch_set_role()` → HOST / DEVICE / NONE (data role)
- `gpio_set` on `usb_sw` → which physical port owns the data lines
- `gpio_set` on `pogo_otg` → pogo power/OTG path
- `mtk_usb_pogo_extcon_set_vbus()` → turn VBUS on/off

For example, "pogo OTG host" sets `usb_sw=1`, `pogo_otg=1`, role HOST, VBUS on; "Type-C device" sets `usb_sw=0`, `pogo_otg=0`, role DEVICE, VBUS off.

---

## 4. Userspace arbitration: `/sys/usb/pogo_state`

When two interfaces conflict, the kernel can't know the user's intent (do they want to charge, or use the docked accessory?), so it exposes a sysfs node under a freshly created `/sys/usb` kobject:

- **read** returns the current `user_flag` (the scenario code from the table above). A UI layer reads this to learn *what* the conflict is.
- **write** of `1` or `2` selects an option; the store handler combines the written value with `case_state` and the `typec_fast`/`pogo_fast` flags to derive an internal `user_chosen` code (1–6), which feeds the conflict route handler:

| user_chosen | Resulting route |
|:-----------:|-----------------|
| 1 | Host, `usb_sw=1`, `pogo_otg=1` |
| 2 | Device, `usb_sw=0`, `pogo_otg=0` |
| 3 | Branch on detected charger type (SDP/CDP → device, DCP → host) |
| 4 | Host, pogo path |
| 5 | Host, pogo path |
| 6 | Host, Type-C path |

If userspace doesn't answer within the timeout, the machine proceeds with a default host configuration.

---

## 5. Power & charger-type detection (BC1.2)

The pogo driver doesn't run BC1.2 detection itself — it drives the **mt6375 charger** through the power-supply framework:

- `mtk_pogo_usb_bc12_work()` sets `POWER_SUPPLY_PROP_ONLINE` and writes `POWER_SUPPLY_PROP_CHARGE_TYPE = UNKNOWN` to *trigger* a detection cycle, then reads back the result.
- `mt6375-charger.c` was extended so that `POWER_SUPPLY_PROP_CHARGE_TYPE` is now readable/writable: writing `UNKNOWN` queues the charger's `bc12_work`; reading returns the detected `psy_usb_type`.
- The notifier (`mtk_pogo_usb_bc12_psy_notifier_work_handler`) latches the result into `charger_state`: **1** for data-capable ports (SDP/CDP) and **2** for dedicated chargers (DCP). That value is what `user_chosen == 3` branches on.

### VBUS / OTG output

`mtk_usb_pogo_extcon_set_vbus()` → `..._set_vbus_v1()` enables OTG boost on the primary charger (`charger_dev_enable_otg`, boost current limited to **1.5 A**). If the optional wireless-charge chip (`CONFIG_WIRELESS_MT5706`) is present, it also disables wireless TX mode and toggles OTG on the wireless device in step with the wired path.

---

## 6. Coordinating with charging algorithms

When the dock is supplying power over pogo, the device must **not** simultaneously negotiate high-voltage USB-PD or run proprietary fast-charging on Type-C. Several charging modules were taught about the pogo state via a hardwired detect line (`gpio_get_value(300)`, low = pogo present):

- **`pd_dpm_core.c`** — when pogo is present, the advertised sink capability count is forced to `1`, effectively limiting the PD contract to **5 V** (no high-voltage profiles).
- **`mtk_pe2.c`** — PE2.0 algorithm returns `ALG_NOT_READY` at `PE2_HW_READY` while pogo is plugged in.
- **`mtk_pe5.c`** — PE5.0 algorithm returns `ALG_NOT_READY` immediately on start while pogo is plugged in.
- The pogo IRQ handler also actively **stops any running PE5/PE2 session** the moment a pogo source attaches.

---

## 7. Device-tree binding

The existing `extcon-usb` node gains the second compatible and the new pogo properties:

```dts
extcon_usb: extcon-usb {
    compatible = "mediatek,extcon-usb", "mediatek,pogo-usb";
    vbus-supply = <&mt6375_otg_vbus>;
    vbus-voltage = <5000000>;
    vbus-current = <1800000>;
    charger = <&mt6375_chg>;
    tcpc = "type_c_port0";
    mediatek,bypss-typec-sink = <1>;

    io-channels = <&pmic_adc (ADC_PURES_OPEN_MASK | AUXADC_VIN3)>;
    io-channel-names = "board-channel";   /* board-ID resistor */

    usb_sw       = <&pio 34 0x0>;  /* USB data-line mux       */
    pogo_otg_int = <&pio 4  0x0>;  /* pogo attach interrupt   */
    pogo_otg     = <&pio 38 0x0>;  /* pogo OTG / power enable */

    mediatek,u2;
    port { usb_role: endpoint { ... }; };
};
```

---

## Author
Aryan Sinha <aryan.sinha@volla.online>
Volla Systeme GmbH
