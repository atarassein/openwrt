# NSS offload for the ASUS / TP-Link OnHub (OpenWrt 25.12.6)

This branch (`ipq806x-nss-25.12.6`, based on tag `v25.12.6`, Linux 6.12.112)
adds Qualcomm NSS (Network SubSystem) hardware offload to the
`ipq806x/chromium` subtarget, i.e. the ASUS OnHub SRT-AC1900 and the
TP-Link OnHub AC1900. The two NSS UBI32 cores then do routing/NAT, so the
Krait CPUs are mostly idle under wired load.

Other ipq806x boards (`ipq806x/generic`) keep the stock DSA + stmmac setup
and get no NSS device tree or packages. The NSS kernel patches are
target-wide, so those boards get them too. They are mostly inert hooks, with
two behaviour changes: bridge hairpin mode is limited to Wi-Fi ports, and
with `br_netfilter` built (as ECM requires) bridge-family netfilter hooks
only run while one of the `net.bridge.bridge-nf-call-*` sysctls is on.

## Why the source and not the .bin

NSS cannot be added to a finished `sysupgrade.bin`. It needs core kernel
changes (conntrack, bridge, PPP, neighbour/route notifiers, clock and
regulator drivers), a different Ethernet driver (`qca-nss-gmac` instead of
stmmac), a new device tree and the NSS firmware. All of that lives in the
vboot-signed FIT kernel, so the kernel has to be rebuilt anyway, and then
the image is too.

## What changed

| Area | Change |
| --- | --- |
| Kernel | 74 NSS/ECM patches, `target/linux/ipq806x/patches-6.12/990-*` |
| Kernel config | `config-6.12`: `REGULATOR_NSS_VOLT=y`, defaults for the two new conntrack symbols. `chromium/config-default`: swconfig + `AR8216_PHY` |
| Packages | `package/qca-nss/`: qca-nss-gmac, qca-nss-drv (+ NSS.AK 11.2 firmware), qca-nss-ecm, qca-nss-clients, qca-nss-crypto, qca-nss-cfi, qca-mcs, nss-ifb, nssinfo/libnl-nss |
| Device tree | new `qcom-ipq8064-nss.dtsi` (NSS cores, crypto engines, nss-gmac common). `qcom-ipq8064-onhub.dtsi`: QCA8337 back on swconfig, both GMACs on the `qcom,nss-gmac` binding |
| Network defaults | `02_network`: `eth0` = WAN, `eth1` = LAN, swconfig `switch0`, untagged CPU ports |
| Image | OnHub images pull in `swconfig kmod-qca-nss-gmac kmod-qca-nss-drv kmod-qca-nss-drv-pppoe kmod-qca-nss-ecm-standard`, compat version 1.0 |
| Wi-Fi | mac80211 NSS redirect (`PACKAGE_MAC80211_NSS_SUPPORT`, on by default when NSS is selected), so ECM can offload Wi-Fi to wired flows |
| tc | iproute2 NSS qdisc/`nssmirred` support, for NSS shaping |

Switch wiring (unchanged hardware, new driver):

```
gmac0 (RGMII) = eth0 --- switch port 0 --- port 2 = WAN
gmac2 (SGMII) = eth1 --- switch port 6 --- port 1 = LAN
```

## Build

You need a Linux build host (Debian/Ubuntu, or WSL2) with the usual
OpenWrt build dependencies. It does not build on plain Windows.

```bash
./scripts/feeds update -a
./scripts/feeds install -a
cat > .config <<'EOF'
CONFIG_TARGET_ipq806x=y
CONFIG_TARGET_ipq806x_chromium=y
CONFIG_TARGET_ipq806x_chromium_DEVICE_asus_onhub=y
CONFIG_PACKAGE_kmod-qca-nss-gmac=y
CONFIG_PACKAGE_kmod-qca-nss-drv=y
CONFIG_PACKAGE_kmod-qca-nss-drv-pppoe=y
CONFIG_PACKAGE_kmod-qca-nss-ecm-standard=y
CONFIG_PACKAGE_MAC80211_NSS_SUPPORT=y
# optional extras, as tested on the C2600 port:
# CONFIG_PACKAGE_kmod-qca-nss-drv-qdisc=y
# CONFIG_PACKAGE_kmod-nss-ifb=y
# CONFIG_PACKAGE_kmod-qca-nss-crypto=y
# CONFIG_PACKAGE_kmod-qca-nss-cfi-cryptoapi=y
# CONFIG_PACKAGE_kmod-qca-mcs=y
CONFIG_PACKAGE_luci=y
EOF
make defconfig
make download -j8
make -j"$(nproc)"
```

For the TP-Link model use `..._DEVICE_tplink_onhub=y` instead.
Images are written to `bin/targets/ipq806x/chromium/`.

## Flash

1. Back up anything you need. The DSA config of the official image cannot be
   carried over, so sysupgrade refuses to keep settings
   ("Config cannot be migrated from DSA to swconfig (NSS)").
2. Copy `openwrt-ipq806x-chromium-asus_onhub-squashfs-sysupgrade.bin` to
   `/tmp` on the router and run `sysupgrade -n /tmp/<file>`.
3. After boot, LAN is `192.168.1.1` on the LAN port as usual.

If it does not come back, boot an official `factory.bin` from USB in
developer mode (see the OpenWrt OnHub wiki page) and reflash.

Going back to an official build also needs `sysupgrade -n`.

## Check that NSS is working

```bash
dmesg | grep -i -E 'nss|gmac'
swconfig dev switch0 show
cat /sys/kernel/debug/qca-nss-drv/stats/cpu_load_ubi
cat /sys/kernel/debug/ecm/ecm_db/connection_count_simple
```

Under a wired download through NAT, the ECM connection count should rise
and `top` should show the CPUs mostly idle.

## What has been verified (and what not)

Checked on a Windows host without a Linux toolchain:

- linux-6.12.112 (hash matches `target/linux/generic/kernel-6.12`) plus the
  full 25.12.6 stack (backport, pending, hack, ipq806x) plus the 74 NSS
  patches apply with no rejects and no fuzz.
- The qca-nss-gmac and qca-nss-ecm patch sets apply cleanly to the pinned
  QSDK commits.
- Both OnHub device trees preprocess against the patched kernel headers,
  and every `&label` and macro resolves. This used a cpp/syntax checker, not
  `dtc`.
- The NSS firmware blobs are byte-identical to the ones in asvio's trees
  (`NSS.AK.11.2-9-002-R`).

Not done: compiling the kernel or packages, building an image, booting it.
The first `make` on a Linux host is the real compile test.

## Behaviour to be aware of

- **Not tested on an OnHub.** The kernel and package code comes from a port
  that is built and running on a TP-Link C2600 (same SoC). The OnHub
  device tree and network layout are new and have not been booted.
- **MAC addresses**: nss-gmac takes `local-mac-address` from the device
  tree, which the OnHub bootloader may fill in. If it doesn't, the MACs are
  random on every boot; set `option macaddr` on the `wan`/`lan` devices.
- **WAN link state**: the GMACs run at a forced 1000/full towards the
  switch, so `eth0` always looks "up". Port link state is visible with
  `swconfig dev switch0 port 2 show`.
- **SQM / tc**: offloaded flows bypass Linux qdiscs. Use the NSS qdiscs
  (`kmod-qca-nss-drv-qdisc` + `tc qdisc ... nssfq_codel`) for shaping, or
  stop ECM.
- **Firewall**: ECM's firewall include rate-limits new connections from WAN
  to 1000/s, and `nf_conntrack_tcp_no_window_check=1` is set because
  offloaded packets skip conntrack window tracking. `br_netfilter` is
  built in (ECM needs bridged flows in conntrack). If you stop ECM, its
  init script turns `bridge-nf-call-ip(6)tables` off, and nftables
  `bridge` tables stop seeing bridged traffic until one of those sysctls is
  turned back on.
- To turn offload off without reflashing: `/etc/init.d/qca-nss-ecm stop`
  and `/etc/init.d/qca-nss-ecm disable`.

## Where it comes from

- NSS packages and kernel patches: ACwifidude/openwrt, asvio/R7800-nss,
  tingalvin/r7800, ported to 25.12.6 / 6.12 by gyimesibalazs
  (`openwrt-ipq806x-nss-25.12`, branch `ipq806x-nss-25.12.6`). Package
  sources are fetched from git.codelinaro.org (QSDK).
- Differences from that port:
  - Scoped to the OnHub. The other ipq806x boards are not switched to
    swconfig.
  - `990-0010-qca-nss-gmac-of-net-return.patch` is dropped. It made
    `of_get_phy_mode()` return the mode instead of 0, which breaks stmmac,
    DSA and `of_mdio`. nss-gmac already uses the normal API.
  - `990-0001`/`990-1000` (DTS edits for the SoC and the other boards) are
    replaced by `qcom-ipq8064-nss.dtsi` plus the OnHub dtsi changes.
  - Added a default for `NF_CONNTRACK_CHAIN_EVENTS`.
  - Refreshed `990-0013` and `990-0204`, which applied with fuzz on
    6.12.112.
  - `990-0085`: without `br_netfilter` the bridge now always runs its
    netfilter hooks, as stock Linux does. Before, nftables `bridge` tables
    and ebtables were silently skipped on non-NSS builds. NSS builds are
    unchanged, since ECM enables `br_netfilter`.
  - The mac80211 NSS redirect patch freed `sk_buff`s with `kfree()`; it now
    uses `dev_kfree_skb_any()`.
  - Not taken: forced `ap_isolate=0`, the cpufreq script rewrite,
    ath10k/QAM-256 tweaks, branding and CI changes.
