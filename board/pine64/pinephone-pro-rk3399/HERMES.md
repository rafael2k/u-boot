# PinePhone Pro: stay off when a charger is plugged in

PinePhone Pro (RK3399 + RK818 PMIC) inside the PinePhone keyboard case. Whenever
5 V appears on USB-C, the RK818 powers the phone on, so plugging in a charger
always booted the OS. With the keyboard case and charger attached, the 5 V also
cycles about every 20 s for a while after a power-off, which made the phone boot
again and again.

This branch enables `CONFIG_ROCKCHIP_RK8XX_DISABLE_BOOT_ON_POWERON` in
`pinephone-pro-rk3399_defconfig` and extends `rk8xx_off_for_plugin()` for the RK818.

**To turn the phone on inside the keyboard case, hold the power key for about
3 s. Pressing Volume Up or Volume Down right after a white blink also works.**

On a plug-in start (RK818 ON_SOURCE 0xae has bit 0x40), U-Boot:

1. boots if the power key is held. It reads the key through gpio-keys "Power"
   (GPIO0_A5, via CONFIG_BUTTON_GPIO) and records `pwrkey-held-at-start`;
2. otherwise blinks all LEDs white once, then watches the power key GPIO and
   the volume keys (SARADC channel 1) for 2 s;
3. boots on a key press and records which key it was;
4. powers off if no key was pressed.

The record goes to the env as `hermes_wake` (`pwrkey-held-at-start`,
`pwrkey-gpio-after-blink`, `volup-after-blink` or `voldown-after-blink`), with
`hermes_wake_on` holding the ON_SOURCE value.

Why the power key needs the GPIO: with 5 V present, a power-key start leaves
ON_SOURCE at 0x40 (plug-in) and never sets the key bit 0x80. The PWRON
interrupt in INT_STS1 is masked as well. Checking only the RK818 therefore
treated a key press as a plug-in and powered off.

Tested on 2026-11-08, in the keyboard case with the charger attached:

| Action | Result |
|---|---|
| Hold the power key ~3 s | boots (`pwrkey-held-at-start`, ON_SOURCE 0x40) |
| Volume Up or Volume Down right after the blink | boots |
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
        tr '\0' '\n' | grep -a -E '^hermes_wake(_on)?=' | tr '\n' ' ')
    echo "PMIC RK818 boot reason: on=$on off=$off; U-Boot record: ${wake:-none}"
