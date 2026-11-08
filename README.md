# rafael2k/u-boot: everyday U-Boot for the PinePhone Pro

**This fork's `main` is what we flash and use every day on the PinePhone
Pro.** It's upstream U-Boot `main` with one change for the PinePhone Pro:
plugging in a charger, or the keyboard case's 5 V, no longer turns the phone on.

- Tested on 2026-11-08 on a PinePhone Pro in the PinePhone keyboard case with a
  charger, built from this branch (`U-Boot 2027.01-rc1-g2e9e0836c358`):
  normal boot, staying off with the charger attached, and starting with the
  power key.
- To turn the phone on with a charger or the keyboard case attached, hold the
  power key for about 3 s. If you let go earlier, the LEDs blink white once;
  press the power key again within 2 s and it boots. Without power key, it
  stays off and keeps charging.
- How it works, how to build and flash it, and how to recover:
  [board/pine64/pinephone-pro-rk3399/HERMES.md](board/pine64/pinephone-pro-rk3399/HERMES.md).
- Build: `make pinephone-pro-rk3399_defconfig && make BL31=.../rk3399/bl31.elf`,
  then flash `u-boot-rockchip-spi.bin` to the SPI flash. Back up the SPI first.

Upstream U-Boot's own documentation is in [README](README) and [doc/](doc/).
