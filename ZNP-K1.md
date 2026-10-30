# Building the OpenNept4une ZNP-K1 images

The Elegoo Neptune 4 printers use the ZNP-K1 controller board (RK3328, a
Makerbase MKS-PI derivative). This branch builds one Armbian image for each
printer family and board revision. Every image contains all of the ZNP-K1 DTBs
(they all ship in the `linux-dtb` package). Which one boots is set at build time
by `ZNP_K1_VARIANT`, which writes `fdtfile=` into `/boot/armbianEnv.txt`.

## Images a release needs

| `ZNP_K1_VARIANT` | Printer | Board | Toolhead | Booted DTB |
|---|---|---|---|---|
| `n4` | Neptune 4 / 4 Pro | ZNP-K1 V1.x | standard | `rk3328-znp-k1-v1x.dtb` |
| `n4plus` | Neptune 4 Plus / Max | ZNP-K1 V2.0 | standard | `rk3328-znp-k1-v2x.dtb` |
| `n4-v1.4` | Neptune 4 / 4 Pro | ZNP-K1 V1.4 | USB-C | `rk3328-znp-k1-n4-v1.4.dtb` |
| `n4plus-v2.3` | Neptune 4 Plus / Max | ZNP-K1 V2.3 | USB-C | `rk3328-znp-k1-n4plus-n4max-v2.3.dtb` |

The variants are defined in `config/boards/znp-k1.conf`. If an image build uses
any other value, it stops with an error. Kernel-only and U-Boot-only builds
don't need a variant.

## Build

Build each image with `run.sh` (the release settings: `current` kernel, Debian
trixie, minimal, xz-compressed):

```sh
./run.sh n4
./run.sh n4plus
./run.sh n4-v1.4
./run.sh n4plus-v2.3
```

Or build all four in one go:

```sh
for v in n4 n4plus n4-v1.4 n4plus-v2.3; do ./run.sh "$v" || break; done
```

Each run is a full Armbian image build, and the first one compiles the kernel,
so expect it to take a long time. Images are written to `output/images/`.

`run.sh` always builds `BRANCH=current` (kernel 6.18). To test the `edge`
kernel (7.1), run `compile.sh` with the same arguments as `run.sh`, but
`BRANCH=edge`.

### Where the build inputs live

- Board trees: `patch/kernel/archive/rockchip64-{6.18,7.1,7.2}/dt/rk3328-znp-k1-*.dts`
  (identical in each kernel dir; keep them in sync). Files in `dt/` are added
  to the kernel's dtbs Makefile automatically.
- Variant → DTB mapping: `config/boards/znp-k1.conf`.
- Kernel configs: `config/kernel/linux-rockchip64-{current,edge}.config`.
- Image customisation: `userpatches/customize-image.sh` and `userpatches/overlay/`.

### Kernel config requirement: legacy GPIO sysfs

On the USB-C machines, toolhead power is SoC GPIO 82 (GPIO2_C2).
OpenNept4une's `opennept4une-toolhead-power.service` drives it through
`/sys/class/gpio/export`, and power-cycles it to put the toolhead into its
bootloader for flashing. Since Linux 6.17 that interface depends on
`CONFIG_GPIO_SYSFS_LEGACY`. The option still defaults to `y`, but it's pinned
explicitly in the `current` and `edge` configs so that an upstream default
change can't silently break toolhead power.

Because these configs are savedefconfig output, regenerating them (e.g.
`rewrite-kernel-config`) drops lines that match the default. After
regenerating, check that `CONFIG_GPIO_SYSFS_LEGACY=y` is still there.

The DTBs deliberately don't gpio-hog GPIO2_C2: a hog would claim the line and
userspace could no longer drive it. The line is only named `toolhead-power`.

## Test on hardware

For every image:

```sh
cat /proc/device-tree/model; echo     # matches the variant, e.g. "Elegoo ZNP-K1 V2.3"
grep fdtfile /boot/armbianEnv.txt      # the DTB from the table above
grep rk805 /proc/interrupts            # PMIC IRQ, count stays low
```

Also for the USB-C images (`n4-v1.4`, `n4plus-v2.3`):

```sh
grep rk805 /proc/interrupts              # rockchip_gpio_irq pin 6 (GPIO2_A6), not 24
                                         # (GPIO1_D0); count stays low
gpioinfo gpiochip2 | grep toolhead       # line 18 "toolhead-power", unused
cat /sys/class/gpio/gpiochip64/label     # gpio2, so GPIO 82 is GPIO2_C2
ls /sys/class/gpio/export                # must exist (legacy sysfs enabled)
sudo opennept4une-toolhead-power status  # after picking USB-C in the printer.cfg setup:
                                         # value=1, toolhead=/dev/serial/by-id/usb-Klipper_...
```

## Switching DTB on an installed system

To switch DTB on an installed system, edit `fdtfile=rockchip/<name>.dtb` in
`/boot/armbianEnv.txt` and reboot. Besides the four above, these also ship:

| DTB | Board |
|---|---|
| `rk3328-znp-k1-n4-v1.0.dtb` | Neptune 4 / 4 Pro, ZNP-K1 V1.0 |
| `rk3328-znp-k1-n4-v1.1.dtb` | Neptune 4 / 4 Pro, ZNP-K1 V1.1 |
| `rk3328-znp-k1-n4plus-n4max.dtb` | Neptune 4 Plus / Max, ZNP-K1 V1.1-V2.0 |

The older "copy the DTB over `rk3328-roc-cc.dtb`" instructions only apply to
the 6.16-based images, not to these.

## Releases (CI)

`.github/workflows/release-znp-k1.yml` runs when a tag is pushed. It builds
the four variants in parallel, each with `./run.sh <variant>`, so CI uses the
same flags as a local build. The images are renamed `<armbian name>-<variant>.img.xz`,
because Armbian names images after the board only and the four would otherwise
collide. A final job then publishes all of them in one prerelease named
after the tag.

## Outstanding

- V2.0 PMIC interrupt: `rk3328-znp-k1-v2x.dts` (and the `n4-v1.0`, `n4-v1.1`
  and `n4plus-n4max` trees) put the RK805 interrupt on GPIO1_D0, as the stock
  V1.1 and V1.1-2.0 trees do. On V2.3 that mapping caused 100,001 unhandled
  IRQs, and GPIO2_A6 is correct there. Whether V2.0 boards have the same
  problem is unknown; `grep rk805 /proc/interrupts` from a V2.0 owner would
  settle it. If the count climbs into the thousands, or `dmesg` reports
  `nobody cared` for that IRQ, move those trees to GPIO2_A6 as well.

## Credits

USB-C toolhead support (OpenNept4une#417): dalgibbard (PMIC interrupt fix,
V2.3 testing), ggggcodes (V2.3 reverse engineering), Mike Knights /
mcknight89 (V1.4 testing).
