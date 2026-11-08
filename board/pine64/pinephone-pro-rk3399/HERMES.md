# PinePhone Pro: stay off when a charger is plugged in

PinePhone Pro (RK3399 + RK818 PMIC) inside the PinePhone keyboard case. Whenever
5 V appears on USB-C, the RK818 powers the phone on, so plugging in a charger
always booted the OS. With the keyboard case and charger attached, the 5 V also
cycles about every 20 s for a while after a power-off, which made the phone boot
again and again.

This branch enables `CONFIG_ROCKCHIP_RK8XX_DISABLE_BOOT_ON_POWERON` in
`pinephone-pro-rk3399_defconfig` and extends `rk8xx_off_for_plugin()` for the RK818.

**To turn the phone on inside the keyboard case: wait for the white blink, then
press Volume Up or Volume Down. The power key does not work there.**

On a plug-in start (RK818 ON_SOURCE 0xae has bit 0x40), U-Boot:

1. blinks all LEDs white once, then watches the volume keys (SARADC channel 1)
   for 2 s;
2. boots on a volume key press, and saves the key in the env as `hermes_wake`
   (`volup-after-blink` or `voldown-after-blink`) plus `hermes_wake_on` (the
   ON_SOURCE value);
3. powers off if no key was pressed.

Tested on 2026-11-08, in the keyboard case with the charger attached:

| Action | Result |
|---|---|
| Volume Up or Volume Down right after the blink | boots |
| No key | stays off; the blinks stop once the 5 V settles |
| Power key, before or after the blink | **does not work**: the phone stays off |

Out of the case, with no charger, the power key starts the phone as usual
(ON_SOURCE 0x80).

The code also has two power-key paths, and neither has worked in testing:

- It boots if ON_SOURCE also has the power-key bit (0x80).
- It watches RK818 INT_STS1 PWRON during the 2 s window.

The cause is unknown. Either the RK818 doesn't report the key while 5 V is
present, or it reports it somewhere else.

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
