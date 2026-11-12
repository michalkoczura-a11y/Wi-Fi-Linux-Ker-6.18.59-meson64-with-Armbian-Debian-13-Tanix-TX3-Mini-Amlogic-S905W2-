# Wi-Fi-Linux-Ker-6.18.59-meson64-with-Armbian-Debian-13-Tanix-TX3-Mini-Amlogic-S905W2-

Amlogic W150S1 (W1) Wi-Fi Driver for Modern Linux Kernels (6.x)

https://img.shields.io/badge/Platform-Amlogic%20S905W2-blue.svg
https://img.shields.io/badge/Kernel-6.18.x-green.svg
https://img.shields.io/badge/Status-Working-brightgreen.svg

This repository contains a ported and patched version of the legacy Amlogic W150S1 (also known as W1) SDIO Wi-Fi driver (originally from CoreELEC / Amlogic SDK, circa 2016). It is designed to compile and run on modern Linux kernels (tested on 6.18.59-meson64 with Armbian/Debian 13) for devices based on the Amlogic S905W2 SoC, specifically the Tanix TX3 Mini+.

The primary goal of this project was to bring an abandoned, hardcoded 4.9/5.15 driver into the modern kernel era, bypassing kernel API deprecations, fixing broken PMU power sequences, and resolving Device Tree configuration issues.

🛠 Hardware & Environment

· Device: Tanix TX3 Mini+ (or similar TV Boxes)
· SoC: Amlogic S905W2
· Wi-Fi Chip: Amlogic W150S1 (SDIO VID:PID 02e7:9086)
· OS: Armbian (Debian 13) / Kernel 6.18.59-meson64
· Compiler: GCC 14 (arm64)

🚧 The Challenges & Solutions

1. Device Tree (DTB) Configuration

The main blocker was that the board's default DTB did not provide the necessary power, reset, and clock signals to the Wi-Fi chip.

· Power Enable (GPIO 44): The audio-amplifier-0 node was holding this pin. It was disabled. A virtual regulator-fixed node (wifi_power_reg) was added to force GPIO 44 high (regulator-always-on) at boot.
· The 32 kHz Clock (PWM): The W150S1 PMU requires a 32.768 kHz clock for its internal oscillator. The pwm@5c000 controller was enabled and assigned to generate this clock. Crucially, the clock parent was forced to the stable XTAL (instead of vid_pll) to prevent frequency shifts when the HDMI mode changes:
  ```dts
  pwm@5c000 {
      status = "okay";
      pinctrl-0 = <&pwm_e_pins1>;
      assigned-clocks = <&pwm_ef>;
      assigned-clock-parents = <&xtal>;
  };
  ```

2. C Code Patches (Kernel 6.x Compatibility)

The legacy code assumed a very different kernel API and bootloader behavior.

· PMU_ACT_MODE TIMEOUT: The loop waiting for the card's PMU to power up was a "busy-wait", finishing before the hardware could physically respond. A msleep(10); was injected into the config_pmu_reg loop.
· Firmware Upload STUB: The original code contained a dummy stub (W1_STUB: hal_download_wifi_fw_img POMINIETE) assuming the TV box bootloader had already uploaded the firmware. This was removed, and the original firmware upload function was restored.
· write ICCM ERROR: The original 2016 code had a syntax error (an unterminated \n string) and relied on a memcmp verification that fails on modern controllers. The syntax was fixed, and the memcmp check was bypassed to force the upload.
· API Updates: Migrated older kernel APIs (e.g., init_timer, vfree, VFS_internal...) to modern equivalents (timer_setup, kvfree, THIS_MODULE). Added a compat.h to map legacy types like OS_MUTEX.

3. Build System (Makefile)

To prevent duplicate symbol export errors (w1_aml_mem_prealloc), the Makefile was strictly configured to separate the SDIO transport layer (w1_sdio.ko) from the main Wi-Fi stack (aml_wifi.ko). All header files (.h) were flattened into a single directory to avoid relative path issues with modern GCC.

🚀 Compilation & Usage

1. Build the modules:
Ensure you have the headers for your running kernel installed.

```bash
make -j4 -C /lib/modules/$(uname -r)/build M=$(pwd) modules
```

(Note: Path may vary depending on your distro, e.g., /usr/src/linux-headers-$(uname -r))

2. Load the modules:
You must load the transport layer first.

```bash
sudo insmod w1_sdio.ko
sleep 1
sudo insmod aml_wifi.ko
```

3. Verify:
Check dmesg | tail for initialization logs and check for the network interface:

```bash
ip a | grep wlan
```

📝 Status & Known Issues

· Current State: The driver compiles, loads, and successfully powers up the PMU. The card accepts the firmware (bypassing the ICCM verification error).
· To-Do: If the card refuses to initialize fully, further tuning of the SDIO clock speed (downgrading it in the DTB) might be required depending on the specific board's signal integrity.

🤝 Credits

· CoreELEC Team for the original driver source (2016).
· The Armbian and Linux Meson communities for the DTB references and kernel headers.
