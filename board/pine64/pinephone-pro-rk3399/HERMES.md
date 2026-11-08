# PinePhone Pro: stay off when a charger is plugged in

PinePhone Pro (RK3399 + RK818 PMIC) inside the PinePhone keyboard case. Whenever
5 V appears on USB-C, the RK818 powers the phone on, so plugging in a charger
always booted the OS. With the keyboard case and charger attached, the 5 V also
cycles about every 20 s for a while after a power-off, which made the phone boot
again and again.

This branch enables `CONFIG_ROCKCHIP_RK8XX_DISABLE_BOOT_ON_POWERON` in
`pinephone-pro-rk3399_defconfig` and extends `rk8xx_off_for_plugin()` for the RK818.

**To turn the phone on with a charger or the keyboard case attached, press and
hold the power key until the LEDs blink white (about 1-2 s). The phone then
boots. Only the power key can start it this way.**

On a plug-in start (RK818 ON_SOURCE 0xae has bit 0x40), U-Boot:

1. boots if the power key is held. It reads the key through gpio-keys "Power"
   (GPIO0_A5, via CONFIG_BUTTON_GPIO) and records `pwrkey-held-at-start`;
2. otherwise blinks all LEDs white once, then watches the power key for 2 s
   more and boots on a press, recorded as `pwrkey-gpio-after-blink`;
3. powers off if the key was not pressed.

The record goes to the env as `hermes_wake`, plus `hermes_wake_on` (the
ON_SOURCE value) and `hermes_key_ms` (ms from power-on to the moment the key
was seen). It is written only when a key press rescues a plug-in start.

Why the power key needs the GPIO: with 5 V present, a power-key start leaves
ON_SOURCE at 0x40 (plug-in) and never sets the key bit 0x80. The PWRON
interrupt in INT_STS1 is masked as well. Checking only the RK818 therefore
treated a key press as a plug-in and powered off.

Tested on 2026-11-08, in the keyboard case with the charger attached:

| Action | Result |
|---|---|
| Hold the power key ~3 s | boots (`pwrkey-held-at-start`) |
| Press the power key ~1 s, then again at the blink | boots (`pwrkey-gpio-after-blink`, key seen at 2990 ms) |
| No key | stays off; the blinks stop once the 5 V settles |

Out of the case, with no charger, the power key starts the phone as usual
(ON_SOURCE 0x80).

## Build and flash (natively on the phone, Debian trixie arm64)

    sudo apt install build-essential bison flex bc libssl-dev libgnutls28-dev \
        uuid-dev python3-dev python3-setuptools python3-pyelftools swig \
        arm-trusted-firmware mtd-utils
    make pinephone-pro-rk3399_defconfig
    make -j6 BL31=/usr/lib/arm-trusted-firmware/rk3399/bl31.elf
    sudo dd if=/dev/mtd0ro of=spi-backup.bin bs=64k     # back up the SPI first
    sudo /usr/sbin/flashcp -v u-boot-rockchip-spi.bin /dev/mtd0

If the phone no longer boots, enter maskrom (the RE button) and write
`spi-backup.bin` back with rkdeveloptool.

## Linux side: log the boot reason

`/usr/local/sbin/pmic-boot-reason`, run once per boot by a oneshot unit:

    #!/bin/sh
    modprobe i2c-dev 2>/dev/null
    on=$(i2cget -f -y 0 0x1c 0xae)
    off=$(i2cget -f -y 0 0x1c 0xaf)
    wake=$(dd if=/dev/mtd0ro bs=4096 skip=$((0x3F8000 / 4096)) count=8 status=none |
        tr '\0' '\n' | grep -a -E '^hermes_(wake|wake_on|key_ms)=' | tr '\n' ' ')
    echo "PMIC RK818 boot reason: on=$on off=$off; U-Boot record: ${wake:-none}"
