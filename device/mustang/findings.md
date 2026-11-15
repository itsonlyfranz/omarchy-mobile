<!-- SPDX-License-Identifier: MIT -->
# Pixel 10 Pro XL (mustang): what the factory image's device trees say about the platform

Written 2026-11-16 for [#18](https://github.com/derekross/omarchy-mobile/issues/18) and [#816](https://github.com/derekross/omarchy-mobile/issues/816). It builds on [#198](https://github.com/derekross/omarchy-mobile/pull/198) (`NOTES.md` parts 1 to 5) and does not repeat it: the image list, header versions, partition names, slot flags and the first compatible-string comparison are there. This adds what that file leaves open about the GPU model, the UART, the interrupt controller and the clocks: the GPU, the console UART, the interrupt controller's fourth cell, how clocks and power domains are described, and which kernel modules do the work on stock. The same facts, in `NOTES.md`'s format, are in its "part 6".

**Nothing here comes from a phone.** Every fact is read from Google's factory image on a laptop, offline. Nothing was flashed, booted or run on a device, and the author has no Pixel 10 yet. Inferences are in their own section and are labelled as such. The commands are listed so any line can be re-checked.

## Source

- **Image:** `mustang-cp3a.261005.005-factory-099acc44.zip`, from `dl.google.com`, SHA-256 `099acc44e2f87fe2735d92952081d64550ff08b4a6390bb6b001c8451695657e` (matches the value in `NOTES.md` part 1). It is the build #198 used, so the two can be cross-checked. A newer build may exist; the factory-images page is rendered by script and could not be listed from the command line, so "latest" was not confirmed.
- **Read:** `boot`, `init_boot`, `vendor_boot`, `vendor_kernel_boot`, `dtbo`, and from the inner zip `vendor_dlkm` and `vendor` (file names and module metadata only).
- **Tools,** none installed with root: `unpack_bootimg`, `mkdtboimg` and `avbtool` fetched as Python scripts from AOSP's `platform/system/tools/mkbootimg`, `platform/system/libufdt` and `platform/external/avb`; `dtc`/`fdtoverlay` 1.8.1 (Homebrew); `lz4`, `cpio` and 7-Zip for the ramdisks and the ext4 images; a 60-line Python DTB reader to resolve phandles.
- **Merged tree:** the B0 base DTB with the MP board overlay (`dtbo` entry 30), as in `NOTES.md` part 5: 710 top-level nodes, 277 distinct compatible strings. The A0 tree differs only in per-revision strings, as #198 found.
- **Reference:** `gs201.dtsi` from the `omamobile` branch, fetched 2026-11-16 (2,032 lines; #198 compared against a 1,163-line version).

```
unpack_bootimg --boot_img X.img --out DIR              # header v4 for all four; cmdline, dtb section
split the dtb section on 0xd00dfeed -> 2 DTBs (A0, B0)
fdtoverlay -i 01.dtb -o m01.dtb dtbo_entry.30         # B0 + MP
lz4 -dc vendor_ramdisk | cpio -id                      # modules.load, firmware names
7zz x vendor_dlkm.img                                  # module files and their metadata
strings pvrsrvkm.ko | grep -E '^(description|alias|depends)='   # module metadata only
```

## Findings

### 1. Clocks come from the CPM firmware over a mailbox, not from clock-controller registers

- **One provider carries most of them.** `/cpm_clk` (`google,cpm-clk`) has 152 child nodes, each a clock with `#clock-cells = <0>`, a `clock-id`, an `lpcm-id` (the clock domain) and a `rates` list. The names are grouped by subsystem: 46 `lsios_*`, 42 `lsioe_*`, 36 `lsion_*`, 14 `hsion_*`, 6 `hsios_*`, 5 `dpu_*`, 2 `gpcm_*`, 1 `cpu_*`. The node has `mboxes = <&goog_mba_cpm_iface>` and a `power-domains` list of six subsystem wrappers. About 207 `clocks` references in the tree point at it.
- **The mailbox is a set of MBA channels.** `/goog_mba_cpm_iface` (`google,mba-cpm-iface`) lists 12 mailbox channels: 8 request and 4 response, `payload-size = <4>`. It sits on `cpm_ap_ns_req_mba_client_0@5260000` and `cpm_ap_ns_resp_mba_client_1@5261000` (`google,mba-ctrl`). The tree has 41 `google,mba-ctrl` clients in all, for the CPM, AoC, GSA, GDMC and the Aurora (TPU) cores. `google,cpm-irq` (`#interrupt-cells = <2>`), `google,dvfs-helper`, the power controller, UFS (`mba-pmic-dest-channel`) and `thermal-cpm-mbox` also use `mboxes`.
- **A smaller set is plain MMIO.** Five `google,google-lcm` nodes (`lsios`, `lsioe`, `lsion`, `hsion`, `hsios`) have `reg-names = gate_r, gate_nr, mux, div`, `#clock-cells = <3>` and `#reset-cells = <1>`. One `google,hsion-lpcm` has `#clock-cells = <1>`. Nine `*_pf_state_clk` nodes under `/pf_state_cpm`, plus the GPU's `google,lga-gpu-pf-state`, are a third kind (the tree says only "pf_state"; the module is `pf_state_cpm`); the rest are `fixed-clock` (`oscillator`, a 38.4 MHz oscillator, `usb_*_clk`, `dma_clk`, `dhub_clock`).
- **Stock modules that implement it** (names from `vendor_kernel_boot`'s `modules.load`): `goog-mba-ctrl`, `goog-mba-cpm-iface` (`description=` set), `clk-cpm` ("Google CPM Clock Driver", depends on `goog-mba-cpm-iface`), `rst-cpm` ("Google CPM Reset Driver"), `irq-google-cpm`, `dvfs_helper`, `google_cpufreq`, `pf_state_cpm`, `gpu_pf_state`.

### 2. Power domains: one controller behind the same mailbox, plus PSCI for CPUs

- **`/power_controller`** (`google,lga-power-controller`, `mboxes = <&goog_mba_cpm_iface>`) has 41 domain children, in two levels: *subsystem wrappers* with a `sswrp-id` (`sswrp_dpu_pd`, `sswrp_hsio_n_pd`, `sswrp_hsio_s_pd`, `sswrp_lsio_{n,s,e}_pd`, `sswrp_gpu_pd`, `sswrp_aoc_pd`, `sswrp_tpu_pd`, `sswrp_aurora_pd`, `sswrp_isp{fe,be}_pd`, `sswrp_g2d_pd`, `sswrp_eh_pd`, `sswrp_gpdma_pd`, `sswrp_gsw_pd`, `sswrp_codec_3p_pd`) and *sub-domains* of them with a `subdomain-id` (for example `hsio_s_ufs_pd` under `hsio_s_ufs_prep_pd` under `sswrp_hsio_s_pd`; `hsio_n_usb_pd`, `hsio_n_pcie_pd`, `dpu_fe0_pd`, `gpu_core_logic_pd`).
- **Per-domain flags** are properties: `on-at-init`, `rpm-always-on`, `no-auto-resume`, `active-wakeup`, `irq-safe`, `use-smc` (two domains, both `lsio_e`).
- **CPUs** use `arm,psci-1.0` over SMC, with eleven `power-domain-cpuN`/cluster nodes under `/psci` and `domain-idle-states`. That part is the standard binding.
- **Stock modules:** `power_controller` (depends on `goog-mba-cpm-iface`), `gs_domain_idle`, `google_ip_idle_notifier`.

### 3. Interrupt controller: GICv3 with a four-cell specifier, and a second layer of aggregators

- **GIC.** `arm,gic-v3` at `0x5880000` (distributor, 64 KiB) with redistributors at `0x5900000` (2 MiB); `reg` has five pairs and the last three are zero. No ITS node. The maintenance interrupt is PPI 9.
- **The fourth cell is the PPI-partition phandle.** Of 231 interrupt specifiers whose parent is the GIC, 229 have a fourth cell of 0. The two non-zero ones are the CPU PMUs (`/arm_pmu-cpu0-6`, `/arm-pmu-cpu7`), whose fourth cell is the phandle of `ppi-partitions/interrupt-partition-0` (affinity: seven CPUs) and `interrupt-partition-1` (affinity: one CPU, the single large core). That is the standard GICv3 binding for 4-cell specifiers (a partition for PPI affinity), not a Google extension. The CPU PMU is split in two nodes (`arm,armv8-pmuv3` and `arm,cortex-x4-pmu`) for that reason.
- **Almost no peripheral interrupt goes to the GIC directly.** 181 nodes have a GIA as `interrupt-parent`. There are 116 GIA nodes (90 `google,level-gia`, 23 `google,pulse-gia`, 3 `google,wide-level-gia`), each a small `interrupt-controller` with its own `reg` and `#interrupt-cells = <1>`; 115 have `interrupts`, 83 name another GIA as `interrupt-parent` (so they nest), and 100 have a `power-domains` entry. The GPU alone has three (`gpu-level-aggr-3`, `gpu-level-aggr-11`, `gpu-pulse-aggr-4`). Stock modules: `irq-gia-lib` ("Generic Interrupt Aggregator Library"), `irq-gia-google`.
- **Pin controllers are interrupt controllers** (`#interrupt-cells = <2>`, `interrupts-extended`, an `irq-pad-mapping` table); `google,cpm-irq` is another, reached over the mailbox.
- **SPI numbers** in the tree run from 0 to 953 and are used as written. `NOTES.md` already warns against copying numbers from gs201; this is why.

### 4. Pin control: eight domain controllers under one driver

`google,lga-{aoc,cpm,gdmc,hsion,hsios,lsioe,lsion,lsios}-pinctrl` (plus `hsios-stby` and `dpu`, both small), one driver (`pinctrl.ko`, "Google LGA PINCTRL Driver"). Most have `gpio-controller`, `gpio-ranges`, `gpio-line-names`, a `gpio_clk` clock, and `power-domains` with names `csr_access_pd` or `pad_function_pd`. The `aoc` controller has `aoc-ssr-capable` and a `shared-pads` table. `gpio-line-names` carry board net names for most pins, a ready-made pin map; none are listed here.

### 5. GPU: Imagination, Volcanic generation, out-of-tree driver

- **Node.** `/gpu0@34800000`, `compatible = "google,lga-gpu", "img,gpu"`, eight register windows (`rgx`, `pfsm`, `req_pbha_lut`, `ra_lut_0..2`, `mba_glue`, `mba_client`), one interrupt, two power domains, one clock (`gpu_pf_state`), two IOMMU stream IDs on a dedicated `arm,smmu-v3` (`/gpu_smmu`, `0x34c00000`), `init-frequency-hz` 300 MHz, `memory-region`, `cooling-max-state = <15>`, and `dvfs-helper-domain-name = "gpu"`.
- **Generation.** The module is `pvrsrvkm.ko` ("Pixel PowerVR GPU Driver", alias `of:*Cgoogle,lga-gpu`, `platform:rgx`; author Imagination Technologies, license "Dual MIT/GPL"). Its embedded source paths are `private/google-modules/powervr/pvrsrvkm/generated/volcanic/*` and `.../pixel/{of,sysconfig,dvfs_governor,fence_manager}.c`. Imagination's Volcanic is the generation after Rogue. The firmware files in `vendor` are `rgx.fw.signed.71.2.2448.1212` and `rgx.sh.71.2.2448.1212`, which by the driver's naming is BVNC 71.2.2448.1212. User space is `vulkan.powervr.so`, `libGLESv2_powervr.so`, `libEGL_powervr.so` and `libPVROCL.so`. That settles the GPU family; the marketing name of the core is not stated anywhere read.
- **Depends** (module metadata): `pixel_stat_sysfs`, `pixel_stat_mm`, `systrace`, `dvfs_helper`, `google_gtc`, `google_vote_manager`, `sbb-mux`, `slc_pt`, `iif`, `trusty-ipc`. The last matters: the GPU module depends on Trusty IPC.
- **It is a different driver from the display.** Display is VeriSilicon (`vs_drm`, `verisilicon,dc9x00`, per #198), and `drm_g2d` and `google_gem` are separate modules.

### 6. Console and the other buses

- **Console.** `/chosen/bootargs` has `earlycon=uart8250,mmio32,0xdb62000`, `console=ttyS0,115200n8` and `google_8250.nr_uarts=4`. `0xdb62000` is the `reg` of `/uart@db61000`, which is `serial0`, has an interrupt (SPI 688) and `status = "okay"`. The node name's unit address (`db61000`) is 0x1000 below `reg` (`db62000`); that holds for all 22 UARTs. A second UART, `/uart@3a321000` (`serial2`), is also enabled.
- **UART driver.** `goog,goog-dw-apb-uart`, with `reg-shift = <2>` and `reg-io-width = <4>`. The stock module `google_8250_dw` (a Synopsys DesignWare 8250 driver) also lists `snps,dw-apb-uart` among its aliases, on top of `google_8250` and `google_8250_base`.
- **22 serial engines in one layout.** The 22 base addresses repeat across `uartN`, `spiN`, `i2cN` and `i3cN` aliases (`/uart@3bf01000`, `/spi@3bf01000`, `/i2c@3bf01000`, `/i3c-master@3bf01000`, …). Stock modules: `i2c-designware-*`, `dw-i3c-master`, `spi-dw`, `spi-dw-mmio`.
- **Storage.** `/ufs@3c400000`, `google,ufshc`, with `ref_clk`/`aux_clk`, a `phy_rst` reset, reset and power GPIOs, a PHY patching mode, `ip-idle-index`, and four supplies (`vdd0p75`, `vdd1p2`, `vcc`, `vccq`). Driver: `ufs.ko`, which depends on `sbb-mux`, `gsa`, `da9188-core`.
- **USB.** `google,dwc3-lga` wraps a standard `snps,dwc3` at `0xc400000`; the PHY is `google,usb_phy-lga` at `0xc410000` with a firmware blob (`usb_phy_fw.bin`) loaded from the `vendor_boot` ramdisk. The type-C controller is on SPMI (`max77759tcpc-spmi`). An eUSB2 repeater is on I2C.
- **PCIe.** Two `google,lga-pcie` controllers (`0xc500000`, `0x3c500000`).
- **IOMMU.** Seven `arm,smmu-v3` nodes (generic `arm_smmu_v3`, plus `arm_smmu_v3_kvm` on stock).
- **DMA.** `arm,pl330` plus `google-gpdma`.

### 7. PMICs are on SPMI, and no S2MPG is present

No node or module with an `s2mpg` name exists. There is an SPMI controller `/spmi@53f1000` (`smartdv,spmi`, driver `spmi-smartdv`) with the `max77779` family (PMIC, charger, fuel gauge, VIMON, SP, I2C master) and the `max77759` type-C controller as children. `/da9188mfd` (`google,da9188mfd`) has regulator, RTC, GPIO and virtual-GPIO children. A `google,bitbang-spmi-controller` (`/s5910_bb`) talks to the modem's PMIC. An `rt6160` buck-boost sits on I2C, and the module list also names `s2dos07x02-regulator` (a Samsung part, the only one) and `tps628600-regulator`. So the S2MPG family that the Pixel 7 and Pixel 9a work uses is not here.

### 8. CPU and frequency

Eight CPUs, as #198 found, each with `dvfs-helper-domain-name` (`cpu0`, `cpu1g0`, `cpu1g1`, `cpu2`), `operating-points-v2`, and a `psci` power domain. Frequency changes go through `google,dvfs-helper` over the CPM mailbox (`google_cpufreq`, `google_dsufreq`, `google_devfreq`). The base tree's memory node is a 256 MiB placeholder at `0x80000000`; the bootloader adds the rest (#198 part 5).

### 9. Boot arguments and the modules that load before everything else

- **`vendor_boot`'s command line** (on cheetah the bootloader ignores it and `/chosen/bootargs` is used, per the Pixel 7 Pro notes; whether mustang's does is unknown): `fips140.load_sequential=1 vh_sched.load_sequential=1 spmi_smartdv.load_sequential=1 regmap-goog-spmi.load_sequential=1 max77779_pmic*.load_sequential=1 …`, `aoc_core.aoc_enable_gsa_boot=1`, `arm_smmu_v3_kvm.smc_s2=true`, `swiotlb=noforce`, `rodata=on`, `rcu_nocbs=all`, and `bootconfig`. `/chosen/bootargs` adds `kvm-arm.protected_modules=goog-mba-ctrl,pinctrl,power_controller,irq-gia-google,arm_smmu_v3_kvm`: the mailbox client, pin controller, power controller, GIA and SMMU drivers are loaded as protected modules under pKVM.
- **Counts.** `vendor_kernel_boot`'s ramdisk loads 203 modules; the 16K-page `vendor_boot` ramdisk loads 376; `vendor_dlkm` loads 117 more, with a parallel `16k-mode/` set. The first group is the platform core (the mailbox, clock, reset, IRQ, pinctrl, power-controller, UART, UFS, USB, SMMU, PMIC, panel and audio-DSP drivers above).

## What this means for a port (inferences, not tested)

1. **The first real driver is probably a mailbox client for the CPM, not a clock controller in the Exynos sense.** Clocks, resets, power domains, DVFS and one interrupt chip all depend on `goog-mba-cpm-iface`. Until that exists, almost nothing past the CPUs and the UART can be powered. Its message format is in Google's GPL module source (the `private/google-modules/...` tree the strings point at), which has to be read, not guessed; nothing here shows that it resembles gs201's ACPM.
2. **The second blocker is the GIA layer.** Most peripheral interrupts are behind one of 116 small aggregators. A first boot that only needs the GIC, the arch timer and one UART can skip it; anything with a driver cannot.
3. **The console UART may come up early.** `snps,dw-apb-uart`, `reg-shift = <2>`, `reg-io-width = <4>` is what mainline's DesignWare 8250 driver takes; the stock driver lists that string as an alias, so a `snps,dw-apb-uart` fallback in a first DT is a reasonable thing to try. Its clock is a CPM clock, though, so it works with the bootloader's setting only as long as nothing re-gates it. Not tried.
4. **GPU.** Volcanic is a different Imagination generation from Rogue. To my knowledge, mainline `drm/imagination` covers Rogue only; whether Volcanic is planned upstream and whether Mesa's PowerVR driver has anything for it should be checked before planning on either, and the answer decides whether the long-term target is software rendering. The GPU module's dependency on Trusty IPC and an SLC page-table module (`slc_pt`) also suggests the firmware path is not self-contained. Treat all of that as a question for the maintainer.
5. **Directory and naming.** Nothing in the tree is Samsung-derived; the compatibles are `google,lga-*`, which supports the maintainer's remark that it should not go under `exynos/google/`.
6. **The S2MPG PMIC work of the other Pixels does not transfer;** the `max77779`/`da9188` family is what laguna has.

## Still unknown

- **How the CPM mailbox protocol works.** Needs the GPL module source, not the binary.
- **The GPU's model name.** Only the BVNC from the firmware file name.
- **Which silicon revision the phone boots** (the live node set fits B0 better than A0, but that is an inference from node names) and the live values (needs root).
- **Whether `vendor_boot`'s command line is applied** by this bootloader.
- **The 16K kernel.** Not read here.
- **Whether a newer factory build changes any of it.** The Android 17 build read here is the one #198 used.
